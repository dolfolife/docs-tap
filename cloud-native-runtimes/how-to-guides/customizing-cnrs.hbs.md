# Customizing Cloud Native Runtimes

There are many package configuration options exposed through data values that allows you to customize your Cloud Native Runtimes installation.

The following command yields all the configuration options available in a given Cloud Native Runtimes package version.

```sh
export CNR_VERSION=2.4.0
tanzu package available get cnrs.tanzu.vmware.com/${CNR_VERSION} --values-schema -n tap-install
```

## Customizing Cloud Native Runtimes

Besides utilizing the out-of-the-box options to configure your package, you can use ytt overlays to further customize your installation.
See [Customize your package installation](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.6/tap/customize-package-installation.html)
for instructions on how to customize any Tanzu Platform Application package.

This section will provide an example on how to update the Knative ConfigMap `config-logging` to override the logging level
of the Knative Serving controller to `debug`.

1. Create a Kubernetes secret containing the ytt overlay by applying the configuration below to your cluster.

    ```shell
    kubectl apply -n tap-install -f - << EOF
    apiVersion: v1
    kind: Secret
    metadata:
     name: cnrs-patch
    stringData:
     patch.yaml: |
       #@ load("@ytt:overlay", "overlay")
       #@overlay/match by=overlay.subset({"kind":"ConfigMap","metadata":{"name":"config-logging","namespace":"knative-serving"}})
       ---
       data:
         #@overlay/match missing_ok=True
         loglevel.controller: "debug"
    EOF
    ```

    To learn more about the Carvel tool `ytt` and how to write overlays, see their [official documentation](https://carvel.dev/ytt/).

2. Update your tap-values.yaml file to add the snippet below.

    The section below informs the Tanzu Application Platform about the secret name where the overlay is stored and also, to apply the overlay to the `cnrs` package.

    ```yaml
    package_overlays:
    - name: cnrs
      secrets:
      - name: cnrs-patch
    ```
   
   **Tip**: You can retrieve your tap-values.yaml file by running the command below.

   ```shell
   kubectl get secret tap-tap-install-values -n tap-install -ojsonpath="{.data.tap-values\.yaml}" | base64 -d
   ```

3. Update the Tanzu Application Platform installation.

    ```sh
    tanzu package installed update tap -p tap.tanzu.vmware.com -v ${TAP_VERSION} --values-file tap-values.yaml -n tap-install
    ```

4. Confirm your changes were applied to the corresponding ConfigMap.

    By running the command below, you can check if your changes were applied to the ConfigMap `config-logging`
    by ensuring `loglevel.controller` is set to `debug`.

    ```sh
    kubectl get configmap config-logging -n knative-serving -oyaml
    ```
