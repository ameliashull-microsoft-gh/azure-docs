---
title: Create a dataflow using Azure IoT Operations
description: Create a dataflow to connect data sources and destinations using Azure IoT Operations.
author: PatAltimore
ms.author: patricka
ms.subservice: azure-data-flows
ms.topic: how-to
ms.date: 10/08/2024
ai-usage: ai-assisted

#CustomerIntent: As an operator, I want to understand how to create a dataflow to connect data sources.
---

# Configure dataflows in Azure IoT Operations

[!INCLUDE [public-preview-note](../includes/public-preview-note.md)]

A dataflow is the path that data takes from the source to the destination with optional transformations. You can configure the dataflow by creating a *Dataflow* custom resource or using the Azure IoT Operations Studio portal. A dataflow is made up of three parts: the **source**, the **transformation**, and the **destination**. 

<!--
```mermaid
flowchart LR
  subgraph Source
  A[DataflowEndpoint]
  end
  subgraph BuiltInTransformation
  direction LR
  Datasets - -> Filter
  Filter - -> Map
  end
  subgraph Destination
  B[DataflowEndpoint]
  end
  Source - -> BuiltInTransformation
  BuiltInTransformation - -> Destination
```
-->

:::image type="content" source="media/howto-create-dataflow/dataflow.svg" alt-text="Diagram of a dataflow showing flow from source to transform then destination.":::

To define the source and destination, you need to configure the dataflow endpoints. The transformation is optional and can include operations like enriching the data, filtering the data, and mapping the data to another field. 

This article shows you how to create a dataflow with an example, including the source, transformation, and destination.

## Prerequisites

- An instance of [Azure IoT Operations Preview](../deploy-iot-ops/howto-deploy-iot-operations.md)
- A [configured dataflow profile](howto-configure-dataflow-profile.md)
- [Dataflow endpoints](howto-configure-dataflow-endpoint.md). For example, create a dataflow endpoint for the [local MQTT broker](./howto-configure-mqtt-endpoint.md#azure-iot-operations-local-mqtt-broker). You can use this endpoint for both the source and destination. Or, you can try other endpoints like Kafka, Event Hubs, or Azure Data Lake Storage. To learn how to configure each type of dataflow endpoint, see [Configure dataflow endpoints](howto-configure-dataflow-endpoint.md).

## Create dataflow

Once you have dataflow endpoints, you can use them to create a dataflow. Recall that a dataflow is made up of three parts: the source, the transformation, and the destination.

# [Portal](#tab/portal)

To create a dataflow in [operations experience](https://iotoperations.azure.com/), select **Dataflow** > **Create dataflow**.

:::image type="content" source="media/howto-create-dataflow/create-dataflow.png" alt-text="Screenshot using operations experience to create a dataflow.":::

# [Bicep](#tab/bicep)

The [Bicep File to create Dataflow](https://github.com/Azure-Samples/explore-iot-operations/blob/main/samples/quickstarts/dataflow.bicep) deploys the necessary resources for dataflows.

1. Download the template file and replace the values for `customLocationName`, `aioInstanceName`, `schemaRegistryName`, `opcuaSchemaName`, and `persistentVCName`.
1. Deploy the resources using the [az stack group](/azure/azure-resource-manager/bicep/deployment-stacks?tabs=azure-powershell) command in your terminal:

    ```azurecli
    az stack group create --name MyDeploymentStack --resource-group $RESOURCE_GROUP --template-file /workspaces/explore-iot-operations/<filename>.bicep --action-on-unmanage 'deleteResources' --deny-settings-mode 'none' --yes
    ```

 The overall structure of a dataflow configuration for Bicep is as follows:

```bicep
resource dataflow 'Microsoft.IoTOperations/instances/dataflowProfiles/dataflows@2024-08-15-preview' = {
  parent: defaultDataflowProfile
  name: 'my-dataflow'
  extendedLocation: {
    name: customLocation.id
    type: 'CustomLocation'
  }
  properties: {
    mode: 'Enabled'
    operations: [
      {
        operationType: 'Source'
        sourceSettings: {
          // See source configuration section
        }
      }
      {
        operationType: 'BuiltInTransformation'
        builtInTransformationSettings: {
          // See transformation configuration section
        }
      }
      {
        operationType: 'Destination'
        destinationSettings: {
          // See destination configuration section
        }
      }
    ]
  }
}
```

# [Kubernetes](#tab/kubernetes)

The overall structure of a dataflow configuration is as follows:

```yaml
apiVersion: connectivity.iotoperations.azure.com/v1beta1
kind: Dataflow
metadata:
  name: my-dataflow
  namespace: azure-iot-operations
spec:
  profileRef: default
  mode: Enabled
  operations:
    - operationType: Source
      sourceSettings:
        # See source configuration section
    - operationType: BuiltInTransformation
      builtInTransformationSettings:
        # See transformation configuration section
    - operationType: Destination
      destinationSettings:
        # See destination configuration section
```

---

<!-- TODO: link to API reference -->

Review the following sections to learn how to configure the operation types of the dataflow.

## Configure a source with a dataflow endpoint to get data

To configure a source for the dataflow, specify the endpoint reference and data source. You can specify a list of data sources for the endpoint.

# [Portal](#tab/portal)

### Use Asset as a source

You can use an [asset](../discover-manage-assets/overview-manage-assets.md) as the source for the dataflow. Using an asset as a source is only available in the operations experience.

1. Under **Source details**, select **Asset**.
1. Select the asset you want to use as the source endpoint.
1. Select **Proceed**.

    A list of datapoints for the selected asset is displayed.

    :::image type="content" source="media/howto-create-dataflow/dataflow-source-asset.png" alt-text="Screenshot using operations experience to select an asset as the source endpoint.":::

1. Select **Apply** to use the asset as the source endpoint.

# [Bicep](#tab/bicep)

Configuring an asset as a source is only available in the operations experience.

# [Kubernetes](#tab/kubernetes)

Configuring an asset as a source is only available in the operations experience.

---

### Use MQTT as a source

# [Portal](#tab/portal)

1. Under **Source details**, select **MQTT**.
1. Enter the **MQTT Topic** that you want to listen to for incoming messages.
1. Choose a **Message schema** from the dropdown list or upload a new schema. If the source data has optional fields or fields with different types, specify a deserialization schema to ensure consistency. For example, the data might have fields that aren't present in all messages. Without the schema, the transformation can't handle these fields as they would have empty values. With the schema, you can specify default values or ignore the fields.

    :::image type="content" source="media/howto-create-dataflow/dataflow-source-mqtt.png" alt-text="Screenshot using operations experience to select MQTT as the source endpoint.":::

1. Select **Apply**.

# [Bicep](#tab/bicep)

The MQTT endpoint is configured in the Bicep template file. For example, the following endpoint is a source for the dataflow.

```bicep
{
  operationType: 'Source'
  sourceSettings: {
    endpointRef: MqttBrokerDataflowEndpoint.name // Reference to the 'mq' endpoint
    dataSources: [
        'azure-iot-operations/data/thermostat', // MQTT topic for thermostat temperature data
        'humidifiers/+/telemetry/humidity/#'  // MQTT topic for humidifier humidity data
        
    ]
  }
}
```

The `dataSources` setting is an array of MQTT topics that define the data source. In this example, `azure-iot-operations/data/thermostat` refers to one of the topics in the dataSources array where thermostat data is published.

Datasources allow you to specify multiple MQTT or Kafka topics without needing to modify the endpoint configuration. This means the same endpoint can be reused across multiple dataflows, even if the topics vary. For more information, see [Reuse dataflow endpoints](./howto-configure-dataflow-endpoint.md#reuse-endpoints).

<!-- TODO: Put the right article link here -->
For more information about creating an MQTT endpoint as a dataflow source, see [MQTT Endpoint](howto-configure-mqtt-endpoint.md).

# [Kubernetes](#tab/kubernetes)

For example, to configure a source using an MQTT endpoint and two MQTT topic filters, use the following configuration:

```yaml
sourceSettings:
  endpointRef: mq
  dataSources:
    - thermostats/+/telemetry/temperature/#
    - humidifiers/+/telemetry/humidity/#
```

Because `dataSources` allows you to specify MQTT or Kafka topics without modifying the endpoint configuration, you can reuse the endpoint for multiple dataflows even if the topics are different. To learn more, see [Reuse dataflow endpoints](./howto-configure-dataflow-endpoint.md#reuse-endpoints).

<!-- TODO: link to API reference -->

#### Specify schema to deserialize data

If the source data has optional fields or fields with different types, specify a deserialization schema to ensure consistency. For example, the data might have fields that aren't present in all messages. Without the schema, the transformation can't handle these fields as they would have empty values. With the schema, you can specify default values or ignore the fields.

```yaml
spec:
  operations:
  - operationType: Source
    sourceSettings:
      serializationFormat: Json
      schemaRef: aio-sr://exampleNamespace/exampleAvroSchema:1.0.0
```

To specify the schema, create the file and store it in the schema registry.

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "name": "Temperature",
  "description": "Schema for representing an asset's key attributes",
  "type": "object",
  "required": [ "deviceId", "asset_name"],
  "properties": {
    "deviceId": {
      "type": "string"
    },
    "temperature": {
      "type": "double"
    },
    "serial_number": {
      "type": "string"
    },
    "production_date": {
      "type": "string",
      "description": "Event duration"
    },
    "asset_name": {
      "type": "string",
      "description": "Name of asset"
    },
    "location": {
      "type": "string",
    },
    "manufacturer": {
      "type": "string",
      "description": "Name of manufacturer"
    }
  }
}
```

> [!NOTE]
> The only supported serialization format is JSON. The schema is optional.

For more information about schema registry, see [Understand message schemas](concept-schema-registry.md).

#### Shared subscriptions

<!-- TODO: may not be final -->

To use shared subscriptions with MQTT sources, you can specify the shared subscription topic in the form of `$shared/<subscription-group>/<topic>`.

```yaml
sourceSettings:
  dataSources:
    - $shared/myGroup/thermostats/+/telemetry/temperature/#
```

> [!NOTE]
> If the instance count in the [dataflow profile](howto-configure-dataflow-profile.md) is greater than 1, then the shared subscription topic must be used.

<!-- TODO: Details -->

---

## Configure transformation to process data

The transformation operation is where you can transform the data from the source before you send it to the destination. Transformations are optional. If you don't need to make changes to the data, don't include the transformation operation in the dataflow configuration. Multiple transformations are chained together in stages regardless of the order in which they're specified in the configuration. The order of the stages is always

1. **Enrich**: Add additional data to the source data given a dataset and condition to match.
1. **Filter**: Filter the data based on a condition.
1. **Map**: Move data from one field to another with an optional conversion.

# [Portal](#tab/portal)

In the operations experience, select **Dataflow** > **Add transform (optional)**.

:::image type="content" source="media/howto-create-dataflow/dataflow-transform.png" alt-text="Screenshot using operations experience to add a transform to a dataflow.":::

# [Bicep](#tab/bicep)

```bicep
{
  operationType: 'BuiltInTransformation'
  builtInTransformationSettings: {
    map: [
      // ...
    ]
    filter: [
      // ...
    ]
  }
}
```

### Specify output schema to transform data

The following configuration demonstrates how to define an output schema in your Bicep file. In this example, the schema defines fields such as `asset_id`, `asset_name`, `location`, `temperature`, `manufacturer`, `production_date`, and `serial_number`. Each field is assigned a data type and marked as non-nullable. The assignment ensures all incoming messages contain these fields with valid data.

```bicep
var assetDeltaSchema = '''
{
    "$schema": "Delta/1.0",
    "type": "object",
    "properties": {
        "type": "struct",
        "fields": [
            { "name": "asset_id", "type": "string", "nullable": false, "metadata": {} },
            { "name": "asset_name", "type": "string", "nullable": false, "metadata": {} },
            { "name": "location", "type": "string", "nullable": false, "metadata": {} },
            { "name": "manufacturer", "type": "string", "nullable": false, "metadata": {} },
            { "name": "production_date", "type": "string", "nullable": false, "metadata": {} },
            { "name": "serial_number", "type": "string", "nullable": false, "metadata": {} },
            { "name": "temperature", "type": "double", "nullable": false, "metadata": {} }
        ]
    }
}
'''
```

The following Bicep configuration registers the schema with the Azure Schema Registry. This configuration creates a schema definition and assigns it a version within the schema registry, allowing it to be referenced later in your data transformations.

```bicep
param opcuaSchemaName string = 'opcua-output-delta'
param opcuaSchemaVer string = '1'

resource opcSchema 'Microsoft.DeviceRegistry/schemaRegistries/schemas@2024-09-01-preview' = {
  parent: schemaRegistry
  name: opcuaSchemaName
  properties: {
    displayName: 'OPC UA Delta Schema'
    description: 'This is a OPC UA delta Schema'
    format: 'Delta/1.0'
    schemaType: 'MessageSchema'
  }
}

resource opcuaSchemaInstance 'Microsoft.DeviceRegistry/schemaRegistries/schemas/schemaVersions@2024-09-01-preview' = {
  parent: opcSchema
  name: opcuaSchemaVer
  properties: {
    description: 'Schema version'
    schemaContent: opcuaSchemaContent
  }
}
```

# [Kubernetes](#tab/kubernetes)

```yaml
builtInTransformationSettings:
  datasets:
    # ...
  filter:
    # ...
  map:
    # ...
```

<!-- TODO: link to API reference -->


---

### Enrich: Add reference data

To enrich the data, you can use the reference dataset in the Azure IoT Operations [distributed state store (DSS)](../create-edge-apps/concept-about-state-store-protocol.md). The dataset is used to add extra data to the source data based on a condition. The condition is specified as a field in the source data that matches a field in the dataset.

Key names in the distributed state store correspond to a dataset in the dataflow configuration.

# [Portal](#tab/portal)

Currently, the enrich operation isn't available in the operations experience.

# [Bicep](#tab/bicep)

This example shows how you could use the `deviceId` field in the source data to match the `asset` field in the dataset:

```bicep
builtInTransformationSettings: {
  datasets: [
    {
      key: 'assetDataset'
      inputs: [
        '$source.deviceId',                // Reference to the device ID from the source
        '$context(assetDataset).asset'     // Reference to the asset from the dataset context
      ]
      expression: '$1 == $2'                // Expression to evaluate the inputs
    }
  ]
}
```

### Passthrough operation

For example, you could apply a passthrough operation that takes all the input fields and maps them to the output field, essentially passing through all fields. 

```bicep
builtInTransformationSettings: {
  map: [
    {
      inputs: array('*')
      output: '*'
    }
  ]
}
```

# [Kubernetes](#tab/kubernetes)

For example, you could use the `deviceId` field in the source data to match the `asset` field in the dataset:

```yaml
builtInTransformationSettings:
  datasets:
  - key: assetDataset
    inputs:
      - $source.deviceId # ------------- $1
      - $context(assetDataset).asset # - $2
    expression: $1 == $2
```

If the dataset has a record with the `asset` field, similar to:

```json
{
  "asset": "thermostat1",
  "location": "room1",
  "manufacturer": "Contoso"
}
```

The data from the source with the `deviceId` field matching `thermostat1` has the `location` and `manufacturer` fields available in `filter` and `map` stages.

<!-- TODO: link to API reference -->

---

You can load sample data into the DSS by using the [DSS set tool sample](https://github.com/Azure-Samples/explore-iot-operations/tree/main/samples/dss_set).

For more information about condition syntax, see [Enrich data by using dataflows](concept-dataflow-enrich.md) and [Convert data using dataflows](concept-dataflow-conversions.md).

### Filter: Filter data based on a condition

To filter the data on a condition, you can use the `filter` stage. The condition is specified as a field in the source data that matches a value.

# [Portal](#tab/portal)

1. Under **Transform (optional)**, select **Filter** > **Add**.
1. Choose the datapoints to include in the dataset.
1. Add a filter condition and description.

    :::image type="content" source="media/howto-create-dataflow/dataflow-filter.png" alt-text="Screenshot using operations experience to add a filter transform.":::

1. Select **Apply**.

# [Bicep](#tab/bicep)

For example, you could use the `temperature` field in the source data to filter the data:

```bicep
builtInTransformationSettings: {
  filter: [
    {
      inputs: [
        'temperature ? $last' // Reference to the last temperature value, if available
      ]
      expression: '$1 > 20'   // Expression to filter based on the temperature value
    }
  ]
}
```

# [Kubernetes](#tab/kubernetes)

For example, you could use the `temperature` field in the source data to filter the data:

```yaml
builtInTransformationSettings:
  filter:
    - inputs:
      - temperature ? $last # - $1
      expression: "$1 > 20"
```

If the `temperature` field is greater than 20, the data is passed to the next stage. If the `temperature` field is less than or equal to 20, the data is filtered.

<!-- TODO: link to API reference -->

---

### Map: Move data from one field to another

To map the data to another field with optional conversion, you can use the `map` operation. The conversion is specified as a formula that uses the fields in the source data.

# [Portal](#tab/portal)

In the operations experience, mapping is currently supported using **Compute** transforms.

1. Under **Transform (optional)**, select **Compute** > **Add**.
1. Enter the required fields and expressions.

    :::image type="content" source="media/howto-create-dataflow/dataflow-compute.png" alt-text="Screenshot using operations experience to add a compute transform.":::

1. Select **Apply**.

# [Bicep](#tab/bicep)

For example, you could use the `temperature` field in the source data to convert the temperature to Celsius and store it in the `temperatureCelsius` field. You could also enrich the source data with the `location` field from the contextualization dataset:

```bicep
builtInTransformationSettings: {
  map: [
    {
      inputs: [
        'temperature'                     // Reference to the temperature input
      ]
      output: 'temperatureCelsius'        // Output variable for the converted temperature
      expression: '($1 - 32) * 5/9'       // Expression to convert Fahrenheit to Celsius
    }
    {
      inputs: [
        '$context(assetDataset).location'  // Reference to the location from the dataset context
      ]
      output: 'location'                   // Output variable for the location
    }
  ]
}
```

# [Kubernetes](#tab/kubernetes)

For example, you could use the `temperature` field in the source data to convert the temperature to Celsius and store it in the `temperatureCelsius` field. You could also enrich the source data with the `location` field from the contextualization dataset:

```yaml
builtInTransformationSettings:
  map:
    - inputs:
      - temperature # - $1
      output: temperatureCelsius
      expression: "($1 - 32) * 5/9"
    - inputs:
      - $context(assetDataset).location  
      output: location
```

<!-- TODO: link to API reference -->

---

To learn more, see [Map data by using dataflows](concept-dataflow-mapping.md) and [Convert data by using dataflows](concept-dataflow-conversions.md).

### Serialize data according to a schema

If you want to serialize the data before sending it to the destination, you need to specify a schema and serialization format. Otherwise, the data is serialized in JSON with the types inferred. Remember that storage endpoints like Microsoft Fabric or Azure Data Lake require a schema to ensure data consistency.

# [Portal](#tab/portal)

Specify the **Output** schema when you add the destination dataflow endpoint.

# [Bicep](#tab/bicep)

When the dataflow resource is created, it includes a `schemaRef` value that points to the generated schema stored in the schema registry. It can be referenced in transformations which creates a new schema in Delta format. 
This [Bicep File to create Dataflow](https://github.com/Azure-Samples/explore-iot-operations/blob/main/samples/quickstarts/dataflow.bicep) provides a streamlined approach to provisioning the dataflow with schema integration.

```bicep
{
  operationType: 'BuiltInTransformation'
  builtInTransformationSettings: {
    // ..
    schemaRef: 'aio-sr://${opcuaSchemaName}:${opcuaSchemaVer}'
    serializationFormat: 'Parquet' 
  }
}
```

For more information about schema registry, see [Understand message schemas](concept-schema-registry.md).

# [Kubernetes](#tab/kubernetes)


```yaml
builtInTransformationSettings:
  serializationFormat: Parquet
  schemaRef: aio-sr://<NAMESPACE>/<SCHEMA>:<VERSION>
```

To specify the schema, you can create a Schema custom resource with the schema definition.

For more information about schema registry, see [Understand message schemas](concept-schema-registry.md).

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "name": "Temperature",
  "description": "Schema for representing an asset's key attributes",
  "type": "object",
  "required": [ "deviceId", "asset_name"],
  "properties": {
    "deviceId": {
      "type": "string"
    },
    "temperature": {
      "type": "double"
    },
    "serial_number": {
      "type": "string"
    },
    "production_date": {
      "type": "string",
      "description": "Event duration"
    },
    "asset_name": {
      "type": "string",
      "description": "Name of asset"
    },
    "location": {
      "type": "string",
    },
    "manufacturer": {
      "type": "string",
      "description": "Name of manufacturer"
    }
  }
}
```

---

Supported serialization formats are JSON, Parquet, and Delta.

## Configure destination with a dataflow endpoint to send data

To configure a destination for the dataflow, specify the endpoint reference and data destination. You can specify a list of data destinations for the endpoint.

> [!IMPORTANT]
> Storage endpoints require a schema reference. If you've created storage destination endpoints for Microsoft Fabric OneLake, ADLS Gen 2, Azure Data Explorer and Local Storage, use bicep to specify the schema reference.

# [Portal](#tab/portal)

1. Select the dataflow endpoint to use as the destination.

    :::image type="content" source="media/howto-create-dataflow/dataflow-destination.png" alt-text="Screenshot using operations experience to select Event Hubs destination endpoint.":::

1. Select **Proceed** to configure the destination.
1. Add the mapping details based on the type of destination.

# [Bicep](#tab/bicep)

The following example configures Fabric OneLake as a destination with a static MQTT topic.

```bicep
resource oneLakeEndpoint 'Microsoft.IoTOperations/instances/dataflowEndpoints@2024-08-15-preview' = {
  parent: aioInstance
  name: 'onelake-ep'
  extendedLocation: {
    name: customLocation.id
    type: 'CustomLocation'
  }
  properties: {
    endpointType: 'FabricOneLake'
    fabricOneLakeSettings: {
      authentication: {
        method: 'SystemAssignedManagedIdentity'
        systemAssignedManagedIdentitySettings: {}
      }
      oneLakePathType: 'Tables'
      host: 'https://msit-onelake.dfs.fabric.microsoft.com'
      names: {
        lakehouseName: '<EXAMPLE-LAKEHOUSE-NAME>'
        workspaceName: '<EXAMPLE-WORKSPACE-NAME>'
      }
      batching: {
        latencySeconds: 5
        maxMessages: 10000
      }
    }
  }
}
```

```bicep
{
  operationType: 'Destination'
  destinationSettings: {
    endpointRef: oneLakeEndpoint.name // oneLake endpoint
    dataDestination: 'sensorData' // static MQTT topic
  }
}
```

# [Kubernetes](#tab/kubernetes)

For example, to configure a destination using the MQTT endpoint created earlier and a static MQTT topic, use the following configuration:

```yaml
destinationSettings:
  endpointRef: mq
  dataDestination: factory
```

## Example

The following example is a dataflow configuration that uses the MQTT endpoint for the source and destination. The source filters the data from the MQTT topics `thermostats/+/telemetry/temperature/#` and `humidifiers/+/telemetry/humidity/#`. The transformation converts the temperature to Fahrenheit and filters the data where the temperature is less than 100000. The destination sends the data to the MQTT topic `factory`.

```yaml
apiVersion: connectivity.iotoperations.azure.com/v1beta1
kind: Dataflow
metadata:
  name: my-dataflow
  namespace: azure-iot-operations
spec:
  profileRef: default
  mode: Enabled
  operations:
    - operationType: Source
      sourceSettings:
        endpointRef: mq
        dataSources:
          - thermostats/+/telemetry/temperature/#
          - humidifiers/+/telemetry/humidity/#
    - operationType: builtInTransformation
      builtInTransformationSettings:
        filter:
          - inputs:
              - 'temperature.Value'
              - '"Tag 10".Value'
            expression: "$1*$2<100000"
        map:
          - inputs:
              - '*'
            output: '*'
          - inputs:
              - temperature.Value
            output: TemperatureF
            expression: cToF($1)
          - inputs:
              - '"Tag 10".Value'
            output: 'Tag 10'
    - operationType: Destination
      destinationSettings:
        endpointRef: mq
        dataDestination: factory
```

<!-- TODO: add links to examples in the reference docs -->

---

## Verify a dataflow is working

Follow [Tutorial: Bi-directional MQTT bridge to Azure Event Grid](tutorial-mqtt-bridge.md) to verify the dataflow is working.

### Export dataflow configuration

To export the dataflow configuration, you can use the operations experience or by exporting the Dataflow custom resource.

# [Portal](#tab/portal)

Select the dataflow you want to export and select **Export** from the toolbar.

:::image type="content" source="media/howto-create-dataflow/dataflow-export.png" alt-text="Screenshot using operations experience to export a dataflow.":::

# [Bicep](#tab/bicep)

Bicep is infrastructure as code and no export is required. Use the [Bicep template file to create a dataflow](https://github.com/Azure-Samples/explore-iot-operations/blob/main/samples/quickstarts/dataflow.bicep) to quickly set up and configure dataflows.

# [Kubernetes](#tab/kubernetes)

```bash
kubectl get dataflow my-dataflow -o yaml > my-dataflow.yaml
```

---