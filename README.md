KOTS HELM CLI
==================

Example project that shows how to package the HELM CLI as a KOTS application. In this example, the HELM CLI is used to install and upgrade a Grafana Helm Chart. 

**NOTE - This is an experimanetal project and NOT the way KOTS is currently designed to deploy Helm Charts. To learn about how KOTS supports Helm Charts, please review https://kots.io/vendor/guides/helm-chart/ and https://kots.io/vendor/helm/using-helm-charts/.**

### Desired Outcome

The desired outcome of this is to learn if it is feasable to have KOTS deploy an application that only consists of the HELM CLI and the charts it will need to install and or upgrade. In this scenario, the deployment of the actual application (Grafana) is managed by the Helm CLI and not KOTS as KOTS will simply manage the deployment of the Helm CLI.

### Helm CLI Container

The container is built using a Dockerfile based on the one found in this [GitHub Repository](https://github.com/alpine-docker/helm/blob/master/README.md). The only modifications are that the `ENTRYPOINT` and `CMD` lines are removed (since these will be passed in the Pod Definition file) and added a `COPY` command to copy the Grafana chart at build time. The Grafana chart is included in the [app](https://github.com/cremerfc/helm-cli-kots/tree/main/app) directory of this repo, which also contains the [Dockerfile](https://github.com/cremerfc/helm-cli-kots/blob/main/app/Dockerfile).

The reason for including the chart at build time is for airgap installations. While the chart could be provided to the end user by other means and then mount it later, this could add complexity and possible points of failure. 

#### Defining the Application in KOTS

An application in KOTS is basically a set of YAML files. In this case, this is a very small and simple application that only consists of the container above. This container will then be tasked with deploying the actual Chart.

Because of the nature of this Application, we are deploying this container as a [Kubernets Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/) and is defined [here](https://github.com/cremerfc/helm-cli-kots/blob/main/manifests/helm-cli-job.yaml). In order to have this Job properly recreated each time an upgrade is performed by KOTS, or the application is "Redeployed", we added the KOTS [label annotation](https://kots.io/vendor/packaging/cleaning-up-jobs/) that tells KOTS to delete the job once it finishes. If this is not included, the Job will remain and any upgrades of the Job will fail.

#### Giving the Job the Proper Permissions

To ensure that the Pod executing the commands has the proper permissions to install/upgrade the Helm chart in the cluster we add the following to the Job defintion file:
```yaml
      serviceAccountName: kotsadm
      automountServiceAccountToken: true

```

#### Executing the Helm Commands

The following lines are added to the Job defintion file:

```yaml

       command: ["/bin/sh"]
       args: ["-c", "helm upgrade --install grafana /apps/grafana-6.1.16.tgz -f kots-values.yaml"]

```

We are using the `Upgrade` command with the `--install` option which allows us to handle both a new install and an upgrade of the chart. The `kots-values.yaml` file is how we pass any and all values we want to override when deploying in this manner.

The `kots-values.yaml` file is mounted as a Kubernetes `ConfigMap` which is defined as part of the application.

Since this is more of an experiment and things are bound to go wrong, we need as much information when troubleshooting. To help us, we added the following to the job definition file:

```yaml
           terminationMessagePolicy: FallbackToLogsOnError
```

This setting will write the last chunck of the pod's log (2048 bytes or 80 lines, whichever is smaller) to the [termination-log](https://kubernetes.io/docs/tasks/debug-application-cluster/determine-reason-pod-failure/). This came in really handy when the container would fail due to a Helm error.


##### Managing Containers

Since this KOTS application is comprised of only one container, KOTS does not know about the containers that the Helm Chart will request. If the containers are public, like the containers in the Grafana chart are, then that is OK.

However, this will not work if the containers are in a private registry and if the appliation needs to be installed in an airgap environment. Let's discuss how to handle airgap first.

In order to have KOTS include these containers in the airgap bundle we need to add them to the `AdditionalImages` [section](https://kots.io/reference/v1beta1/application/#additionalimages) of the KOTS [Application](https://kots.io/reference/v1beta1/application/) Defintion file. 

Below are all of the images referenced in the Chart's `Values.yaml` file added to the [replicated-app.yaml](https://github.com/cremerfc/helm-cli-kots/blob/main/manifests/replicated-app.yaml) file:

```yaml
  additionalImages: 
    - grafana/grafana:7.3.5
    - bats/bats:v1.1.0
    - curlimages/curl:7.73.0
    - busybox:1.31.1
    - kiwigrid/k8s-sidecar:1.1.0
    - grafana/grafana-image-renderer:latest
```


#### Managing Private Containers

If the images are in your private repository, the tag for your repository will depend on how you choose to manage them. Remember, when the Helm Chart is deployed, KOTS is no longer managing this process. This means that KOTS can not automatically handle your images tag to account for where to request it from.

For example, if you would like to use the Replicated Registry (as a proxy or to store your images), you will need to append/change the image tag to account for this. Furthermore, when this is deployed in an airgap environment we will also need to manually account for changing the tag to point to the local registry.

To do this, we simply take advantage of the ConfigMap we are using to override the image tag. In the Grafana Chart, all the images are defined in the `Values.yaml` file which makes it really easy for us to manage.

Below is how we handle the `grafana` image in the configMap:

```yaml
           image:
             repository: {{repl if HasLocalRegistry }}{{repl LocalRegistryAddress}}{{repl else}}grafana{{repl end}}/grafana
             pullSecrets:
             - your-registry-secret-if-needed
```

Note that in the example above, the default value for the `grafana` image is simply `grafana/grafana`, which works in an online install. However, in an airgap install the `grafana` repository will instead be the local registry. 

To account for airgap installs, we use the `{{repl if` statement to determine if there is a [local registry](https://kots.io/reference/template-functions/config-context/#haslocalregistry) defined (this is always true for airgap installs). If true, we use the [local registry address](https://kots.io/reference/template-functions/config-context/#localregistryaddress) to replace the public `grafana` repository in the image tag with the local registry.

In the case of a private repository, instead of `grafana` we would replace this with something like `registry.replicated.com/your-app-slug/` if you use the Replicated registry to store the images, or with your own private registry address. Note that you may also need to include a `pullSecret` as well.

#### Overriding Values at Install/Upgrade Time

As mentioned above, values that we want to override at install/upgrade time will be passed in the `kots-values.yaml` file, and its contents are defined in the [helm-values-config-map](https://github.com/cremerfc/helm-cli-kots/blob/main/manifests/helm-values-config-map.yaml) file.

One of the advantages of using KOTS, is that it can provide the end user with a web UI to enter the values to use at runtime. To do this we use the KOTS [Config](https://kots.io/reference/v1beta1/config/) custom resource as defined in the [Config.yaml](https://github.com/cremerfc/helm-cli-kots/blob/main/manifests/config.yaml) file in this repository. Those values are then mapped to the [helm-values-config-map](https://github.com/cremerfc/helm-cli-kots/blob/main/manifests/helm-values-config-map.yaml) file.

