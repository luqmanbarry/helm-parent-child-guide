# Helm Parent Child Example

A Helm chart to demonstrate a parent-child (sub-chart) setup for deploying apps with dependencies on one or more other charts.

There are 3 main scenarios this setup can be applied to:

- Parent objects/manifests deploy before chart dependencies: Annotate the manifests defined in the `templates` directory with `"helm.sh/hook": "pre-install, pre-upgrade"` if you want the parent chart resources to be applied before the dependencies.
- Parent objects/manifests deploy after chart dependencies: Annotate the manifests defined in the `templates/` directory with `"helm.sh/hook": "post-install, post-upgrade"` to get the parent chart resources to apply after the dependencies.
- Some parent resources deploy before, some after chart dependencies: Annotate resources (ie: Secrets, ConfigMaps, CRDs) that should deploy before dependencies with the `"helm.sh/hook": "pre-install, pre-upgrade"` hook; and add `"helm.sh/hook": "post-install, post-upgrade"` to those resources you want deployed after the dependencies charts.

You can use helm hook weight to further organize the deployment order of the manifests. For example, if I have 2 manifests I want deployed after the dependencies, with Manifest1 deployed before Manifest2, I could achieve it as:

- Manifest1:
    ```yaml
    <...>
    metadata:
      annotations:
        "helm.sh/hook": "post-install, post-upgrade"
        "helm.sh/hook-weight": "1"  # Hook Weight
    <...>
    ```
- Manifest2:
    ```yaml
    <...>
    metadata:
      annotations:
        "helm.sh/hook": "post-install, post-upgrade"
        "helm.sh/hook-weight": "2"  # Hook Weight
    <...>
    ```

## Pre-requisites
- Software Packages: Helm, kubectl/openshift-client, jq
- A Kubernetes/OpenShift cluster

## Procedure

1. Declare your dependencies in the [Chart.yaml](./Chart.yaml) file
    ```yaml
    apiVersion: v2
    name: helm-parent-child-example
    description: A Helm chart to demonstrate a parent-child setup to deploy apps with dependencies on other charts.

    type: application

    version: 0.1.0

    appVersion: "1.16.0"

    dependencies:
      - name: mysql
        version: "12.2.4"
        repository: "https://charts.bitnami.com/bitnami"
      - name: wordpress
        version: "24.1.12"
        repository: "https://charts.bitnami.com/bitnami"
    ```

2. Prepare the the values file: `values.yaml` for common values, `values.<env-name>.yaml` for env specific values.
    ```yaml
    # SUB CHARTS DEFINITION
    mysql:  # use sub-chart name and add its values as children of this key (indented in)
      # Chart Params: https://artifacthub.io/packages/helm/bitnami/mysql
      global:
        defaultStorageClass: "azure-csi"
        compatibility:
          openshift:
            adaptSecurityContext: auto
      nameOverride: "mysql-sub-chart-demo"
      auth:
        rootPassword: ""
        createDatabase: true
        database: "sub-chart-demo"
        username: "sub-chart-demo"
        password: "sub-chart-demo"

    wordpress:  # use sub-chart name and add its values as children of this key (indented in)
      # Chart Params: https://artifacthub.io/packages/helm/bitnami/wordpress
      global:
        defaultStorageClass: "azure-csi"
        compatibility:
          openshift:
            adaptSecurityContext: auto
      nameOverride: "wordpress-sub-chart-demo"
      wordpressUsername: user
      wordpressPassword: ""
      wordpressEmail: user@example.com
      wordpressFirstName: FirstName
      wordpressLastName: LastName
      wordpressBlogName: LuqmanBarry's Blog!
      mariadb:
        enabled: false
      externalDatabase:
        host: "mysql-sub-chart-demo"
        port: 3306
        user: "sub-chart-demo"
        password: "sub-chart-demo"
        database: "sub-chart-demo"

    # PARENT CHART DEFINITION
    business_unit: sales
    aws_region: us-east-1
    storageClassName: managed-csi
    micro_services:
      - name: luqman-eshop
        namespace: luqman-eshop
        pvc_name: luqman-eshop
        pvc_size: 20Gi
        image: image-registry.openshift-image-registry.svc:5000/openshift/nginx

    ```

3. Download the helm dependencies - They should not be committed to git (use .`gitignore`)
    ```sh
    helm repo add bitnami https://charts.bitnami.com/bitnami
    helm repo update
    helm dependency update
    ```
4. Lint the chart - Fix errors if any
    ```sh
    helm lint
    ```

5. Print the chart Template file to verify the hooks were applied properly
   ```sh
   helm template parent-child-demo . > template.yaml
   ```

6. Verify the helm template command output meets expectations
7. Deploy the helm chart to a test environment for further verification