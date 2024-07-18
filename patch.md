Step-by-step commands, including the specific patch command to remove the `keystore` init container, create the secret, and add the necessary volume mounts:


### Goal

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka-broker
  namespace: primary-kafka
spec:
  serviceName: "kafka-broker"
  replicas: 1
  selector:
    matchLabels:
      app: kafka
  template:
    metadata:
      labels:
        app: kafka
    spec:
      containers:
      - name: kafka
        image: confluentinc/cp-kafka:latest
        ports:
        - containerPort: 9092
        volumeMounts:
        - name: kafka-secrets
          mountPath: /vault/secrets/kafka.keystore.jks
          subPath: kafka.keystore.jks
        - name: kafka-secrets
          mountPath: /vault/secrets/kafka.truststore.jks
          subPath: kafka.truststore.jks
      volumes:
      - name: kafka-secrets
        secret:
          secretName: cloudeng-kafka-jks

```



### Step 1: Create the Secret

```sh
kubectl create secret generic cloudeng-kafka-jks \
  --from-file=kafka.keystore.jks=./kafka.keystore.jks \
  --from-file=kafka.truststore.jks=./kafka.truststore.jks \
  -n primary-kafka
```

### Step 2: Remove the `keystore` Init Container

Create a JSON patch file named `remove-keystore-initcontainer.json`:

```json
[
  {
    "op": "remove",
    "path": "/spec/template/spec/initContainers/0"
  }
]
```

Apply the patch to remove the init container:

```sh
kubectl patch statefulset kafka-broker -n primary-kafka --type=json --patch "$(cat remove-keystore-initcontainer.json)"
```

### Step 3: Add Volume Mounts for the JKS Files

Create a JSON patch file named `add-jks-volume-mounts.json`:

```json
[
  {
    "op": "add",
    "path": "/spec/template/spec/volumes/-",
    "value": {
      "name": "kafka-secrets",
      "secret": {
        "secretName": "cloudeng-kafka-jks"
      }
    }
  },
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/volumeMounts/-",
    "value": {
      "name": "kafka-secrets",
      "mountPath": "/vault/secrets/kafka.keystore.jks",
      "subPath": "kafka.keystore.jks"
    }
  },
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/volumeMounts/-",
    "value": {
      "name": "kafka-secrets",
      "mountPath": "/vault/secrets/kafka.truststore.jks",
      "subPath": "kafka.truststore.jks"
    }
  }
]
```

Apply the patch to add the volume mounts:

```sh
kubectl patch statefulset kafka-broker -n primary-kafka --type=json --patch "$(cat add-jks-volume-mounts.json)"
```

### Verification

1. **Check the StatefulSet Configuration**:

   ```sh
   kubectl get statefulset kafka-broker -n primary-kafka -o yaml
   ```

2. **Describe the Pods**:

   ```sh
   kubectl describe pod <pod-name> -n primary-kafka
   ```

Replace `<pod-name>` with the name of one of the pods in the StatefulSet to verify the changes.

These steps ensure that the `keystore` init container is removed, the secret is created, and the necessary volume mounts are added.
