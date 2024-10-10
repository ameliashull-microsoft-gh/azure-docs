---
title: Configure dataflow endpoints in Azure IoT Operations
description: Configure dataflow endpoints to create connection points for data sources.
author: PatAltimore
ms.author: patricka
ms.subservice: azure-data-flows
ms.topic: how-to
ms.date: 09/17/2024

#CustomerIntent: As an operator, I want to understand how to configure source and destination endpoints so that I can create a dataflow.
---

# Configure dataflow endpoints

[!INCLUDE [public-preview-note](../includes/public-preview-note.md)]

To get started with dataflows, first create dataflow endpoints. A dataflow endpoint is the connection point for the dataflow. You can use an endpoint as a source or destination for the dataflow. Some endpoint types can be used as both sources and destinations, while others are for destinations only. A dataflow needs at least one source endpoint and one destination endpoint.

## Get started

To get started, use the following table to choose the endpoint type to configure:

| Endpoint type | Description | Can be used as a source | Can be used as a destination |
|---------------|-------------|-------------------------|------------------------------|
| [MQTT](howto-configure-mqtt-endpoint.md) | For bi-directional messaging with MQTT brokers, including the one built-in to Azure IoT Operations and Event Grid. | Yes | Yes |
| [Kafka](howto-configure-kafka-endpoint.md) | For bi-directional messaging with Kafka brokers, including Azure Event Hubs. | Yes | Yes |
| [Data Lake](howto-configure-adlsv2-endpoint.md) | For uploading data to Azure Data Lake Gen2 storage accounts. | No | Yes |
| [Microsoft Fabric OneLake](howto-configure-fabric-endpoint.md) | For uploading data to Microsoft Fabric OneLake lakehouses. | No | Yes |
| [Local storage](howto-configure-local-storage-endpoint.md) | For sending data to a locally available persistent volume, through which you can upload data via Azure Container Storage enabled by Azure Arc edge volumes. | No | Yes |

## Reuse endpoints

Think of each dataflow endpoint as a bundle of configuration settings that contains where the data should come from or go to (the `host` value), how to authenticate with the endpoint, and other settings like TLS configuration or batching preference. So you just need to create it once and then you can reuse it in multiple dataflows where these settings would be the same.

To make it easier to reuse endpoints, the MQTT or Kafka topic filter isn't part of the endpoint configuration. Instead, you specify the topic filter in the dataflow configuration. This means you can use the same endpoint for multiple dataflows that use different topic filters. 

For example, you can use the default MQTT broker dataflow endpoint. You can use it for both the source and destination with different topic filters:

# [Portal](#tab/portal)

:::image type="content" source="media/howto-configure-dataflow-endpoint/create-dataflow-mq-mq.png" alt-text="Screenshot using operations experience to create a dataflow from MQTT to MQTT.":::

# [Kubernetes](#tab/kubernetes)

```yaml
apiVersion: connectivity.iotoperations.azure.com/v1beta1
kind: Dataflow
metadata:
  name: mq-to-mq
  namespace: azure-iot-operations
spec:
  profileRef: default
  operations:
    - operationType: Source
      sourceSettings:
        endpointRef: mq
        dataSources:
        - example/topic/1
    - operationType: Destination
      destinationSettings:
        endpointRef: mq
        dataDestination: example/topic/2
```

---

Similarly, you can create multiple dataflows that use the same MQTT endpoint for other endpoints and topics. For example, you can use the same MQTT endpoint for a dataflow that sends data to a Kafka endpoint.

# [Portal](#tab/portal)

:::image type="content" source="media/howto-configure-dataflow-endpoint/create-dataflow-mq-kafka.png" alt-text="Screenshot using operations experience to create a dataflow from MQTT to Kafka.":::

# [Kubernetes](#tab/kubernetes)

```yaml
apiVersion: connectivity.iotoperations.azure.com/v1beta1
kind: Dataflow
metadata:
  name: mq-to-kafka
  namespace: azure-iot-operations
spec:
  profileRef: default
  operations:
    - operationType: Source
      sourceSettings:
        endpointRef: mq
        dataSources:
        - example/topic/3
    - operationType: Destination
      destinationSettings:
        endpointRef: example-kafka-endpoint
        dataDestination: example/topic/4
```

---

Similar to the MQTT example, you can create multiple dataflows that use the same Kafka endpoint for different topics, or the same Data Lake endpoint for different tables.
