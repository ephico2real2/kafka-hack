# kafka-hack

The script takes an additional flag to decide whether to perform the verification:

If you are not verifying the keystore and truststore every time, then you do not need the `-s` (store password) argument. Here's the updated script with the `-s` argument made optional and used only when the `-v` flag is provided:

### Updated Bash Script

```bash
#!/bin/bash

# Function to display usage information
usage() {
    echo "Usage: $0 -p <pod> -c <container> -n <namespace> -k <keystore_path> -t <truststore_path> [-s <storepass>] [-v]"
    echo "  -p <pod>            Pod name"
    echo "  -c <container>      Container name"
    echo "  -n <namespace>      Namespace"
    echo "  -k <keystore_path>  Path to the keystore file inside the container"
    echo "  -t <truststore_path> Path to the truststore file inside the container"
    echo "  -s <storepass>      Store password (required if -v is used)"
    echo "  -v                  Optional flag to verify the keystore and truststore"
    exit 1
}

VERIFY=false
STOREPASS=""

# Parse command-line arguments
while getopts "p:c:n:k:t:s:v" opt; do
    case ${opt} in
        p )
            POD_NAME=$OPTARG
            ;;
        c )
            CONTAINER_NAME=$OPTARG
            ;;
        n )
            NAMESPACE=$OPTARG
            ;;
        k )
            KEYSTORE_PATH=$OPTARG
            ;;
        t )
            TRUSTSTORE_PATH=$OPTARG
            ;;
        s )
            STOREPASS=$OPTARG
            ;;
        v )
            VERIFY=true
            ;;
        \? )
            usage
            ;;
    esac
done

# Check if all mandatory arguments are provided
if [ -z "${POD_NAME}" ] || [ -z "${CONTAINER_NAME}" ] || [ -z "${NAMESPACE}" ] || [ -z "${KEYSTORE_PATH}" ] || [ -z "${TRUSTSTORE_PATH}" ]; then
    usage
fi

# Verify flag requires the store password
if [ "$VERIFY" = true ] && [ -z "${STOREPASS}" ]; then
    usage
fi

# Copy the keystore file
kubectl cp ${NAMESPACE}/${POD_NAME}:${KEYSTORE_PATH} ./kafka.keystore.jks -c ${CONTAINER_NAME}
if [ $? -ne 0 ]; then
    echo "Failed to copy keystore file."
    exit 1
fi

# Copy the truststore file
kubectl cp ${NAMESPACE}/${POD_NAME}:${TRUSTSTORE_PATH} ./kafka.truststore.jks -c ${CONTAINER_NAME}
if [ $? -ne 0 ]; then
    echo "Failed to copy truststore file."
    exit 1
fi

# Optional verification
if [ "$VERIFY" = true ]; then
    echo "Verifying keystore file (kafka.keystore.jks):"
    keytool -list -v -keystore ./kafka.keystore.jks -storepass ${STOREPASS} | grep -i "valid from"
    if [ $? -ne 0 ]; then
        echo "Failed to verify keystore file."
        exit 1
    fi

    echo "Verifying truststore file (kafka.truststore.jks):"
    keytool -list -v -keystore ./kafka.truststore.jks -storepass ${STOREPASS} | grep -i "valid from"
    if [ $? -ne 0 ]; then
        echo "Failed to verify truststore file."
        exit 1
    fi

    echo "Keystore and Truststore verification successful."
else
    echo "Keystore and Truststore copied successfully."
fi

```

### Make the Script Executable

Give the script executable permissions:

```sh
chmod +x copy_and_verify_jks.sh
```

### Usage Example

Run the script with the required parameters:

- Without verification:

```sh
./copy_and_verify_jks.sh -p kafka-broker-0 -c kafka -n primary-kafka -k /vault/secrets/kafka.keystore.jks -t /vault/secrets/kafka.truststore.jks
```

- With verification:

```sh
./copy_and_verify_jks.sh -p kafka-broker-0 -c kafka -n primary-kafka -k /vault/secrets/kafka.keystore.jks -t /vault/secrets/kafka.truststore.jks -s my-password -v
```

### Explanation

- `-p <pod>`: Pod name (e.g., `kafka-broker-0`)
- `-c <container>`: Container name (e.g., `kafka`)
- `-n <namespace>`: Namespace (e.g., `primary-kafka`)
- `-k <keystore_path>`: Path to the keystore file inside the container (e.g., `/vault/secrets/kafka.keystore.jks`)
- `-t <truststore_path>`: Path to the truststore file inside the container (e.g., `/vault/secrets/kafka.truststore.jks`)
- `-s <storepass>`: Password for the keystore and truststore (required only if `-v` is used)
- `-v`: Optional flag to verify the keystore and truststore

This script will copy the specified JKS keystore and truststore files from the specified pod and container. If the `-v` flag is provided, it will also verify them using the provided password. If any step fails, the script will print an error message and exit.
