---
title: Configure MQTT broker authorization
description: Configure MQTT broker authorization using BrokerAuthorization.
author: PatAltimore
ms.author: patricka
ms.subservice: azure-mqtt-broker
ms.topic: how-to
ms.custom:
  - ignite-2023
ms.date: 09/09/2024

#CustomerIntent: As an operator, I want to configure authorization so that I have secure MQTT broker communications.
---

# Configure MQTT broker authorization

[!INCLUDE [public-preview-note](../includes/public-preview-note.md)]

Authorization policies determine what actions the clients can perform on the broker, such as connecting, publishing, or subscribing to topics. Configure MQTT broker to use one or multiple authorization policies with the *BrokerAuthorization* resource. Each *BrokerAuthorization* resource contains a list of rules that specify the principals and resources for the authorization policies.

## Link BrokerAuthorization to BrokerListener

To link a *BrokerListener* to a *BrokerAuthorization* resource, specify the `authenticationRef` field in the `ports` setting of the *BrokerListener* resource. Similar to BrokerAuthentication, the *BrokerAuthorization* resource can be linked to multiple *BrokerListener* ports. The authorization policies apply to all linked listener ports. However, there's one key difference compared with BrokerAuthentication:

> [!IMPORTANT]
> To have the *BrokerAuthorization* configuration apply to a listener port, at least one BrokerAuthentication must also be linked to that listener port.

To learn more about *BrokerListener*, see [BrokerListener resource](howto-configure-brokerlistener.md).

## Authorization rules

To configure authorization, create a *BrokerAuthorization* resource in your Kubernetes cluster. The following sections provide examples of how to configure authorization for clients that use usernames, attributes, X.509 certificates, and Kubernetes Service Account Tokens (SATs). For a list of the available settings, see the [Broker Authorization](/rest/api/iotoperationsmq/broker-authorization) API reference.

The following example shows how to create a *BrokerAuthorization* resource using both usernames and attributes:

```yaml
apiVersion: mqttbroker.iotoperations.azure.com/v1beta1
kind: BrokerAuthorization
metadata:
  name: "my-authz-policies"
  namespace: azure-iot-operations
spec:
  authorizationPolicies:
    cache: Enabled
    rules:
      - principals:
          usernames:
            - "temperature-sensor"
            - "humidity-sensor"
          attributes:
            - city: "seattle"
              organization: "contoso"
        brokerResources:
          - method: Connect
          - method: Publish
            topics:
              - "/telemetry/{principal.username}"
              - "/telemetry/{principal.attributes.organization}"
          - method: Subscribe
            topics:
              - "/commands/{principal.attributes.organization}"
```

This broker authorization allows clients with usernames `temperature-sensor` or `humidity-sensor`, or clients with attributes `organization` with value `contoso` and `city` with value `seattle`, to:

- Connect to the broker.
- Publish messages to telemetry topics scoped with their usernames and organization. For example:
  - `temperature-sensor` can publish to `/telemetry/temperature-sensor` and `/telemetry/contoso`.
  - `humidity-sensor` can publish to `/telemetry/humidity-sensor` and `/telemetry/contoso`.
  - `some-other-username` can publish to `/telemetry/contoso`.
- Subscribe to commands topics scoped with their organization. For example:
  - `temperature-sensor` can subscribe to `/commands/contoso`.
  - `some-other-username` can subscribe to `/commands/contoso`.

To create this *BrokerAuthorization* resource, apply the YAML manifest to your Kubernetes cluster.

### Further limit access based on client ID

Because the `principals` field is a logical OR, you can further restrict access based on client ID by adding the `clientIds` field to the `brokerResources` field. For example, to allow clients with client IDs that start with its building number to connect and publish telemetry to topics scoped with their building, use the following configuration:

```yaml
apiVersion: mqttbroker.iotoperations.azure.com/v1beta1
kind: BrokerAuthorization
metadata:
  name: "my-authz-policies"
  namespace: azure-iot-operations
spec:
  authorizationPolicies:
    cache: Enabled
    rules:
    - principals:
        attributes:
          - building: "building22"
          - building: "building23"
      brokerResources:
      - method: Connect
        clientIds:
          - "{principal.attributes.building}*" # client IDs must start with building22
      - method: Publish
        topics:
          - "sensors/{principal.attributes.building}/{principal.clientId}/telemetry"
```

Here, if the `clientIds` were not set under the `Connect` method, a client with any client ID could connect as long as it had the `building` attribute set to `building22` or `building23`. By adding the `clientIds` field, only clients with client IDs that start with `building22` or `building23` can connect. This ensures not only that the client has the correct attribute but also that the client ID matches the expected pattern.

## Authorize clients that use X.509 authentication

Clients that use [X.509 certificates for authentication](./howto-configure-authentication.md) can be authorized to access resources based on X.509 properties present on their certificate or their issuing certificates up the chain.

### Using attributes

To create rules based on properties from a client's certificate, its root CA, or intermediate CA, define the X.509 attributes in the *BrokerAuthorization* resource. For more information, see [Certificate attributes](howto-configure-authentication.md#certificate-attributes-for-authorization).

### With client certificate subject common name as username

To create authorization policies based on the *client* certificate subject common name (CN) only, create rules based on the CN.

For example, if a client has a certificate with subject `CN = smart-lock`, its username is `smart-lock`. From there, create authorization policies as normal.

## Authorize clients that use Kubernetes Service Account Tokens

Authorization attributes for SATs are set as part of the Service Account annotations. For example, to add an authorization attribute named `group` with value `authz-sat`, run the command:

```bash
kubectl annotate serviceaccount mqtt-client aio-broker-auth/group=authz-sat
```

Attribute annotations must begin with `aio-broker-auth/` to distinguish them from other annotations.

As the application has an authorization attribute called `authz-sat`, there's no need to provide a `clientId` or `username`. The corresponding *BrokerAuthorization* resource uses this attribute as a principal, for example:

```yaml
apiVersion: mqttbroker.iotoperations.azure.com/v1beta1
kind: BrokerAuthorization
metadata:
  name: "my-authz-policies"
  namespace: azure-iot-operations
spec:
  authorizationPolicies:
    enableCache: false
    rules:
      - principals:
          attributes:
            - group: "authz-sat"
        brokerResources:
          - method: Connect
          - method: Publish
            topics:
              - "odd-numbered-orders"
          - method: Subscribe
            topics:
              - "orders"
```

To learn more with an example, see [Set up Authorization Policy with Dapr Client](../create-edge-apps/howto-develop-dapr-apps.md).

## Distributed state store

MQTT broker provides a distributed state store (DSS) that clients can use to store state. The DSS can also be configured to be highly available.

To set up authorization for clients that use the DSS, provide the following permissions:

- Permission to publish to the system key value store `$services/statestore/_any_/command/invoke/request` topic
- Permission to subscribe to the response-topic (set during initial publish as a parameter) `<response_topic>/#`

For more information about DSS authorization, see [state store keys](https://github.com/Azure/iotedge-broker/blob/main/docs/authorization/readme.md#state-store-keys).

## Update authorization

Broker authorization resources can be updated at runtime without restart. All clients connected at the time of the update of policy are disconnected. Changing the policy type is also supported.

```bash
kubectl edit brokerauthorization my-authz-policies
```

## Disable authorization

To disable authorization, omit `authorizationRef` in the `ports` setting of a *BrokerListener* resource.

## Unauthorized publish in MQTT 3.1.1

With MQTT 3.1.1, when a publish is denied, the client receives the PUBACK with no error because the protocol version doesn't support returning error code. MQTTv5 return PUBACK with reason code 135 (Not authorized) when publish is denied.

## Related content

- About [BrokerListener resource](howto-configure-brokerlistener.md)
- [Configure authentication for a BrokerListener](./howto-configure-authentication.md)
- [Configure TLS with manual certificate management](./howto-configure-tls-manual.md)
- [Configure TLS with automatic certificate management](./howto-configure-tls-auto.md)
