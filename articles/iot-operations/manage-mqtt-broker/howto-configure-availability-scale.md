---
title: Configure core MQTT broker settings
description: Configure core MQTT broker settings for high availability, scale, memory usage, and disk-backed message buffer behavior.
author: PatAltimore
ms.author: patricka
ms.topic: how-to
ms.subservice: azure-mqtt-broker
ms.custom:
  - ignite-2023
ms.date: 09/09/2024

#CustomerIntent: As an operator, I want to understand the settings for the MQTT broker so that I can configure it for high availability and scale.
---

# Configure core MQTT broker settings

[!INCLUDE [public-preview-note](../includes/public-preview-note.md)]

The *Broker* resource is the main resource that defines the overall settings for MQTT broker. It also determines the number and type of pods that run the *Broker* configuration, such as the frontends and the backends. You can also use the *Broker* resource to configure its memory profile. Self-healing mechanisms are built in to the broker and it can often automatically recover from component failures. For example, a node fails in a Kubernetes cluster configured for high availability. 

You can horizontally scale the MQTT broker by adding more frontend replicas and backend chains. The frontend replicas are responsible for accepting MQTT connections from clients and forwarding them to the backend chains. The backend chains are responsible for storing and delivering messages to the clients. The frontend pods distribute message traffic across the backend pods, and the backend redundancy factor determines the number of data copies to provide resiliency against node failures in the cluster.

For a list of the available settings, see the [Broker](/rest/api/iotoperationsmq/broker) API reference.

## Configure scaling settings

> [!IMPORTANT]
> At this time, the *Broker* resource can only be configured at initial deployment time using the Azure CLI, Portal or GitHub Action. A new deployment is required if *Broker* configuration changes are needed. 

To configure the scaling settings MQTT broker, you need to specify the `cardinality` fields in the specification of the *Broker* custom resource. For more information on setting the mode and cardinality settings using Azure CLI, see [az iot ops init](/cli/azure/iot/ops#az-iot-ops-init).

The `cardinality` field is a nested field that has these subfields:

- `frontend`: This subfield defines the settings for the frontend pods, such as:
  - `replicas`: The number of frontend pods to deploy. This subfield is required if the `mode` field is set to `distributed`.
  - `workers`: The number of workers to deploy per frontend, currently it must be set to `1`. This subfield is required if the `mode` field is set to `distributed`.
- `backendChain`: This subfield defines the settings for the backend chains, such as:
  - `redundancyFactor`: The number of data copies in each backend chain. This subfield is required if the `mode` field is set to `distributed`.
  - `partitions`: The number of partitions to deploy. This subfield is required if the `mode` field is set to `distributed`.
  - `workers`: The number of workers to deploy per backend, currently it must be set to `1`. This subfield is required if the `mode` field is set to `distributed`.

If `cardinality` field is omitted, cardinality is determined by MQTT broker operator automatically deploys the appropriate number of pods based on the cluster hardware.

To configure the scaling settings MQTT broker, you need to specify the `mode` and `cardinality` fields in the specification of the *Broker* custom resource. For more information on setting the mode and cardinality settings using Azure CLI, see [az iot ops init](/cli/azure/iot/ops#az-iot-ops-init).

## Configure memory profile

> [!IMPORTANT]
> At this time, the *Broker* resource can only be configured at initial deployment time using the Azure CLI, Portal or GitHub Action. A new deployment is required if *Broker* configuration changes are needed.

To configure the memory profile settings MQTT broker, specify the `memoryProfile` fields in the spec of the *Broker* custom resource. For more information on setting the memory profile setting using Azure CLI, see [az iot ops init](/cli/azure/iot/ops#az-iot-ops-init).

`memoryProfile`: This subfield defines the settings for the memory profile. There are a few profiles for the memory usage you can choose:

### Tiny

When using this profile:

- Maximum memory usage of each frontend replica is approximately 99 MiB but the actual maximum memory usage might be higher.
- Maximum memory usage of each backend replica is approximately 102 MiB but the actual maximum memory usage might be higher.

Recommendations when using this profile:

- Only one frontend should be used.
- Clients shouldn't send large packets. You should only send packets smaller than 4 MiB.

### Low

When using this profile:

- Maximum memory usage of each frontend replica is approximately 387 MiB but the actual maximum memory usage might be higher.
- Maximum memory usage of each backend replica is approximately 390 MiB multiplied by the number of backend workers, but the actual maximum memory usage might be higher.

Recommendations when using this profile:

- Only one or two frontends should be used.
- Clients shouldn't send large packets. You should only send packets smaller than 10 MiB.

### Medium

Medium is the default profile.

- Maximum memory usage of each frontend replica is approximately 1.9 GiB but the actual maximum memory usage might be higher.
- Maximum memory usage of each backend replica is approximately 1.5 GiB multiplied by the number of backend workers, but the actual maximum memory usage might be higher.

### High

- Maximum memory usage of each frontend replica is approximately 4.9 GiB but the actual maximum memory usage might be higher.
- Maximum memory usage of each backend replica is approximately 5.8 GiB multiplied by the number of backend workers, but the actual maximum memory usage might be higher.

## Default broker

By default, Azure IoT Operations Preview deploys a default Broker resource named `broker`. It's deployed in the `azure-iot-operations` namespace with cardinality and memory profile settings as configured during the initial deployment with Azure portal or Azure CLI. To see the settings, run the following command:

```bash
kubectl get broker broker -n azure-iot-operations -o yaml
```

### Modify default broker by redeploying

Only [cardinality](#configure-scaling-settings) and [memory profile](#configure-memory-profile) are configurable with Azure portal or Azure CLI during initial deployment. Other settings and can only be configured by modifying the YAML file and redeploying the broker.

To delete the default broker, run the following command:

```bash
kubectl delete broker broker -n azure-iot-operations
```

Then, create a YAML file with desired settings. For example, the following YAML file configures the broker with name `broker` in namespace `azure-iot-operations` with `medium` memory profile and `distributed` mode with two frontend replicas and two backend chains with two partitions and two workers each. Also, the [encryption of internal traffic option](#configure-encryption-of-internal-traffic) is disabled.

```yaml
apiVersion: mqttbroker.iotoperations.azure.com/v1beta1
kind: Broker
metadata:
  name: broker
  namespace: azure-iot-operations
spec:
  memoryProfile: medium
  cardinality:
    backendChain:
      partitions: 2
      redundancyFactor: 2
      workers: 2
    frontend:
      replicas: 2
      workers: 2
  encryptInternalTraffic: false
```

Then, run the following command to deploy the broker:

```bash
kubectl apply -f <path-to-yaml-file>
```

## Configure MQTT broker advanced settings

The broker advanced settings include client configurations, encryption of internal traffic, and certificate rotations. For more information on the advanced settings, see the [Broker]() API reference.

Here's an example of a *Broker* with advanced settings:

```yml
apiVersion: mqttbroker.iotoperations.azure.com/v1beta1
kind: Broker
metadata:
  name: broker
  namespace: azure-iot-operations
spec:
  advanced:
    clients:
        maxSessionExpirySeconds: 282277
        maxMessageExpirySeconds: 1622
        subscriberQueueLimit:
          length: 1000
          strategy: DropOldest
        maxReceiveMaximum: 15000
        maxKeepAliveSeconds: 300
    encryptInternalTraffic: Enabled
    internalCerts:
      duration: 240h
      renewBefore: 45m
      privateKey:
        algorithm: Rsa2048
        rotationPolicy: Always
```

## Configure MQTT broker diagnostic settings

Diagnostic settings allow you to enable metrics and tracing for MQTT broker.

- Metrics provide information about the resource utilization and throughput of MQTT broker.
- Tracing provides detailed information about the requests and responses handled by MQTT broker.

To override default diagnostic settings for MQTT broker, update the `spec.diagnostics` section in  the *Broker* resource. Adjust the log level to control the amount and detail of information that is logged. The log level can be set for different components of MQTT broker. The default log level is `info`.

You can configure diagnostics using the *Broker* custom resource definition (CRD). For more information on the diagnostics settings, see the [Broker](/rest/api/iotoperationsmq/broker) API reference.

Here's an example of a *Broker* custom resource with metrics and tracing enabled and self-check disabled:

```yaml
apiVersion: mqttbroker.iotoperations.azure.com/v1beta1
kind: Broker
metadata:
  name: broker
  namespace: azure-iot-operations
spec:
  diagnostics:
    logs:
      level: "debug"
      opentelemetryExportConfig:
        otlpGrpcEndpoint: "endpoint"
    metrics:
      opentelemetryExportConfig:
        otlpGrpcEndpoint: "endpoint"
        intervalSeconds: 60
    selfCheck:
      mode: Enabled
      intervalSeconds: 120
      timeoutSeconds: 60
    traces:
      cacheSizeMegabytes: 32
      mode: Enabled
      opentelemetryExportConfig:
        otlpGrpcEndpoint: "endpoint"
      selfTracing:
        mode: Enabled
        intervalSeconds: 120
      spanChannelCapacity: 1000
```

## Configure encryption of internal traffic

> [!IMPORTANT]
> At this time, this feature can't be configured using the Azure CLI or Azure portal during initial deployment. To modify this setting, you need to modify the YAML file and [redeploy the broker](#modify-default-broker-by-redeploying).

The **encryptInternalTraffic** feature is used to encrypt the internal traffic between the frontend and backend pods. To use this feature, cert-manager must be installed in the cluster, which is installed by default when using the Azure IoT Operations.

The benefits include:

- **Secure internal traffic**: All internal traffic between the frontend and backend pods is encrypted.

- **Secure data at rest**: All data at rest is encrypted.

- **Secure data in transit**: All data in transit is encrypted.

- **Secure data in memory**: All data in memory is encrypted.

- **Secure data in the message buffer**: All data in the message buffer is encrypted.

- **Secure data in the message buffer on disk**: All data in the message buffer on disk is encrypted.

By default, the **encryptInternalTraffic** feature is enabled. To disable the feature, set the `encryptInternalTraffic` field to `false` in the spec of the *Broker* custom resource when redeploying the broker.

## Configure disk-backed message buffer behavior

> [!IMPORTANT]
> At this time, this feature can't be configured using the Azure CLI or Azure portal during initial deployment. To modify this setting, you need to modify the YAML file and [redeploy the broker](#modify-default-broker-by-redeploying).

The **diskBackedMessageBufferSettings** feature is used for efficient management of message queues within the MQTT broker distributed MQTT broker. The benefits include:

- **Efficient queue management**: In an MQTT broker, each subscriber is associated with a message queue. The speed a subscriber processes messages directly impacts the size of the queue. If a subscriber processes messages slowly or if they disconnect but request an MQTT persistent session, the queue can grow larger than the available memory.

- **Data preservation for persistent sessions**: The **diskBackedMessageBufferSettings** feature ensures that when a queue exceeds the available memory, it's seamlessly buffered to disk. This feature prevents data loss and supports MQTT persistent sessions, allowing subscribers to resume their sessions with their message queues intact upon reconnection. The disk is used as ephemeral storage and serves as a spillover from memory. Data written to disk isn't durable and is lost when the pod exits, but as long as at least one pod in each backend chain remains functional, the broker as a whole doesn't lose any data.

- **Handling connectivity challenges**: Cloud connectors are treated as subscribers with persistent sessions that can face connectivity challenges when unable to communicate with external systems like Event Grid MQTT broker due to network disconnect. In such scenarios, messages (PUBLISHes) accumulate. The MQTT broker intelligently buffers these messages to memory or disk until connectivity is restored, ensuring message integrity.

Understanding and configuring the **diskBackedMessageBufferSettings** feature maintains a robust and reliable message queuing system. Proper configuration is important in scenarios where message processing speed and connectivity are critical factors.

### Configuration options

Tailor the broker message buffer options by adjusting the following settings:

- **Configure the volume**: Specify a persistent volume claim template to mount a dedicated storage volume for your message buffer.

    - **Select a storage class**: Define the desired *StorageClass* using the `storageClassName` property.

    - **Define access modes**: Determine the access modes you need for your volume. For more information, see [persistent volume access modes](https://kubernetes.io/docs/concepts/storage/persistent-volumes#access-modes-1).

Use the following sections to understand the different volume modes.

*Ephemeral* volume is the preferred option. *Persistent* volume is preferred next then *emptyDir* volume. Both persistent volume and ephemeral volume are generally provided by the same storage classes. If you have both options, choose ephemeral. However, ephemeral requires Kubernetes 1.23 or higher.

### Disabled

If you don't want to use the disk-backed message buffer, don't include the `diskBackedMessageBufferSettings` property in your *Broker* CRD.

### Ephemeral volume

[Ephemeral volume](https://kubernetes.io/docs/concepts/storage/ephemeral-volumes#generic-ephemeral-volumes) is the preferred option for your message buffer.

For *ephemeral* volume, follow the advice in the [Considerations for storage providers](#considerations-for-storage-providers) section.

The value of the *ephemeralVolumeClaimSpec* property is used as the ephemeral.*volumeClaimTemplate.spec* property of the volume in the StatefulSet specs of the backend chains.

For example, to use an ephemeral volume with a capacity of 1 gigabyte, specify the following parameters in your Broker CRD:

```yaml
diskBackedMessageBuffer:
  maxSize: "1G"
  ephemeralVolumeClaimSpec:
    storageClassName: "foo"
    accessModes:
    - "ReadWriteOnce"
```

### Persistent volume

[Persistent volume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) is the next preferred option for your message buffer after *ephemeral* volume.

For *persistent* volume, follow the advice in [Considerations for storage providers](#considerations-for-storage-providers) section.

The value of the *persistentVolumeClaimSpec* property is used as the *volumeClaimTemplates.spec* property of the *StatefulSet* specs of the backend chains.

For example, to use a *persistent* volume with a capacity of 1 gigabyte, specify the following parameters in your Broker CRD:

```yaml 
diskBackedMessageBuffer:
  maxSize: "1G"
  persistentVolumeClaimSpec:
    storageClassName: "foo"
    accessModes:
    - "ReadWriteOnce"
```

### emptyDir volume

Use an [emptyDir volume](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir). An *emptyDir volume* is the next preferred option after persistent volume. 

Only use *emptyDir* volume when using a cluster with filesystem quotas. For more information, see details in the [Filesystem project quota tab](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#resource-emphemeralstorage-consumption). If the feature isn't enabled, the cluster does *periodic scanning* that doesn't enforce any limit and allows the host node to fill disk space and mark the whole host node as unhealthy. 

For example, to use an emptyDir volume with a capacity of 1 gigabyte, specify the following parameters in your Broker CRD:

```yaml
      diskBackedMessageBuffer:
        maxSize: "1G"
```

### Considerations for storage providers

Consider the behavior of your chosen storage provider. For example, when using providers like `rancher.io/local-path`. If the provider doesn't support limits, filling up the volume consumes the node's disk space. This could lead to Kubernetes marking the node and all associated pods as unhealthy. It's crucial to understand how your storage provider behaves in such scenarios.

### Persistence

It's important to understand that the **diskBackedMessageBufferSettings** feature isn't synonymous with *persistence*. In this context, *persistence* refers to data that survives across pod restarts. However, this feature provides temporary storage space for data to be saved to disk, preventing memory overflows and data loss during pod restarts.
