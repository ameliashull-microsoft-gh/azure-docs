---
title: Configure TLS with automatic certificate management to secure MQTT communication
description: Configure TLS with automatic certificate management to secure MQTT communication between the MQTT broker and client.
author: PatAltimore
ms.author: patricka
ms.subservice: azure-mqtt-broker
ms.topic: how-to
ms.custom:
  - ignite-2023
ms.date: 08/22/2024

#CustomerIntent: As an operator, I want to configure MQTT broker to use TLS so that I have secure communication between the MQTT broker and client.
---

# Configure TLS with automatic certificate management to secure MQTT communication in MQTT broker

[!INCLUDE [public-preview-note](../includes/public-preview-note.md)]

You can configure TLS to secure MQTT communication between the MQTT broker and client using a [BrokerListener resource](howto-configure-brokerlistener.md). You can configure TLS with manual or automatic certificate management. 

## Verify cert-manager installation

With automatic certificate management, you use cert-manager to manage the TLS server certificate. By default, cert-manager is installed alongside Azure IoT Operations Preview in the `azure-iot-operations` namespace already. Verify the installation before proceeding.

1. Use `kubectl` to check for the pods matching the cert-manager app labels.

    ```bash
    kubectl get pods --namespace azure-iot-operations -l 'app in (cert-manager,cainjector,webhook)'
    ```
    
    ```Output
    NAME                                           READY   STATUS    RESTARTS       AGE
    aio-cert-manager-64f9548744-5fwdd              1/1     Running   4 (145m ago)   4d20h
    aio-cert-manager-cainjector-6c7c546578-p6vgv   1/1     Running   4 (145m ago)   4d20h
    aio-cert-manager-webhook-7f676965dd-8xs28      1/1     Running   4 (145m ago)   4d20h
    ```

1. If you see the pods shown as ready and running, cert-manager is installed and ready to use. 

> [!TIP]
> To further verify the installation, check the cert-manager documentation [verify installation](https://cert-manager.io/docs/installation/kubernetes/#verifying-the-installation). Remember to use the `azure-iot-operations` namespace.

## Create an Issuer for the TLS server certificate

The cert-manager Issuer resource defines how certificates are automatically issued. Cert-manager [supports several Issuers types natively](https://cert-manager.io/docs/configuration/). It also supports an [external](https://cert-manager.io/docs/configuration/external/) issuer type for extending functionality beyond the natively supported issuers. MQTT broker can be used with any type of cert-manager issuer.

> [!IMPORTANT]
> During initial deployment, Azure IoT Operations is installed with a default Issuer for TLS server certificates. You can use this issuer for development and testing. For more information, see [Default root CA and issuer with Azure IoT Operations](#default-root-ca-and-issuer). The steps below are only required if you want to use a different issuer.

The approach to create the issuer is different depending on your scenario. The following sections list examples to help you get started.

# [Development or test](#tab/test)

The CA issuer is useful for development and testing. It must be configured with a certificate and private key stored in a Kubernetes secret.

### Set up the root certificate as a Kubernetes secret

If you have an existing CA certificate, create a Kubernetes secret with the CA certificate and private key PEM files. Run the following command and you have set up the root certificate as a Kubernetes secret and can skip the next section.

```bash
kubectl create secret tls test-ca --cert tls.crt --key tls.key -n azure-iot-operations
```

If you don't have a CA certificate, cert-manager can generate a root CA certificate for you. Using cert-manager to generate a root CA certificate is known as [bootstrapping](https://cert-manager.io/docs/configuration/selfsigned/#bootstrapping-ca-issuers) a CA issuer with a self-signed certificate.

1. Start by creating `ca.yaml`:

    ```yaml
    apiVersion: cert-manager.io/v1
    kind: Issuer
    metadata:
      name: selfsigned-ca-issuer
      namespace: azure-iot-operations
    spec:
      selfSigned: {}
    ---
    apiVersion: cert-manager.io/v1
    kind: Certificate
    metadata:
      name: selfsigned-ca-cert
      namespace: azure-iot-operations
    spec:
      isCA: true
      commonName: test-ca
      secretName: test-ca
      issuerRef:
        # Must match Issuer name above
        name: selfsigned-ca-issuer
        # Must match Issuer kind above
        kind: Issuer
        group: cert-manager.io
      # Override default private key config to use an EC key
      privateKey:
        rotationPolicy: Always
        algorithm: ECDSA
        size: 256
    ```

1. Create the self-signed CA certificate with the following command:

    ```bash
    kubectl apply -f ca.yaml
    ```

Cert-manager creates a CA certificate using its defaults. The properties of this certificate can be changed by modifying the Certificate specification. See [cert-manager documentation](https://cert-manager.io/docs/reference/api-docs/#cert-manager.io/v1.CertificateSpec) for a list of valid options.

### Distribute the root certificate

The prior example stores the CA certificate in a Kubernetes secret called `test-ca`. The certificate in PEM format can be retrieved from the secret and stored in a file `ca.crt` with the following command:

```bash
kubectl get secret test-ca -n azure-iot-operations -o json | jq -r '.data["tls.crt"]' | base64 -d > ca.crt
```

This certificate must be distributed and trusted by all clients. For example, use `--cafile` flag for a mosquitto client.

### Create issuer based on CA certificate

Cert-manager needs an issuer based on the CA certificate generated or imported in the earlier step. Create the following file as `issuer-ca.yaml`:

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: my-issuer
  namespace: azure-iot-operations
spec:
  ca:
    # Must match secretName of generated or imported CA cert
    secretName: test-ca
```

Create the issuer with the following command:

```bash
kubectl apply -f issuer-ca.yaml
```

The prior command creates an issuer for issuing the TLS server certificates. Note the name and kind of the issuer. In the example,  name `my-issuer` and kind `Issuer`. These values are set in the BrokerListener resource later.

# [Production](#tab/prod)

For production, check cert-manager documentation to see which issuer works best for you. For example, with [Vault Issuer](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-cert-manager):

1. Deploy the vault in an environment of choice, like [Kubernetes](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-raft-deployment-guide). Initialize and unseal the vault accordingly.
1. Create and configure the PKI secrets engine by importing your CA certificate.
1. Configure vault with role and policy for issuing server certificates.

    The following is an example role. Note `ExtKeyUsageServerAuth` makes the server certificate work:

    ```bash
    vault write pki/roles/my-issuer \
      allow_any_name=true \
      client_flag=false \
      ext_key_usage=ExtKeyUsageServerAuth \
      no_store=true \
      max_ttl=240h
    ```

    Example policy for the role:

    ```hcl
    path "pki/sign/my-issuer" {
      capabilities = ["create", "update"]
    }
    ```

1. Set up authentication between cert-manager and vault using a method of choice, like [SAT](https://developer.hashicorp.com/vault/docs/auth/kubernetes).
1. [Configure the cert-manager Vault Issuer](https://cert-manager.io/docs/configuration/vault/).

---

### Enable TLS for a port

Modify the `tls` setting in a BrokerListener resource to specify a TLS port and Issuer for the frontends. The following is an example of a BrokerListener resource that enables TLS on port 8884 with automatic certificate management.

```yaml
apiVersion: mqttbroker.iotoperations.azure.com/v1beta1
kind: BrokerListener
metadata:
  name: my-new-tls-listener
  namespace: azure-iot-operations
spec:
  brokerRef: broker
  serviceType: loadBalancer
  serviceName: my-new-tls-listener # Avoid conflicts with default service name 'aio-broker'
  ports:
  - port: 8884 # Avoid conflicts with default port 18883
    tls:
      mode: Automatic
      certManagerCertificateSpec:
        issuerRef:
          name: my-issuer
          kind: Issuer
```

Once the BrokerListener resource is configured, MQTT broker automatically creates a new service with the specified port and TLS enabled.

### Optional: Configure server certificate parameters

The only required parameters are `issuerRef.name` and `issuerRef.kind`. All properties of the generated TLS server certificates are automatically chosen. However, MQTT broker allows certain properties to be customized by specifying them in the BrokerListener resource, under `tls.automatic.issuerRef`. The following is an example of all supported properties:

```yaml
# cert-manager issuer for TLS server certificate. Required.
issuerRef:
  # Name of issuer. Required.
  name: my-issuer
  # 'Issuer' or 'ClusterIssuer'. Required.
  kind: Issuer
  # Issuer group. Optional; defaults to 'cert-manager.io'.
  # External issuers may use other groups.
  group: cert-manager.io
# Namespace of certificate. Optional; omit to use default namespace.
namespace: az
# Where to store the generated TLS server certificate. Any existing
# data at the provided secret will be overwritten.
# Optional; defaults to 'my-issuer-{port}'.
secret: my-issuer-8884
# Parameters for the server certificate's private key.
# Optional; defaults to rotationPolicy: Always, algorithm: ECDSA, size: 256.
privateKey:
  rotationPolicy: Always
  algorithm: ECDSA
  size: 256
# Total lifetime of the TLS server certificate. Optional; defaults to '720h' (30 days).
duration: 720h
# When to begin renewing the certificate. Optional; defaults to '240h' (10 days).
renewBefore: 240h
# Any additional SANs to add to the server certificate. Omit if not required.
san:
  dns:
  - iotmq.example.com
  # To connect to the broker from a different namespace, add the following DNS name:
  - aio-broker.azure-iot-operations.svc.cluster.local
  ip:
  - 192.168.1.1
```

### Verify deployment

Use kubectl to check that the service associated with the BrokerListener resource is running. From the example above, the service name is `my-new-tls-listener` and the namespace is `azure-iot-operations`. The following command checks the service status:

```console 
$ kubectl get service my-new-tls-listener -n azure-iot-operations
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
my-new-tls-listener    LoadBalancer   10.43.241.171   XXX.XX.X.X    8884:32457/TCP   33s
```

### Connect to the broker with TLS

Once the server certificate is configured, TLS is enabled. To test with mosquitto:

```bash
mosquitto_pub -h $HOST -p 8884 -V mqttv5 -i "test" -t "test" -m "test" --cafile ca.crt
```

The `--cafile` argument enables TLS on the mosquitto client and specifies that the client should trust all server certificates issued by the given file. You must specify a file that contains the issuer of the configured TLS server certificate.

Replace `$HOST` with the appropriate host:

- If connecting from [within the same cluster](howto-test-connection.md#connect-from-a-pod-within-the-cluster-with-default-configuration), replace with the service name given (`my-new-tls-listener` in example) or the service `CLUSTER-IP`.
- If connecting from outside the cluster, the service `EXTERNAL-IP`.

Remember to specify authentication methods if needed.

## Default root CA and issuer

To help you get started, Azure IoT Operations is deployed with a default "quickstart" root CA and issuer for TLS server certificates. You can use this issuer for development and testing. For more information, see [Default root CA and issuer for TLS server certificates](./concept-default-root-ca.md).

For production, you must configure a CA issuer with a certificate from a trusted CA, as described in the previous sections.

## Related content

- About [BrokerListener resource](howto-configure-brokerlistener.md)
- [Configure authorization for a BrokerListener](./howto-configure-authorization.md)
- [Configure authentication for a BrokerListener](./howto-configure-authentication.md)
- [Configure TLS with manual certificate management](./howto-configure-tls-manual.md)
