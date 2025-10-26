# onprem-k8s-monitoring

This repository provides a way to monitor multiple machines running in an on-premise environment using Grafana and Prometheus deployed on Kubernetes.

## Overview

This repository helps you build a monitoring environment with the following features:

-   Deploy `node-exporter` with Docker on the machines to be monitored to collect machine metrics.
-   Deploy `Prometheus` on a Kubernetes cluster to collect and store metrics from each machine's `node-exporter`.
-   Deploy `Grafana` on the Kubernetes cluster to visualize the metrics collected by `Prometheus`.
-   The machines to be monitored do not need to be nodes in the Kubernetes cluster.

<img src="docs/images/architecture.drawio.png">

## Directory Structure

```
.
├── helm-charts/
│   └── prometheus-custom/ # Custom Helm chart for deploying Prometheus
├── helmfiles/
│   └── grafana/           # Helmfile for deploying Grafana
└── node-exporter/
    └── docker-compose.yml # Docker Compose file for deploying node-exporter on each machine
```

## Prerequisites

-   **Monitored Machines**:
    -   Docker and Docker Compose must be installed.
-   **Kubernetes Cluster**:
    -   Must have network access to each machine where `node-exporter` is deployed.
    -   Helm must be available.
    -   Helmfile must be available.

## Setup Steps

### 1. Deploy Node Exporter

Start the `node-exporter` container on each machine you want to monitor.

1.  Copy the `node-exporter/` directory from this repository to the target machine.

2.  Set environment variables for `node-exporter`, such as its version. Choose one of the following methods:

    *   **Using a `.env` file:**

        Copy `.env.example` to create a `.env` file and edit the variables within it.

        ```bash
        cp .env.example .env
        # Edit the .env file
        ```

    *   **Setting environment variables at runtime:**

        Specify the environment variables before the `docker-compose` command.

        ```bash
        export NODE_EXPORTER_VERSION=v1.10.2
        ```

3.  Run the following command to start `node-exporter`.

    ```bash
    docker-compose up -d
    ```

### 2. Deploy Prometheus

Deploy Prometheus on your Kubernetes cluster.

1.  **Change to the working directory.**

    First, navigate to the directory containing the Prometheus Helm chart. Subsequent operations will be performed from this directory.

    ```bash
    cd helm-charts/prometheus-custom
    ```

2.  **Configure `node_exporter` scrape targets.**

    You need to set the list of IP addresses and ports for the machines running `node_exporter` in the `configmap.scrape_configs` section of `values.yaml`. It is recommended to manage this configuration by creating your own values file rather than editing the repository's `values.yaml` directly. There are two main ways to do this:

    *   **Method A: Create a new values file**

        Copy `values.yaml` to a new file, such as `values-local.yaml`, and edit the `targets` under `configmap.scrape_configs` to match your environment.

        **Example `values-local.yaml`:**
        ```yaml
        # ... (other settings omitted) ...
        configmap:
          # ...
          scrape_configs:
            - job_name: 'prometheus'
              static_configs:
                - targets: ['localhost:9090']
            - job_name: 'node_exporters'
              static_configs:
                - targets:
                    - '192.168.1.1:9100' # Machine 1
                    - '192.168.1.2:9100' # Machine 2
                    # ... add other machines to monitor ...
        ```

    *   **Method B: Specify the list of targets in a separate file**

        Create a YAML file (e.g., `targets.yaml`) that contains only the list of monitoring targets.

        **Example `targets.yaml`:**
        ```yaml
        - '192.168.1.1:9100'
        - '192.168.1.2:9100'
        ```
        With this method, you will use the `--set-file` flag in the `helm install` command to override the specific key in `values.yaml` with the content of this file.

3.  **Deploy Prometheus.**

    Run one of the following commands to deploy Prometheus.

    *   **For Method A:**
        ```bash
        helm install prometheus . \
          --namespace monitoring \
          --create-namespace \
          -f values-local.yaml
        ```

    *   **For Method B:**
        ```bash
        helm install prometheus . \
          --namespace monitoring \
          --create-namespace \
          --set-file configmap.scrape_configs[1].static_configs[0].targets=targets.yaml
        ```

### 3. Deploy Grafana

Deploy Grafana on your Kubernetes cluster.

1.  **Change to the working directory.**

    First, navigate to the directory containing the Grafana Helmfiles. Subsequent operations will be performed from this directory.

    ```bash
    cd helmfiles
    ```

2.  **Prepare the configuration files.**

    Create the configuration files for `helmfile` and the Grafana Helm chart by copying the examples.

    *   **Helmfile variables file:**
        Copy `vars/grafana.yaml.example` to `vars/grafana.yaml` and edit the content to match your environment.

    *   **Grafana chart values file:**
        Copy `values/grafana.yaml.example` to `values/grafana.yaml` and edit the values you want to set for Grafana.

3.  **Deploy Grafana.**

    Run the following command to deploy Grafana.

    ```bash
    helmfile apply .
    ```

This completes the setup. You can now access Grafana and add Prometheus as a data source to visualize the metrics of your machines.