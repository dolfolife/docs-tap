# Install Cloud Native Runtimes

This document describes how you can install Cloud Native Runtimes, commonly known as CNR, from the Tanzu Application Platform package repository.

>**Note:** Use the instructions on this page if you do not want to use a profile to install packages.
Both the full and light profiles include Cloud Native Runtimes.
For more information about profiles, see [Installing the Tanzu Application Platform Package and Profiles](https://docs.vmware.com/en/Tanzu-Application-Platform/1.5/tap/install.html).

## <a id='cnr-prereqs'></a>Prerequisites

Before installing Cloud Native Runtimes:

- Complete all prerequisites to install Tanzu Application Platform. For more information, see [Prerequisites](https://docs.vmware.com/en/Tanzu-Application-Platform/1.5/tap/prerequisites.html).
- Contour is installed in the cluster. Contour can be installed from the [Tanzu Application package repository](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/cert-mgr-contour-fcd-install-cert-mgr.html#install-contour-2). If you have have an existing Contour installation, see [Installing Cloud Native Runtimes with an Existing Contour Installation](https://docs.vmware.com/en/Cloud-Native-Runtimes-for-VMware-Tanzu/2.2/tanzu-cloud-native-runtimes/contour.html).

- By default, Tanzu Application Platform installs and uses a self-signed certificate authority for issuing TLS certificates to components by using ingress issuer. For more information, see [Ingress Certificates](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/security-and-compliance-tls-and-certificates-ingress-about.html).
  To successfully install Cloud Native Runtimes, `shared.ingress_domain` or `cnrs.domain_name` property is required to be set when `ingress_issuer` property is set. For example:

  ```sh
  shared:
    ingress_domain: "foo.bar.com"
  ```

  or

  ```sh
  cnrs:
    domain_name: "foo.bar.com"
  ```

  If the domain name is not available or desired, domain name can be set to any valid value as long as no process is relying on the domain name resolving to the envoy IP.
  (Not recommended for production environments) Another alternative to bypass setting domain name is to disable auto-TLS. For more information, see [Disabling Automatic TLS Certificate Provisioning](../auto-tls/tls-guides-deactivate-autotls.hbs.md).

## <a id='cnr-install'></a> Install

To install Cloud Native Runtimes:

1. List version information for the package by running:

    ```
    tanzu package available list cnrs.tanzu.vmware.com --namespace tap-install
    ```

     For example:

    ```
    tanzu package available list cnrs.tanzu.vmware.com --namespace tap-install

      NAME                   VERSION  RELEASED-AT
      cnrs.tanzu.vmware.com  2.4.0    2023-06-05 19:00:00 -0500 -05 
    ```

1. (Optional) Make changes to the default installation settings:

    1. Gather values schema.

        ```
        tanzu package available get cnrs.tanzu.vmware.com/2.4.0 --values-schema -n tap-install
        ```

        For example:

        ```
        tanzu package available get cnrs.tanzu.vmware.com/2.4.0 --values-schema -n tap-install
       
        KEY                            DEFAULT                               TYPE     DESCRIPTION                                                                       
        domain_config                  <nil>                                 <nil>    Optional. Overrides the Knative Serving "config-domain" ConfigMap, allowing you to map Knative Services to specific domains. Must be valid YAML and conform to the "config-domain" specification.
        namespace_selector                                                   string   Specifies a LabelSelector which determines which namespaces should have a wildcard certificate provisioned. Set this property only if the Cluster issuer is type DNS-01 challenge.
        pdb.enable                     true                                  <nil>    Optional. Set to true to enable a PodDisruptionBudget for the Knative Serving activator and webhook deployments.
        domain_name                                                          string   Optional. Default domain name for Knative Services.
        ingress.external.namespace     tanzu-system-ingress                  string   Required. Specify a namespace where an existing Contour is installed on your cluster. CNR will use this Contour instance for external services.
        ingress.internal.namespace     tanzu-system-ingress                  string   Required. Specify a namespace where an existing Contour is installed on your cluster. CNR will use this Contour instance for internal services.
        lite.enable                    false                                 <nil>    Optional. Set to "true" to enable lite mode. Reduces CPU and Memory resource requests for all cnrs Deployments, Daemonsets, and StatefulSets by half. Not recommended for production.
        domain_template                {{.Name}}.{{.Namespace}}.{{.Domain}}  string   Optional. Specifies the golang text template string to use when constructing the DNS name for a Knative Service.
        kubernetes_distribution        <nil>                                 <nil>    Optional. Type of K8s infrastructure being used. Supported Values: openshift
        kubernetes_version             0.0.0                                 <nil>    Optional. Version of K8s infrastructure being used. Supported Values: valid Kubernetes major.minor.patch versions
        allow_manual_configmap_update  true                                  boolean  Specifies how updates to some CNRs ConfigMaps can be made. Set to True, CNRs allows updates to those ConfigMaps to be made only manually. Set to False, updates to those CNRs ConfigMaps can be made only using overlays. Supported Values: True, False.
        ca_cert_data                                                         string   Optional. PEM Encoded certificate data to trust TLS connections with a private CA.
        default_external_scheme        <nil>                                 string   Optional. Specifies the default scheme to use for Knative Service URLs, regardless of other TLS configurations. Supports either http or https. Cannot be set along with default_tls_secret
        default_tls_secret                                                   string   Optional. Specify a fallback TLS Certificate for use by Knative Services if autoTLS is disabled. Will set default exterenal scheme for Knative Service URLs to "https". Requires either "domain_name" or "domain_config" to be set and cannot be set along with "default_external_scheme".
        https_redirection              true                                  boolean  CNRs ingress will send a 301 redirect for all http connections, asking the clients to use HTTPS
        ingress_issuer                                                       string   Cluster issuer to be used in CNRs. To use this property the domain_name or domain_config must be set. Under the hood, when this property is set auto-tls is Enabled.
        ```

    1. Create a `cnr-values.yaml` file by using the following sample as a guide to configure Cloud Native Runtimes:

        >**Note:** For most installations, you can leave the `cnr-values.yaml` empty, and use the default values.
        ```
        ---
        # Configures the domain that Knative Services will use
        domain_name: "mydomain.com"
        ```

       **Configuration Notes**:
       * If you are running on a single-node cluster, such as minikube, set the `lite.enable: true`
        option to lower CPU and memory requests for resources. In case you also want to deactivate pod disruption budgets
        on Knative Serving and high availability is not indispensable in your development environment, you can set `pbd.enable` to `false`.

        * Cloud Native Runtimes reuses the existing `tanzu-system-ingress` Contour installation for
        external and internal access when installed in the `light` or `full` profile.
        If you want to use a separate Contour installation for system-internal traffic, set
        `cnrs.contour.internal.namespace` to the namespace of your separate Contour installation.

        * If you install Cloud Native Runtimes with the default value of `true` for the `allow_manual_configmap_update` configuration, you will only be able to update some ConfigMaps (Ex: config-features manually. If you would like to update all ConfigMaps using overlays, please change this value to `false`. In a future release, `false` will be the default configuration. At some point after that, Cloud Native Runtimes will be released without the option to switch and `false` will be the permanent behavior.

1. Install the package by running:

    ```
    tanzu package install cloud-native-runtimes -p cnrs.tanzu.vmware.com -v 2.4.0 -n tap-install -f cnr-values.yaml --poll-timeout 30m
    ```

    For example:

    ```
    tanzu package install cloud-native-runtimes -p cnrs.tanzu.vmware.com -v 2.4.0 -n tap-install -f cnr-values.yaml --poll-timeout 30m

    | Installing package 'cnrs.tanzu.vmware.com'
    | Getting package metadata for 'cnrs.tanzu.vmware.com'
    | Creating service account 'cloud-native-runtimes-tap-install-sa'
    | Creating cluster admin role 'cloud-native-runtimes-tap-install-cluster-role'
    | Creating cluster role binding 'cloud-native-runtimes-tap-install-cluster-rolebinding'
    - Creating package resource
    - Package install status: Reconciling

     Added installed package 'cloud-native-runtimes' in namespace 'tap-install'
    ```

1. Verify the package install by running:

    ```
    tanzu package installed get cloud-native-runtimes -n tap-install
    ```

    For example:

    ```
    tanzu package installed get cloud-native-runtimes -n tap-install

    Retrieving installation details for cloud-native-runtimes...
    NAME:                    cloud-native-runtimes
    PACKAGE-NAME:            cnrs.tanzu.vmware.com
    PACKAGE-VERSION:         2.4.0
    STATUS:                  Reconcile succeeded
    CONDITIONS:              [{ReconcileSucceeded True  }]
    USEFUL-ERROR-MESSAGE:
    ```

    Verify that `STATUS` is `Reconcile succeeded`.
