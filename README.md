# Gradle Build Cache Node StatefulSet Deployment in AKS

This project provides a Kubernetes StatefulSet deployment for Gradle Build Cache Nodes in Azure Kubernetes Service (AKS). The setup includes two replicas of the build cache node, a service to expose the cache, an ingress controller for external access, and AzureFile for persistent storage. 

## Prerequisites

- Azure Kubernetes Service (AKS) cluster
- Kubernetes CLI (`kubectl`)
- Azure CLI (`az`)
- Docker CLI

## Project Structure

- `gradle-cache-stateful-set.yaml`: Defines the StatefulSet for deploying Gradle Build Cache Nodes.
- `service.yaml`: Defines the Kubernetes Service on port 5071.
- `ingress.yaml`: Defines the Ingress configuration for external access.
- `config.yaml`: Custom configuration file for user authentication. Masked by .gitignore. You should have own copy of that.

## Setup Instructions

**Note**: The manifests are expected to be applied in the `build-cache` namespace. You can change this as needed.

### 1. Create Azure File Share

1. **Create a storage account**:
    ```bash
    az storage account create --resource-group <resource-group> --name <storage-account-name> --location <location> --sku Standard_LRS
    ```

2. **Create a file share**:
    ```bash
    az storage share create --account-name <storage-account-name> --name <file-share-name>
    ```

3. **Get storage account key**:
    ```bash
    az storage account keys list --resource-group <resource-group> --account-name <storage-account-name>
    ```

4. **Create Kubernetes secret**:
    ```bash
    kubectl create secret generic azure-build-cache-secret --from-literal=azurestorageaccountname=<storage-account-name> --from-literal=azurestorageaccountkey=<storage-account-key> -n build-cache
    ```

### 2. Deploy StatefulSet and Service

1. **Apply StatefulSet**:
    ```bash
    kubectl apply -f gradle-cache-stateful-set.yaml
    ```

2. **Apply Service**:
    ```bash
    kubectl apply -f service.yaml
    ```

### 3. Configure Ingress

1. **Apply Ingress**:
    ```bash
    kubectl apply -f ingress.yaml
    ```

### 4. User Authentication Setup

1. **Generate salted hash for password**:
    ```bash
    docker run --interactive --tty gradle/build-cache-node:19.0 hash
    ```

2. **Create `config.yaml` with user credentials**:
   This is complete file example
    ```yaml
    version: 5
    # Restrict UI access
    uiAccess:
        type: "secure"
        username: "<UI access user>"
        password: "<salted-hash passwrod>"
    # List users to read/write cache
    cache:
        accessControl:
        anonymousLevel: "read"
        users:
            <user name/role>: # e.g developer, reader etc.
                password: "<salted-hash passwrod>"
                level: "readwrite"
                note: "Developer to read/write cache"
    ```

3. **Apply config to StatefulSet**:
   
   The details of the applying custom configuration for build cache node is [here](https://docs.gradle.com/build-cache-node/#providing_a_config_file_from_a_secret_or_configmap) 
    - Make a secret for config.yaml file:
      ```
      kubectl create secret generic gradle-build-cache-config-secret -n build-cache --from-file=config.yaml
      ```
    - You will use it as a volume to mount to InitialContainer
      ```yaml
      ...
        initContainers:
        - name: config-mounter
          image: "busybox:1.33.0"
          command: [
                "sh",
                "-ce",
                "cp /tmp/config.yaml /data/conf/config.yaml" ]
          volumeMounts:
            - name: tmp-build-cache-config-file
                mountPath: /tmp
            - name: build-cache-config-dir
                mountPath: /data/conf
      ...
        - name: tmp-build-cache-config-file
            secret:
            secretName: gradle-build-cache-config-secret
      ```

## Accessing the Build Cache Node

1. **Find the external IP of your ingress**:
    ```bash
    kubectl get ingress -n build-cache
    ```

2. **Access the Gradle Build Cache Node UI**:
    - Navigate to `http://<external-ip>:5071` in your browser.

## Benefits of Using AzureFile

- **Durability**: AzureFile ensures that your data is highly available and durable.
- **Simplicity**: Easy to use and manage within the Azure ecosystem.
- **Flexibility**: Can be used by multiple pods concurrently.

## Custom Configuration

Custom configurations are necessary to handle user authentication for pushing cache from a Gradle build. The default Gradle setup poses challenges for saving changes through the UI. To configure a user later, create a special `config.yaml` file and provide a hash-salted password.

## Troubleshooting

- Ensure all Kubernetes resources are correctly applied.
- Verify the external IP and DNS settings for ingress.
- Check the StatefulSet and Service status using `kubectl get statefulsets` and `kubectl get services` in `build-cache` namespace.

## Contributions

Feel free to contribute by submitting issues or pull requests.
