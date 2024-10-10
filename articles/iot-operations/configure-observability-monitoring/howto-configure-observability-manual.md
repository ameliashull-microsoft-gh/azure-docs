---
title: Deploy observability resources manually
description: How to configure observability features manually in Azure IoT Operations so that you can monitor your solution.
author: kgremban
ms.author: kgremban
ms.topic: how-to
ms.date: 02/27/2024

# CustomerIntent: As an IT admin or operator, I want to monitor and visualize data
# on the health of my industrial assets and edge environment.
---

# Deploy observability resources manually

[!INCLUDE [public-preview-note](../includes/public-preview-note.md)]

This article shows how to install and configure Azure IoT Operations observability components manually. This approach provides more options and control over your environment. For a simplified setup process that installs all the components you need to get started, see [Deploy observability resources with a script](howto-configure-observability.md).

## Configure your subscription

Run the following code to register providers with the subscription where your cluster is located.

>[!NOTE]
>This step only needs to be run once per subscription. To register resource providers, you need permission to do the `/register/action` operation, which is included in subscription Contributor and Owner roles. For more information, see [Azure resource providers and types](../../azure-resource-manager/management/resource-providers-and-types.md).

```azurecli
az account set -s <subscription-id>
az provider register -n "Microsoft.Insights"
az provider register -n "Microsoft.AlertsManagement"
```

## Install Azure Monitor managed service for Prometheus

Azure Monitor managed service for Prometheus is a component of Azure Monitor Metrics. This managed service provides flexibility in the types of metric data that you can collect and analyze with Azure Monitor. Prometheus metrics share some features with platform and custom metrics. Prometheus metrics also use some different features to better support open-source tools such as PromQL and Grafana.

Azure Monitor managed service for Prometheus allows you to collect and analyze metrics at scale using a Prometheus-compatible monitoring solution. This fully managed service is based on the Prometheus project from the Cloud Native Computing Foundation (CNCF). The service allows you to use the Prometheus query language (PromQL) to analyze and alert on the performance of monitored infrastructure and workloads, without having to operate the underlying infrastructure.

To set up Prometheus metrics collection for the new Arc-enabled cluster, follow the steps in [Configure Prometheus metrics collection](howto-configure-observability.md#configure-prometheus-metrics-collection).

## Install Container Insights

Container Insights monitors the performance of container workloads deployed to the cloud. It gives you performance visibility by collecting memory and processor metrics from controllers, nodes, and containers that are available in Kubernetes through the Metrics API. After you enable monitoring from Kubernetes clusters, metrics and container logs are automatically collected through a containerized version of the Log Analytics agent for Linux. Metrics are sent to the metrics database in Azure Monitor. Log data is sent to your Log Analytics workspace.

To monitor container workload performance, complete the steps to [enable container insights](/azure/azure-monitor/containers/kubernetes-monitoring-enable).

## Install Grafana

Azure Managed Grafana is a data visualization platform built on top of the Grafana software by Grafana Labs. Azure Managed Grafana is a fully managed Azure service operated and supported by Microsoft. Grafana helps you bring together metrics, logs and traces into a single user interface. With its extensive support for data sources and graphing capabilities, you can view and analyze your application and infrastructure telemetry data in real-time.

Azure IoT Operations provides a collection of dashboards designed to give you many of the visualizations you need to understand the health and performance of your Azure IoT Operations deployment.

To install Azure Managed Grafana, complete the following steps:

1. Use the Azure portal to [create an Azure Managed Grafana instance](../../managed-grafana/quickstart-managed-grafana-portal.md).

1. Configure an [Azure Monitor managed service for Prometheus as a data source for Azure Managed Grafana](/azure/azure-monitor/essentials/prometheus-grafana).

1. Configure the dashboards by following the steps in [Deploy dashboards to Grafana](howto-configure-observability.md#deploy-dashboards-to-grafana).

## Install OpenTelemetry (OTel) Collector

OpenTelemetry Collector is a key component in the OpenTelemetry project, which is an open-source observability framework aimed at providing unified tracing, metrics, and logging for distributed systems. The collector is designed to receive, process, and export telemetry data from multiple sources, such as applications and infrastructure, and send it to a monitoring backend. This OTel Collector collects metrics from Azure IoT Operations and makes it available to use with other observability tooling like Azure Monitor managed service for Prometheus, Container Insights, and Grafana.

To install OTel collector, complete the following steps:

[!INCLUDE [deploy-otel-collector.md](../includes/deploy-otel-collector.md)]

## Related content

- [Deploy observability resources with a script](howto-configure-observability.md)
