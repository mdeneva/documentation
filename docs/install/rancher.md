---
title: "Rancher"
linkTitle: "Rancher"
weight: 30
--- 

Ondat is a certified Rancher application. We offer two installation
methods:

* Rancher Catalogue - this is the easiest and requires just a few clicks
* Manual - allowing more control and visibility

> ⚠️ Before proceeding, ensure that you have followed our [prerequisites](/docs/prerequisites/).

> ⚠️ On Rancher, pay particular attention to
> the OS version and image used - some platforms require extra mainline kernel
> modules to be enabled.

## Catalog Install

Ondat is a Certified application in the [Rancher
Catalog](https://rancher.com/docs/rancher/v2.0-v2.4/en/helm-charts/). You can install
Ondat using the Rancher application install.

Before completing the steps below, you will need an etcd cluster. For
evaluation use our simple [test](/docs/prerequisites/etcd#testing) recipe.
For production installations, follow our [production](/docs/prerequisites/etcd#production) recipe.
Make a note of the etcd endpoint URL in either case.

1. Select the `System` project of your cluster

    ![install-1](/images/docs/rancher-ui-v2/rancher-1.png)

2. Select the `Apps` tab and click `Launch`

    ![install-2](/images/docs/rancher-ui-v2/rancher-2.png)

3. Search for Ondat and click on the App

    ![install-3](/images/docs/rancher-ui-v2/rancher-3.png)

    This will install the Ondat operator, which manages the Ondat
    DaemonSet.

4. Check and ammend installation options

    A generic configuration for Ondat is preset using the default values in
    the form. Be sure to check the etcd address and ensure it matches the value
    you noted at the beginning of this guide.

    The catalog form exposes several useful parameters - documented
    [below](/docs/install/rancher#simplecustomization}).

    For further customization, you can opt to set the option to 'Install
    Ondat Cluster' to false and install a custom CR. See [below](/docs/install/rancher#advancedcustomization) for this.

    ![install-4](/images/docs/rancher-ui-v2/rancher-4.png)

5. Launch the Ondat cluster

    ![install-8](/images/docs/rancher-ui-v2/rancher-8.png)

6. Verify the cluster bootstrap has successfully completed

    ![install-81](/images/docs/rancher-ui-v2/rancher-81.png)

7. License the newly installed cluster

    > ⚠️ Newly installed Ondat clusters must be licensed within 24 hours. Our
    > personal license is free, and supports up to 1TiB of provisioned storage.

    You will need access to the Ondat API on port 5705 of any of your nodes.
    For convenience, it is often easiest to port forward the service using the
    following kubectl incantation (this will block, so a second terminal window may
    be advisable):

      ```bash
      kubectl port-forward -n storageos svc/storageos 5705
      ```

    Now follow the instructions on our [licensing operations](/docs/operations/licensing)
    page to obtain and apply a license.

    Installation of Ondat is now complete.

### Simple Customization - Modify Catalog Form

The following options are exposed by the catalog form to allow some simple
customization of the Ondat installation.

![install-5](/images/docs/rancher-ui-v2/rancher-5.png)
![install-6](/images/docs/rancher-ui-v2/rancher-6.png)
![install-7](/images/docs/rancher-ui-v2/rancher-7.png)

* **Cluster Operator namespace**
: The Kubernetes namespace where the [Ondat Cluster Operator](/docs/reference/cluster-operator/) and other resources will be
created.
* **Container Images** : By default images are pulled from DockerHub, you can
* specify the image URLs
when using private registries.
* **Install Ondat cluster**
: Controls the automatic deployment of Ondat after installing the Cluster
Operator. If set to `false`, the Operator will be created, but a Custom Resource will
not be applied to the cluster. Launch the operator and proceed to the section
[Advanced Customization](/docs/install/rancher#advancedcustomization) below.
* **Namespace** : The Kubernetes namespace where Ondat will be
installed. By default, Ondat installs into the `storageos` namespace,
which will add a priority class to ensure high priority resource allocation.
Installing Ondat with the priority class prevents Ondat from being
evicted during periods of resource contention. It is inadvisable to modify this
under normal circumstances.
* **Username/Password** : Default Username and Password for the admin account
to be created at Ondat bootstrap. A random password will be generated by
leaving the field empty or clicking the `Generate` button.
* **External etcd address(es)** : Connection and configuration details for an
external Etcd cluster.See our documentation [here](/docs/prerequisites/etcd).
* **Node Selectors and Tolerations**
: Control placement of Ondat DaemonSet Pods. Ondat will only be
installed on the selected nodes.
* **Tolerations** : Define any tolerations you wish the DaemonSet to observe.

### Advanced Customization - Apply Custom CR

If `Install Ondat Cluster` was set to `false`, Ondat will not be
bootstrapped automatically. After the Ondat Operator is installed, you can
now create a Custom Resource that describes the Ondat cluster.

1. Select the `System Workloads` and `Import YAML`
    ![install-9](/images/docs/rancher-ui-v2/rancher-9.png)

1. Create the `Secret` and `CustomResource`
    ![install-10](/images/docs/rancher-ui-v2/rancher-10.png)
    ![install-11](/images/docs/rancher-ui-v2/rancher-11.png)
    ![install-12](/images/docs/rancher-ui-v2/rancher-12.png)

    This is an example.

    ```bash
    ---
    apiVersion: v1
    kind: Secret
    metadata:
      name: "storageos-api"
      labels:
        app: "storageos"
    type: "kubernetes.io/storageos"
    data:
      # echo -n '<secret>' | base64
      username: c3RvcmFnZW9z
      password: c3RvcmFnZW9z
    ---
    apiVersion: "storageos.com/v1"
    kind: StorageOSCluster
    metadata:
      name: "storageos"
    spec:
      # Ondat Pods are in storageos NS by default
      secretRefName: "storageos-api" # Reference from the Secret created in the previous step
      k8sDistro: "rancher"
      storageClassName: "ondat" # The storage class created by the Ondat operator is configurable
      images:
        nodeContainer: "storageos/node:< param latest_node_version >" # Ondat version
      kvBackend:
        address: 'storageos-etcd-client.etcd:2379' # Example address, change for your etcd endpoint
      # address: '10.42.15.23:2379,10.42.12.22:2379,10.42.13.16:2379' # You can set ETCD server ips
      sharedDir: '/var/lib/kubelet/plugins/kubernetes.io~storageos' # Needed when Kubelet as a container
      resources:
        requests:
          memory: "512Mi"
      nodeSelectorTerms:
        - matchExpressions:
          - key: "node-role.kubernetes.io/worker" # Compute node label will vary according to your installation
            operator: In
            values:
            - "true"
    ```

    > 💡 Additional `spec` parameters are available on the [Cluster Operator
    > configuration](/docs/reference/cluster-operator/configuration) page.

    > 💡 You can find more examples such as deployments referencing a external
    > etcd kv store for Ondat in the [Cluster Operator examples](/docs/reference/cluster-operator/examples) page.

## Manual Installation

### Install the storageos kubectl plugin

```
curl -sSLo kubectl-storageos.tar.gz \
    https://github.com/storageos/kubectl-storageos/releases/download/v1.0.0/kubectl-storageos_1.0.0_linux_amd64.tar.gz \
    && tar -xf kubectl-storageos.tar.gz \
    && chmod +x kubectl-storageos \
    && sudo mv kubectl-storageos /usr/local/bin/ \
    && rm kubectl-storageos.tar.gz
```

> 💡 You can find binaries for different architectures and systems in [kubectl
> plugin](https://github.com/storageos/kubectl-storageos/releases).

### Install Ondat

```bash
kubectl storageos install \
    --etcd-endpoints 'storageos-etcd-client.storageos-etcd:2379' \
    --admin-username "myuser" \
    --admin-password "my-password"
```

> 💡 Define the etcd endpoints as a comma delimited list, e.g. 10.42.3.10:2379,10.42.1.8:2379,10.42.2.8:2379

> 💡 If the etcd endpoints are not defined, the plugin will promt you and
> request the endpoints.

For more details on the installation options, check the [kubectl storageos plugin reference page](/docs/reference/kubectl-plugin).

## Verify Ondat installation

Ondat installs all its components in the `storageos` namespace.

```bash
$ kubectl -n storageos get pod -w
NAME                                     READY   STATUS    RESTARTS   AGE
storageos-api-manager-65f5c9dbdf-59p2j   1/1     Running   0          36s
storageos-api-manager-65f5c9dbdf-nhxg2   1/1     Running   0          36s
storageos-csi-helper-65dc8ff9d8-ddsh9    3/3     Running   0          36s
storageos-node-4njd4                     3/3     Running   0          55s
storageos-node-5qnl7                     3/3     Running   0          56s
storageos-node-7xc4s                     3/3     Running   0          52s
storageos-node-bkzkx                     3/3     Running   0          58s
storageos-node-gwp52                     3/3     Running   0          62s
storageos-node-zqkk7                     3/3     Running   0          62s
storageos-operator-8f7c946f8-npj7l       2/2     Running   0          64s
storageos-scheduler-86b979c6df-wndj4     1/1     Running   0          64s
```

> 💡 Wait until all the pods are ready. It usually takes ~60 seconds to complete

## License cluster

> ⚠️ Newly installed Ondat clusters must be licensed within 24 hours. Our
> personal license is free, and supports up to 1TiB of provisioned storage.

To obtain a license, follow the instructions on our [licensing operations](/docs/operations/licensing) page.

## First Ondat volume

If this is your first installation you may wish to follow the [Ondat Volume guide](/docs/operations/firstpvc) for an example of how
to mount an Ondat volume in a Pod.
