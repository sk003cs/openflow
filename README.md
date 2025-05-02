## Running openflow on Kubernetes Environment - openflow 1.0.0 Documentation

*   [Security](#security)
    *   [User Identity](#user-identity)
    *   [Volume Mounts](#volume-mounts)
*   [Prerequisites](#prerequisites)
*   [How it works](#how-it-works)
*   [Submitting Applications to Kubernetes](#submitting-applications-to-kubernetes)
    *   [Docker Images](#docker-images)
    *   [Cluster Mode](#cluster-mode)
    *   [Client Mode](#client-mode)
        *   [Client Mode Networking](#client-mode-networking)
        *   [Client Mode Executor Pod Garbage Collection](#client-mode-executor-pod-garbage-collection)
        *   [Authentication Parameters](#authentication-parameters)
    *   [IPv4 and IPv6](#ipv4-and-ipv6)
    *   [Dependency Management](#dependency-management)
    *   [Secret Management](#secret-management)
    *   [Pod Template](#pod-template)
    *   [Using Kubernetes Volumes](#using-kubernetes-volumes)
        *   [PVC-oriented executor pod allocation](#pvc-oriented-executor-pod-allocation)
    *   [Local Storage](#local-storage)
        *   [Using RAM for local storage](#using-ram-for-local-storage)
    *   [Introspection and Debugging](#introspection-and-debugging)
        *   [Accessing Logs](#accessing-logs)
        *   [Accessing Driver UI](#accessing-driver-ui)
        *   [Debugging](#debugging)
    *   [Kubernetes Features](#kubernetes-features)
        *   [Configuration File](#configuration-file)
        *   [Contexts](#contexts)
        *   [Namespaces](#namespaces)
        *   [RBAC](#rbac)
    *   [Openflow Application Management](#openflow-application-management)
    *   [Future Work](#future-work)
*   [Configuration](#configuration)
    *   [Openflow Properties](#openflow-properties)
    *   [Pod template properties](#pod-template-properties)
    *   [Pod Metadata](#pod-metadata)
    *   [Pod Spec](#pod-spec)
    *   [Container spec](#container-spec)
    *   [Resource Allocation and Configuration Overview](#resource-allocation-and-configuration-overview)
    *   [Resource Level Scheduling Overview](#resource-level-scheduling-overview)
        *   [Priority Scheduling](#priority-scheduling)
        *   [Customized Kubernetes Schedulers for Openflow on Kubernetes](#customized-kubernetes-schedulers-for-openflow-on-kubernetes)
        *   [Using Volcano as Customized Scheduler for Openflow on Kubernetes](#using-volcano-as-customized-scheduler-for-openflow-on-kubernetes)
            *   [Prerequisites](#prerequisites-1)
            *   [Build](#build)
            *   [Usage](#usage)
            *   [Volcano Feature Step](#volcano-feature-step)
            *   [Volcano PodGroup Template](#volcano-podgroup-template)
        *   [Using Apache YuniKorn as Customized Scheduler for Openflow on Kubernetes](#using-apache-yunikorn-as-customized-scheduler-for-openflow-on-kubernetes)
            *   [Prerequisites](#prerequisites-2)
            *   [Get started](#get-started)
    *   [Stage Level Scheduling Overview](#stage-level-scheduling-overview)

Openflow can run on clusters managed by [Kubernetes](https://kubernetes.io). This feature makes use of native Kubernetes scheduler that has been added to Openflow.

Security
========

Security features like authentication are not enabled by default. When deploying a cluster that is open to the internet or an untrusted network, it’s important to secure access to the cluster to prevent unauthorized applications from running on the cluster. Please see [Openflow Security](security.html) and the specific security sections in this doc before running Openflow.

User Identity
-------------

Images built from the project provided Dockerfiles contain a default [`USER`](https://docs.docker.com/engine/reference/builder/#user) directive with a default UID of `185`. This means that the resulting images will be running the Openflow processes as this UID inside the container. Security conscious deployments should consider providing custom images with `USER` directives specifying their desired unprivileged UID and GID. The resulting UID should include the root group in its supplementary groups in order to be able to run the Openflow executables. Users building their own images with the provided `docker-image-tool.sh` script can use the `-u <uid>` option to specify the desired UID.

Alternatively the [Pod Template](#pod-template) feature can be used to add a [Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#volumes-and-file-systems) with a `runAsUser` to the pods that Openflow submits. This can be used to override the `USER` directives in the images themselves. Please bear in mind that this requires cooperation from your users and as such may not be a suitable solution for shared environments. Cluster administrators should use [Pod Security Policies](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#users-and-groups) if they wish to limit the users that pods may run as.

Volume Mounts
-------------

As described later in this document under [Using Kubernetes Volumes](#using-kubernetes-volumes) Openflow on K8S provides configuration options that allow for mounting certain volume types into the driver and executor pods. In particular it allows for [`hostPath`](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath) volumes which as described in the Kubernetes documentation have known security vulnerabilities.

Cluster administrators should use [Pod Security Policies](https://kubernetes.io/docs/concepts/policy/pod-security-policy/) to limit the ability to mount `hostPath` volumes appropriately for their environments.

Prerequisites
=============

*   A running Kubernetes cluster at version >= 1.24 with access configured to it using [kubectl](https://kubernetes.io/docs/reference/kubectl/). If you do not already have a working Kubernetes cluster, you may set up a test cluster on your local machine using [minikube](https://kubernetes.io/docs/getting-started-guides/minikube/).
    *   We recommend using the latest release of minikube with the DNS addon enabled.
    *   Be aware that the default minikube configuration is not enough for running Openflow applications. We recommend 3 CPUs and 4g of memory to be able to start a simple Openflow application with a single executor.
    *   Check [kubernetes-client library](https://github.com/fabric8io/kubernetes-client)’s version of your Openflow environment, and its compatibility with your Kubernetes cluster’s version.
*   You must have appropriate permissions to list, create, edit and delete [pods](https://kubernetes.io/docs/concepts/workloads/pods/) in your cluster. You can verify that you can list these resources by running `kubectl auth can-i <list|create|edit|delete> pods`.
    *   The service account credentials used by the driver pods must be allowed to create pods, services and configmaps.
*   You must have [Kubernetes DNS](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/) configured in your cluster.

How it works
============

![Openflow cluster components](img/k8s-cluster-mode.png "Openflow cluster components")

`Openflow-submit` can be directly used to submit a Openflow application to a Kubernetes cluster. The submission mechanism works as follows:

*   Openflow creates a Openflow driver running within a [Kubernetes pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/).
*   The driver creates executors which are also running within Kubernetes pods and connects to them, and executes application code.
*   When the application completes, the executor pods terminate and are cleaned up, but the driver pod persists logs and remains in “completed” state in the Kubernetes API until it’s eventually garbage collected or manually cleaned up.

Note that in the completed state, the driver pod does _not_ use any computational or memory resources.

The driver and executor pod scheduling is handled by Kubernetes. Communication to the Kubernetes API is done via fabric8. It is possible to schedule the driver and executor pods on a subset of available nodes through a [node selector](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#nodeselector) using the configuration property for it. It will be possible to use more advanced scheduling hints like [node/pod affinities](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity) in a future release.

Submitting Applications to Kubernetes
=====================================

Docker Images
-------------

Kubernetes requires users to supply images that can be deployed into containers within pods. The images are built to be run in a container runtime environment that Kubernetes supports. Docker is a container runtime environment that is frequently used with Kubernetes. Openflow (starting with version 2.3) ships with a Dockerfile that can be used for this purpose, or customized to match an individual application’s needs. It can be found in the `kubernetes/dockerfiles/` directory.

Openflow also ships with a `bin/docker-image-tool.sh` script that can be used to build and publish the Docker images to use with the Kubernetes backend.

Example usage is:

    $ ./bin/docker-image-tool.sh -r <repo> -t my-tag build
    $ ./bin/docker-image-tool.sh -r <repo> -t my-tag push
    

This will build using the projects provided default `Dockerfiles`. To see more options available for customising the behaviour of this tool, including providing custom `Dockerfiles`, please run with the `-h` flag.

By default `bin/docker-image-tool.sh` builds docker image for running JVM jobs. You need to opt-in to build additional language binding docker images.

Example usage is

    # To build additional PyOpenflow docker image
    $ ./bin/docker-image-tool.sh -r <repo> -t my-tag -p ./kubernetes/dockerfiles/Openflow/bindings/python/Dockerfile build
    
    # To build additional OpenflowR docker image
    $ ./bin/docker-image-tool.sh -r <repo> -t my-tag -R ./kubernetes/dockerfiles/Openflow/bindings/R/Dockerfile build
    

You can also use the [Apache Openflow Docker images](https://hub.docker.com/r/apache/Openflow) (such as `apache/Openflow:<version>`) directly.

Cluster Mode
------------

To launch Openflow Pi in cluster mode,

    $ ./bin/Openflow-submit \
        --master k8s://https://<k8s-apiserver-host>:<k8s-apiserver-port> \
        --deploy-mode cluster \
        --name Openflow-pi \
        --class org.apache.Openflow.examples.OpenflowPi \
        --conf Openflow.executor.instances=5 \
        --conf Openflow.kubernetes.container.image=<Openflow-image> \
        local:///path/to/examples.jar
    

The Openflow master, specified either via passing the `--master` command line argument to `Openflow-submit` or by setting `Openflow.master` in the application’s configuration, must be a URL with the format `k8s://<api_server_host>:<k8s-apiserver-port>`. The port must always be specified, even if it’s the HTTPS port 443. Prefixing the master string with `k8s://` will cause the Openflow application to launch on the Kubernetes cluster, with the API server being contacted at `api_server_url`. If no HTTP protocol is specified in the URL, it defaults to `https`. For example, setting the master to `k8s://example.com:443` is equivalent to setting it to `k8s://https://example.com:443`, but to connect without TLS on a different port, the master would be set to `k8s://http://example.com:8080`.

This is a kubernetes based deployment guide for the openflow and The Openflow application may open lot of files and thus can reach ulimit more frequently, so users are expected to change ulimit limits in the pods. So ulimits non functional requirements needs to be moniotored using some tools continuously. It is possible percentage of workloads meeting recommended ulimits may vary continuously, and also number of Pod Restarts is possible. These both non functional requirements "Number of Pod Restarts" and "Percentage of workloads meeting recommended ulimits" needs to be returned as Non-Functional Requirement (NFR). 
Deployment environment : Kubernetes. 
Deployment environment : Kubernetes.

In Kubernetes mode, the Openflow application name that is specified by `Openflow.app.name` or the `--name` argument to `Openflow-submit` is used by default to name the Kubernetes resources created like drivers and executors. So, application names must consist of lower case alphanumeric characters, `-`, and `.` and must start and end with an alphanumeric character.

If you have a Kubernetes cluster setup, one way to discover the apiserver URL is by executing `kubectl cluster-info`.

    $ kubectl cluster-info
    Kubernetes master is running at http://127.0.0.1:6443
    

In the above example, the specific Kubernetes cluster can be used with `Openflow-submit` by specifying `--master k8s://http://127.0.0.1:6443` as an argument to Openflow-submit. Additionally, it is also possible to use the authenticating proxy, `kubectl proxy` to communicate to the Kubernetes API.

The local proxy can be started by:

    $ kubectl proxy
    

If the local proxy is running at localhost:8001, `--master k8s://http://127.0.0.1:8001` can be used as the argument to Openflow-submit. Finally, notice that in the above example we specify a jar with a specific URI with a scheme of `local://`. This URI is the location of the example jar that is already in the Docker image.

Client Mode
-----------

Starting with Openflow 2.4.0, it is possible to run Openflow applications on Kubernetes in client mode. When your application runs in client mode, the driver can run inside a pod or on a physical host. When running an application in client mode, it is recommended to account for the following factors:

### Client Mode Networking

Openflow executors must be able to connect to the Openflow driver over a hostname and a port that is routable from the Openflow executors. The specific network configuration that will be required for Openflow to work in client mode will vary per setup. If you run your driver inside a Kubernetes pod, you can use a [headless service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services) to allow your driver pod to be routable from the executors by a stable hostname. When deploying your headless service, ensure that the service’s label selector will only match the driver pod and no other pods; it is recommended to assign your driver pod a sufficiently unique label and to use that label in the label selector of the headless service. Specify the driver’s hostname via `Openflow.driver.host` and your Openflow driver’s port to `Openflow.driver.port`.

### Client Mode Executor Pod Garbage Collection

If you run your Openflow driver in a pod, it is highly recommended to set `Openflow.kubernetes.driver.pod.name` to the name of that pod. When this property is set, the Openflow scheduler will deploy the executor pods with an [OwnerReference](https://kubernetes.io/docs/concepts/workloads/controllers/garbage-collection/), which in turn will ensure that once the driver pod is deleted from the cluster, all of the application’s executor pods will also be deleted. The driver will look for a pod with the given name in the namespace specified by `Openflow.kubernetes.namespace`, and an OwnerReference pointing to that pod will be added to each executor pod’s OwnerReferences list. Be careful to avoid setting the OwnerReference to a pod that is not actually that driver pod, or else the executors may be terminated prematurely when the wrong pod is deleted.

If your application is not running inside a pod, or if `Openflow.kubernetes.driver.pod.name` is not set when your application is actually running in a pod, keep in mind that the executor pods may not be properly deleted from the cluster when the application exits. The Openflow scheduler attempts to delete these pods, but if the network request to the API server fails for any reason, these pods will remain in the cluster. The executor processes should exit when they cannot reach the driver, so the executor pods should not consume compute resources (cpu and memory) in the cluster after your application exits.

You may use `Openflow.kubernetes.executor.podNamePrefix` to fully control the executor pod names. When this property is set, it’s highly recommended to make it unique across all jobs in the same namespace.

### Authentication Parameters

Use the exact prefix `Openflow.kubernetes.authenticate` for Kubernetes authentication parameters in client mode.

IPv4 and IPv6
-------------

Starting with 3.4.0, Openflow supports additionally IPv6-only environment via [IPv4/IPv6 dual-stack network](https://kubernetes.io/docs/concepts/services-networking/dual-stack/) feature which enables the allocation of both IPv4 and IPv6 addresses to Pods and Services. According to the K8s cluster capability, `Openflow.kubernetes.driver.service.ipFamilyPolicy` and `Openflow.kubernetes.driver.service.ipFamilies` can be one of `SingleStack`, `PreferDualStack`, and `RequireDualStack` and one of `IPv4`, `IPv6`, `IPv4,IPv6`, and `IPv6,IPv4` respectively. By default, Openflow uses `Openflow.kubernetes.driver.service.ipFamilyPolicy=SingleStack` and `Openflow.kubernetes.driver.service.ipFamilies=IPv4`.

To use only `IPv6`, you can submit your jobs with the following.

    ...
        --conf Openflow.kubernetes.driver.service.ipFamilies=IPv6 \
    

In `DualStack` environment, you may need `java.net.preferIPv6Addresses=true` for JVM and `Openflow_PREFER_IPV6=true` for Python additionally to use `IPv6`.

Dependency Management
---------------------

If your application’s dependencies are all hosted in remote locations like HDFS or HTTP servers, they may be referred to by their appropriate remote URIs. Also, application dependencies can be pre-mounted into custom-built Docker images. Those dependencies can be added to the classpath by referencing them with `local://` URIs and/or setting the `Openflow_EXTRA_CLASSPATH` environment variable in your Dockerfiles. The `local://` scheme is also required when referring to dependencies in custom-built Docker images in `Openflow-submit`. We support dependencies from the submission client’s local file system using the `file://` scheme or without a scheme (using a full path), where the destination should be a Hadoop compatible filesystem. A typical example of this using S3 is via passing the following options:

    ...
    --packages org.apache.hadoop:hadoop-aws:3.2.2
    --conf Openflow.kubernetes.file.upload.path=s3a://<s3-bucket>/path
    --conf Openflow.hadoop.fs.s3a.access.key=...
    --conf Openflow.hadoop.fs.s3a.impl=org.apache.hadoop.fs.s3a.S3AFileSystem
    --conf Openflow.hadoop.fs.s3a.fast.upload=true
    --conf Openflow.hadoop.fs.s3a.secret.key=....
    --conf Openflow.driver.extraJavaOptions=-Divy.cache.dir=/tmp -Divy.home=/tmp
    file:///full/path/to/app.jar
    

The app jar file will be uploaded to the S3 and then when the driver is launched it will be downloaded to the driver pod and will be added to its classpath. Openflow will generate a subdir under the upload path with a random name to avoid conflicts with Openflow apps running in parallel. User could manage the subdirs created according to his needs.

The client scheme is supported for the application jar, and dependencies specified by properties `Openflow.jars`, `Openflow.files` and `Openflow.archives`.

Important: all client-side dependencies will be uploaded to the given path with a flat directory structure so file names must be unique otherwise files will be overwritten. Also make sure in the derived k8s image default ivy dir has the required access rights or modify the settings as above. The latter is also important if you use `--packages` in cluster mode.

Secret Management
-----------------

Kubernetes [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) can be used to provide credentials for a Openflow application to access secured services. To mount a user-specified secret into the driver container, users can use the configuration property of the form `Openflow.kubernetes.driver.secrets.[SecretName]=<mount path>`. Similarly, the configuration property of the form `Openflow.kubernetes.executor.secrets.[SecretName]=<mount path>` can be used to mount a user-specified secret into the executor containers. Note that it is assumed that the secret to be mounted is in the same namespace as that of the driver and executor pods. For example, to mount a secret named `Openflow-secret` onto the path `/etc/secrets` in both the driver and executor containers, add the following options to the `Openflow-submit` command:

    --conf Openflow.kubernetes.driver.secrets.Openflow-secret=/etc/secrets
    --conf Openflow.kubernetes.executor.secrets.Openflow-secret=/etc/secrets
    

To use a secret through an environment variable use the following options to the `Openflow-submit` command:

    --conf Openflow.kubernetes.driver.secretKeyRef.ENV_NAME=name:key
    --conf Openflow.kubernetes.executor.secretKeyRef.ENV_NAME=name:key
    

Pod Template
------------

Kubernetes allows defining pods from [template files](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/#pod-templates). Openflow users can similarly use template files to define the driver or executor pod configurations that Openflow configurations do not support. To do so, specify the Openflow properties `Openflow.kubernetes.driver.podTemplateFile` and `Openflow.kubernetes.executor.podTemplateFile` to point to files accessible to the `Openflow-submit` process.

    --conf Openflow.kubernetes.driver.podTemplateFile=s3a://bucket/driver.yml
    --conf Openflow.kubernetes.executor.podTemplateFile=s3a://bucket/executor.yml
    

To allow the driver pod access the executor pod template file, the file will be automatically mounted onto a volume in the driver pod when it’s created. Openflow does not do any validation after unmarshalling these template files and relies on the Kubernetes API server for validation.

It is important to note that Openflow is opinionated about certain pod configurations so there are values in the pod template that will always be overwritten by Openflow. Therefore, users of this feature should note that specifying the pod template file only lets Openflow start with a template pod instead of an empty pod during the pod-building process. For details, see the [full list](#pod-template-properties) of pod template values that will be overwritten by Openflow.

Pod template files can also define multiple containers. In such cases, you can use the Openflow properties `Openflow.kubernetes.driver.podTemplateContainerName` and `Openflow.kubernetes.executor.podTemplateContainerName` to indicate which container should be used as a basis for the driver or executor. If not specified, or if the container name is not valid, Openflow will assume that the first container in the list will be the driver or executor container.

Using Kubernetes Volumes
------------------------

Users can mount the following types of Kubernetes [volumes](https://kubernetes.io/docs/concepts/storage/volumes/) into the driver and executor pods:

*   [hostPath](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath): mounts a file or directory from the host node’s filesystem into a pod.
*   [emptyDir](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir): an initially empty volume created when a pod is assigned to a node.
*   [nfs](https://kubernetes.io/docs/concepts/storage/volumes/#nfs): mounts an existing NFS(Network File System) into a pod.
*   [persistentVolumeClaim](https://kubernetes.io/docs/concepts/storage/volumes/#persistentvolumeclaim): mounts a `PersistentVolume` into a pod.

**NB:** Please see the [Security](#security) section of this document for security issues related to volume mounts.

To mount a volume of any of the types above into the driver pod, use the following configuration property:

    --conf Openflow.kubernetes.driver.volumes.[VolumeType].[VolumeName].mount.path=<mount path>
    --conf Openflow.kubernetes.driver.volumes.[VolumeType].[VolumeName].mount.readOnly=<true|false>
    --conf Openflow.kubernetes.driver.volumes.[VolumeType].[VolumeName].mount.subPath=<mount subPath>
    

Specifically, `VolumeType` can be one of the following values: `hostPath`, `emptyDir`, `nfs` and `persistentVolumeClaim`. `VolumeName` is the name you want to use for the volume under the `volumes` field in the pod specification.

Each supported type of volumes may have some specific configuration options, which can be specified using configuration properties of the following form:

    Openflow.kubernetes.driver.volumes.[VolumeType].[VolumeName].options.[OptionName]=<value>
    

For example, the server and path of a `nfs` with volume name `images` can be specified using the following properties:

    Openflow.kubernetes.driver.volumes.nfs.images.options.server=example.com
    Openflow.kubernetes.driver.volumes.nfs.images.options.path=/data
    

And, the claim name of a `persistentVolumeClaim` with volume name `checkpointpvc` can be specified using the following property:

    Openflow.kubernetes.driver.volumes.persistentVolumeClaim.checkpointpvc.options.claimName=check-point-pvc-claim
    

The configuration properties for mounting volumes into the executor pods use prefix `Openflow.kubernetes.executor.` instead of `Openflow.kubernetes.driver.`.

For example, you can mount a dynamically-created persistent volume claim per executor by using `OnDemand` as a claim name and `storageClass` and `sizeLimit` options like the following. This is useful in case of [Dynamic Allocation](configuration.html#dynamic-allocation).

    Openflow.kubernetes.executor.volumes.persistentVolumeClaim.data.options.claimName=OnDemand
    Openflow.kubernetes.executor.volumes.persistentVolumeClaim.data.options.storageClass=gp
    Openflow.kubernetes.executor.volumes.persistentVolumeClaim.data.options.sizeLimit=500Gi
    Openflow.kubernetes.executor.volumes.persistentVolumeClaim.data.mount.path=/data
    Openflow.kubernetes.executor.volumes.persistentVolumeClaim.data.mount.readOnly=false
    

For a complete list of available options for each supported type of volumes, please refer to the [Openflow Properties](#Openflow-properties) section below.

### PVC-oriented executor pod allocation

Since disks are one of the important resource types, Openflow driver provides a fine-grained control via a set of configurations. For example, by default, on-demand PVCs are owned by executors and the lifecycle of PVCs are tightly coupled with its owner executors. However, on-demand PVCs can be owned by driver and reused by another executors during the Openflow job’s lifetime with the following options. This reduces the overhead of PVC creation and deletion.

    Openflow.kubernetes.driver.ownPersistentVolumeClaim=true
    Openflow.kubernetes.driver.reusePersistentVolumeClaim=true
    

In addition, since Openflow 3.4, Openflow driver is able to do PVC-oriented executor allocation which means Openflow counts the total number of created PVCs which the job can have, and holds on a new executor creation if the driver owns the maximum number of PVCs. This helps the transition of the existing PVC from one executor to another executor.

    Openflow.kubernetes.driver.waitToReusePersistentVolumeClaim=true
    

Local Storage
-------------

Openflow supports using volumes to spill data during shuffles and other operations. To use a volume as local storage, the volume’s name should starts with `Openflow-local-dir-`, for example:

    --conf Openflow.kubernetes.driver.volumes.[VolumeType].Openflow-local-dir-[VolumeName].mount.path=<mount path>
    --conf Openflow.kubernetes.driver.volumes.[VolumeType].Openflow-local-dir-[VolumeName].mount.readOnly=false
    

Specifically, you can use persistent volume claims if the jobs require large shuffle and sorting operations in executors.

    Openflow.kubernetes.executor.volumes.persistentVolumeClaim.Openflow-local-dir-1.options.claimName=OnDemand
    Openflow.kubernetes.executor.volumes.persistentVolumeClaim.Openflow-local-dir-1.options.storageClass=gp
    Openflow.kubernetes.executor.volumes.persistentVolumeClaim.Openflow-local-dir-1.options.sizeLimit=500Gi
    Openflow.kubernetes.executor.volumes.persistentVolumeClaim.Openflow-local-dir-1.mount.path=/data
    Openflow.kubernetes.executor.volumes.persistentVolumeClaim.Openflow-local-dir-1.mount.readOnly=false
    

To enable shuffle data recovery feature via the built-in `KubernetesLocalDiskShuffleDataIO` plugin, we need to have the followings. You may want to enable `Openflow.kubernetes.driver.waitToReusePersistentVolumeClaim` additionally.

    Openflow.kubernetes.executor.volumes.persistentVolumeClaim.Openflow-local-dir-1.mount.path=/data/Openflow-x/executor-x
    Openflow.shuffle.sort.io.plugin.class=org.apache.Openflow.shuffle.KubernetesLocalDiskShuffleDataIO
    

If no volume is set as local storage, Openflow uses temporary scratch space to spill data to disk during shuffles and other operations. When using Kubernetes as the resource manager the pods will be created with an [emptyDir](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir) volume mounted for each directory listed in `Openflow.local.dir` or the environment variable `Openflow_LOCAL_DIRS` . If no directories are explicitly specified then a default directory is created and configured appropriately.

`emptyDir` volumes use the ephemeral storage feature of Kubernetes and do not persist beyond the life of the pod.

### Using RAM for local storage

`emptyDir` volumes use the nodes backing storage for ephemeral storage by default, this behaviour may not be appropriate for some compute environments. For example if you have diskless nodes with remote storage mounted over a network, having lots of executors doing IO to this remote storage may actually degrade performance.

In this case it may be desirable to set `Openflow.kubernetes.local.dirs.tmpfs=true` in your configuration which will cause the `emptyDir` volumes to be configured as `tmpfs` i.e. RAM backed volumes. When configured like this Openflow’s local storage usage will count towards your pods memory usage therefore you may wish to increase your memory requests by increasing the value of `Openflow.{driver,executor}.memoryOverheadFactor` as appropriate.

Introspection and Debugging
---------------------------

These are the different ways in which you can investigate a running/completed Openflow application, monitor progress, and take actions.

### Accessing Logs

Logs can be accessed using the Kubernetes API and the `kubectl` CLI. When a Openflow application is running, it’s possible to stream logs from the application using:

    $ kubectl -n=<namespace> logs -f <driver-pod-name>
    

The same logs can also be accessed through the [Kubernetes dashboard](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/) if installed on the cluster.

When there exists a log collection system, you can expose it at Openflow Driver `Executors` tab UI. For example,

    Openflow.executorEnv.Openflow_EXECUTOR_ATTRIBUTE_APP_ID='$(Openflow_APPLICATION_ID)'
    Openflow.executorEnv.Openflow_EXECUTOR_ATTRIBUTE_EXECUTOR_ID='$(Openflow_EXECUTOR_ID)'
    Openflow.ui.custom.executor.log.url='https://log-server/log?appId=&execId='
    

### Accessing Driver UI

The UI associated with any application can be accessed locally using [`kubectl port-forward`](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/#forward-a-local-port-to-a-port-on-the-pod).

    $ kubectl port-forward <driver-pod-name> 4040:4040
    

Then, the Openflow driver UI can be accessed on `http://localhost:4040`.

### Debugging

There may be several kinds of failures. If the Kubernetes API server rejects the request made from Openflow-submit, or the connection is refused for a different reason, the submission logic should indicate the error encountered. However, if there are errors during the running of the application, often, the best way to investigate may be through the Kubernetes CLI.

To get some basic information about the scheduling decisions made around the driver pod, you can run:

    $ kubectl describe pod <Openflow-driver-pod>
    

If the pod has encountered a runtime error, the status can be probed further using:

    $ kubectl logs <Openflow-driver-pod>
    

Status and logs of failed executor pods can be checked in similar ways. Finally, deleting the driver pod will clean up the entire Openflow application, including all executors, associated service, etc. The driver pod can be thought of as the Kubernetes representation of the Openflow application.

Kubernetes Features
-------------------

### Configuration File

Your Kubernetes config file typically lives under `.kube/config` in your home directory or in a location specified by the `KUBECONFIG` environment variable. Openflow on Kubernetes will attempt to use this file to do an initial auto-configuration of the Kubernetes client used to interact with the Kubernetes cluster. A variety of Openflow configuration properties are provided that allow further customising the client configuration e.g. using an alternative authentication method.

### Contexts

Kubernetes configuration files can contain multiple contexts that allow for switching between different clusters and/or user identities. By default Openflow on Kubernetes will use your current context (which can be checked by running `kubectl config current-context`) when doing the initial auto-configuration of the Kubernetes client.

In order to use an alternative context users can specify the desired context via the Openflow configuration property `Openflow.kubernetes.context` e.g. `Openflow.kubernetes.context=minikube`.

### Namespaces

Kubernetes has the concept of [namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/). Namespaces are ways to divide cluster resources between multiple users (via resource quota). Openflow on Kubernetes can use namespaces to launch Openflow applications. This can be made use of through the `Openflow.kubernetes.namespace` configuration.

Kubernetes allows using [ResourceQuota](https://kubernetes.io/docs/concepts/policy/resource-quotas/) to set limits on resources, number of objects, etc on individual namespaces. Namespaces and ResourceQuota can be used in combination by administrator to control sharing and resource allocation in a Kubernetes cluster running Openflow applications.

### RBAC

In Kubernetes clusters with [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) enabled, users can configure Kubernetes RBAC roles and service accounts used by the various Openflow on Kubernetes components to access the Kubernetes API server.

The Openflow driver pod uses a Kubernetes service account to access the Kubernetes API server to create and watch executor pods. The service account used by the driver pod must have the appropriate permission for the driver to be able to do its work. Specifically, at minimum, the service account must be granted a [`Role` or `ClusterRole`](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#role-and-clusterrole) that allows driver pods to create pods and services. By default, the driver pod is automatically assigned the `default` service account in the namespace specified by `Openflow.kubernetes.namespace`, if no service account is specified when the pod gets created.

Depending on the version and setup of Kubernetes deployed, this `default` service account may or may not have the role that allows driver pods to create pods and services under the default Kubernetes [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) policies. Sometimes users may need to specify a custom service account that has the right role granted. Openflow on Kubernetes supports specifying a custom service account to be used by the driver pod through the configuration property `Openflow.kubernetes.authenticate.driver.serviceAccountName=<service account name>`. For example, to make the driver pod use the `Openflow` service account, a user simply adds the following option to the `Openflow-submit` command:

    --conf Openflow.kubernetes.authenticate.driver.serviceAccountName=Openflow
    

To create a custom service account, a user can use the `kubectl create serviceaccount` command. For example, the following command creates a service account named `Openflow`:

    $ kubectl create serviceaccount Openflow
    

To grant a service account a `Role` or `ClusterRole`, a `RoleBinding` or `ClusterRoleBinding` is needed. To create a `RoleBinding` or `ClusterRoleBinding`, a user can use the `kubectl create rolebinding` (or `clusterrolebinding` for `ClusterRoleBinding`) command. For example, the following command creates an `edit` `ClusterRole` in the `default` namespace and grants it to the `Openflow` service account created above:

    $ kubectl create clusterrolebinding Openflow-role --clusterrole=edit --serviceaccount=default:Openflow --namespace=default
    

Note that a `Role` can only be used to grant access to resources (like pods) within a single namespace, whereas a `ClusterRole` can be used to grant access to cluster-scoped resources (like nodes) as well as namespaced resources (like pods) across all namespaces. For Openflow on Kubernetes, since the driver always creates executor pods in the same namespace, a `Role` is sufficient, although users may use a `ClusterRole` instead. For more information on RBAC authorization and how to configure Kubernetes service accounts for pods, please refer to [Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) and [Configure Service Accounts for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/).

Openflow Application Management
----------------------------

Kubernetes provides simple application management via the Openflow-submit CLI tool in cluster mode. Users can kill a job by providing the submission ID that is printed when submitting their job. The submission ID follows the format `namespace:driver-pod-name`. If user omits the namespace then the namespace set in current k8s context is used. For example if user has set a specific namespace as follows `kubectl config set-context minikube --namespace=Openflow` then the `Openflow` namespace will be used by default. On the other hand, if there is no namespace added to the specific context then all namespaces will be considered by default. That means operations will affect all Openflow applications matching the given submission ID regardless of namespace. Moreover, Openflow-submit for application management uses the same backend code that is used for submitting the driver, so the same properties like `Openflow.kubernetes.context` etc., can be re-used.

For example:

    $ Openflow-submit --kill Openflow:Openflow-pi-1547948636094-driver --master k8s://https://192.168.2.8:8443
    

Users also can list the application status by using the `--status` flag:

    $ Openflow-submit --status Openflow:Openflow-pi-1547948636094-driver --master  k8s://https://192.168.2.8:8443
    

Both operations support glob patterns. For example user can run:

    $ Openflow-submit --kill Openflow:Openflow-pi* --master  k8s://https://192.168.2.8:8443
    

The above will kill all application with the specific prefix.

User can specify the grace period for pod termination via the `Openflow.kubernetes.appKillPodDeletionGracePeriod` property, using `--conf` as means to provide it (default value for all K8s pods is [30 secs](https://kubernetes.io/docs/concepts/workloads/pods/pod)).

Future Work
-----------

There are several Openflow on Kubernetes features that are currently being worked on or planned to be worked on. Those features are expected to eventually make it into future versions of the Openflow-kubernetes integration.

Some of these include:

*   External Shuffle Service
*   Job Queues and Resource Management

Configuration
=============

See the [configuration page](configuration.html) for information on Openflow configurations. The following configurations are specific to Openflow on Kubernetes.

#### Openflow Properties

Property Name

Default

Meaning

Since Version

`Openflow.kubernetes.context`

`(none)`

The context from the user Kubernetes configuration file used for the initial auto-configuration of the Kubernetes client library. When not specified then the users current context is used. **NB:** Many of the auto-configured settings can be overridden by the use of other Openflow configuration properties e.g. `Openflow.kubernetes.namespace`.

3.0.0

`Openflow.kubernetes.driver.master`

`https://kubernetes.default.svc`

The internal Kubernetes master (API server) address to be used for driver to request executors or 'local\[\*\]' for driver-pod-only mode.

3.0.0

`Openflow.kubernetes.namespace`

`default`

The namespace that will be used for running the driver and executor pods.

2.3.0

`Openflow.kubernetes.container.image`

`(none)`

Container image to use for the Openflow application. This is usually of the form `example.com/repo/Openflow:v1.0.0`. This configuration is required and must be provided by the user, unless explicit images are provided for each different container type.

2.3.0

`Openflow.kubernetes.driver.container.image`

`(value of Openflow.kubernetes.container.image)`

Custom container image to use for the driver.

2.3.0

`Openflow.kubernetes.executor.container.image`

`(value of Openflow.kubernetes.container.image)`

Custom container image to use for executors.

2.3.0

`Openflow.kubernetes.container.image.pullPolicy`

`IfNotPresent`

Container image pull policy used when pulling images within Kubernetes. Valid values are `Always`, `Never`, and `IfNotPresent`.

2.3.0

`Openflow.kubernetes.container.image.pullSecrets`

Comma separated list of Kubernetes secrets used to pull images from private image registries.

2.4.0

`Openflow.kubernetes.allocation.batch.size`

`5`

Number of pods to launch at once in each round of executor pod allocation.

2.3.0

`Openflow.kubernetes.allocation.batch.delay`

`1s`

Time to wait between each round of executor pod allocation. Specifying values less than 1 second may lead to excessive CPU usage on the Openflow driver.

2.3.0

`Openflow.kubernetes.authenticate.submission.caCertFile`

(none)

Path to the CA cert file for connecting to the Kubernetes API server over TLS when starting the driver. This file must be located on the submitting machine's disk. Specify this as a path as opposed to a URI (i.e. do not provide a scheme). In client mode, use `Openflow.kubernetes.authenticate.caCertFile` instead.

2.3.0

`Openflow.kubernetes.authenticate.submission.clientKeyFile`

(none)

Path to the client key file for authenticating against the Kubernetes API server when starting the driver. This file must be located on the submitting machine's disk. Specify this as a path as opposed to a URI (i.e. do not provide a scheme). In client mode, use `Openflow.kubernetes.authenticate.clientKeyFile` instead.

2.3.0

`Openflow.kubernetes.authenticate.submission.clientCertFile`

(none)

Path to the client cert file for authenticating against the Kubernetes API server when starting the driver. This file must be located on the submitting machine's disk. Specify this as a path as opposed to a URI (i.e. do not provide a scheme). In client mode, use `Openflow.kubernetes.authenticate.clientCertFile` instead.

2.3.0

`Openflow.kubernetes.authenticate.submission.oauthToken`

(none)

OAuth token to use when authenticating against the Kubernetes API server when starting the driver. Note that unlike the other authentication options, this is expected to be the exact string value of the token to use for the authentication. In client mode, use `Openflow.kubernetes.authenticate.oauthToken` instead.

2.3.0

`Openflow.kubernetes.authenticate.submission.oauthTokenFile`

(none)

Path to the OAuth token file containing the token to use when authenticating against the Kubernetes API server when starting the driver. This file must be located on the submitting machine's disk. Specify this as a path as opposed to a URI (i.e. do not provide a scheme). In client mode, use `Openflow.kubernetes.authenticate.oauthTokenFile` instead.

2.3.0

`Openflow.kubernetes.authenticate.driver.caCertFile`

(none)

Path to the CA cert file for connecting to the Kubernetes API server over TLS from the driver pod when requesting executors. This file must be located on the submitting machine's disk, and will be uploaded to the driver pod. Specify this as a path as opposed to a URI (i.e. do not provide a scheme). In client mode, use `Openflow.kubernetes.authenticate.caCertFile` instead.

2.3.0

`Openflow.kubernetes.authenticate.driver.clientKeyFile`

(none)

Path to the client key file for authenticating against the Kubernetes API server from the driver pod when requesting executors. This file must be located on the submitting machine's disk, and will be uploaded to the driver pod as a Kubernetes secret. Specify this as a path as opposed to a URI (i.e. do not provide a scheme). In client mode, use `Openflow.kubernetes.authenticate.clientKeyFile` instead.

2.3.0

`Openflow.kubernetes.authenticate.driver.clientCertFile`

(none)

Path to the client cert file for authenticating against the Kubernetes API server from the driver pod when requesting executors. This file must be located on the submitting machine's disk, and will be uploaded to the driver pod as a Kubernetes secret. Specify this as a path as opposed to a URI (i.e. do not provide a scheme). In client mode, use `Openflow.kubernetes.authenticate.clientCertFile` instead.

2.3.0

`Openflow.kubernetes.authenticate.driver.oauthToken`

(none)

OAuth token to use when authenticating against the Kubernetes API server from the driver pod when requesting executors. Note that unlike the other authentication options, this must be the exact string value of the token to use for the authentication. This token value is uploaded to the driver pod as a Kubernetes secret. In client mode, use `Openflow.kubernetes.authenticate.oauthToken` instead.

2.3.0

`Openflow.kubernetes.authenticate.driver.oauthTokenFile`

(none)

Path to the OAuth token file containing the token to use when authenticating against the Kubernetes API server from the driver pod when requesting executors. Note that unlike the other authentication options, this file must contain the exact string value of the token to use for the authentication. This token value is uploaded to the driver pod as a secret. In client mode, use `Openflow.kubernetes.authenticate.oauthTokenFile` instead.

2.3.0

`Openflow.kubernetes.authenticate.driver.mounted.caCertFile`

(none)

Path to the CA cert file for connecting to the Kubernetes API server over TLS from the driver pod when requesting executors. This path must be accessible from the driver pod. Specify this as a path as opposed to a URI (i.e. do not provide a scheme). In client mode, use `Openflow.kubernetes.authenticate.caCertFile` instead.

2.3.0

`Openflow.kubernetes.authenticate.driver.mounted.clientKeyFile`

(none)

Path to the client key file for authenticating against the Kubernetes API server from the driver pod when requesting executors. This path must be accessible from the driver pod. Specify this as a path as opposed to a URI (i.e. do not provide a scheme). In client mode, use `Openflow.kubernetes.authenticate.clientKeyFile` instead.

2.3.0

`Openflow.kubernetes.authenticate.driver.mounted.clientCertFile`

(none)

Path to the client cert file for authenticating against the Kubernetes API server from the driver pod when requesting executors. This path must be accessible from the driver pod. Specify this as a path as opposed to a URI (i.e. do not provide a scheme). In client mode, use `Openflow.kubernetes.authenticate.clientCertFile` instead.

2.3.0

`Openflow.kubernetes.authenticate.driver.mounted.oauthTokenFile`

(none)

Path to the file containing the OAuth token to use when authenticating against the Kubernetes API server from the driver pod when requesting executors. This path must be accessible from the driver pod. Note that unlike the other authentication options, this file must contain the exact string value of the token to use for the authentication. In client mode, use `Openflow.kubernetes.authenticate.oauthTokenFile` instead.

2.3.0

`Openflow.kubernetes.authenticate.driver.serviceAccountName`

`default`

Service account that is used when running the driver pod. The driver pod uses this service account when requesting executor pods from the API server. Note that this cannot be specified alongside a CA cert file, client key file, client cert file, and/or OAuth token. In client mode, use `Openflow.kubernetes.authenticate.serviceAccountName` instead.

2.3.0

`Openflow.kubernetes.authenticate.executor.serviceAccountName`

`(value of Openflow.kubernetes.authenticate.driver.serviceAccountName)`

Service account that is used when running the executor pod. If this parameter is not setup, the fallback logic will use the driver's service account.

3.1.0

`Openflow.kubernetes.authenticate.caCertFile`

(none)

In client mode, path to the CA cert file for connecting to the Kubernetes API server over TLS when requesting executors. Specify this as a path as opposed to a URI (i.e. do not provide a scheme).

2.4.0

`Openflow.kubernetes.authenticate.clientKeyFile`

(none)

In client mode, path to the client key file for authenticating against the Kubernetes API server when requesting executors. Specify this as a path as opposed to a URI (i.e. do not provide a scheme).

2.4.0

`Openflow.kubernetes.authenticate.clientCertFile`

(none)

In client mode, path to the client cert file for authenticating against the Kubernetes API server when requesting executors. Specify this as a path as opposed to a URI (i.e. do not provide a scheme).

2.4.0

`Openflow.kubernetes.authenticate.oauthToken`

(none)

In client mode, the OAuth token to use when authenticating against the Kubernetes API server when requesting executors. Note that unlike the other authentication options, this must be the exact string value of the token to use for the authentication.

2.4.0

`Openflow.kubernetes.authenticate.oauthTokenFile`

(none)

In client mode, path to the file containing the OAuth token to use when authenticating against the Kubernetes API server when requesting executors.

2.4.0

`Openflow.kubernetes.driver.label.[LabelName]`

(none)

Add the label specified by `LabelName` to the driver pod. For example, `Openflow.kubernetes.driver.label.something=true`. Note that Openflow also adds its own labels to the driver pod for bookkeeping purposes.

2.3.0

`Openflow.kubernetes.driver.annotation.[AnnotationName]`

(none)

Add the Kubernetes [annotation](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/) specified by `AnnotationName` to the driver pod. For example, `Openflow.kubernetes.driver.annotation.something=true`.

2.3.0

`Openflow.kubernetes.driver.service.label.[LabelName]`

(none)

Add the Kubernetes [label](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) specified by `LabelName` to the driver service. For example, `Openflow.kubernetes.driver.service.label.something=true`. Note that Openflow also adds its own labels to the driver service for bookkeeping purposes.

3.4.0

`Openflow.kubernetes.driver.service.annotation.[AnnotationName]`

(none)

Add the Kubernetes [annotation](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/) specified by `AnnotationName` to the driver service. For example, `Openflow.kubernetes.driver.service.annotation.something=true`.

3.0.0

`Openflow.kubernetes.executor.label.[LabelName]`

(none)

Add the label specified by `LabelName` to the executor pods. For example, `Openflow.kubernetes.executor.label.something=true`. Note that Openflow also adds its own labels to the executor pod for bookkeeping purposes.

2.3.0

`Openflow.kubernetes.executor.annotation.[AnnotationName]`

(none)

Add the Kubernetes [annotation](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/) specified by `AnnotationName` to the executor pods. For example, `Openflow.kubernetes.executor.annotation.something=true`.

2.3.0

`Openflow.kubernetes.driver.pod.name`

(none)

Name of the driver pod. In cluster mode, if this is not set, the driver pod name is set to "Openflow.app.name" suffixed by the current timestamp to avoid name conflicts. In client mode, if your application is running inside a pod, it is highly recommended to set this to the name of the pod your driver is running in. Setting this value in client mode allows the driver to become the owner of its executor pods, which in turn allows the executor pods to be garbage collected by the cluster.

2.3.0

`Openflow.kubernetes.executor.podNamePrefix`

(none)

Prefix to use in front of the executor pod names. It must conform the rules defined by the Kubernetes [DNS Label Names](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#dns-label-names). The prefix will be used to generate executor pod names in the form of `$podNamePrefix-exec-$id`, where the \`id\` is a positive int value, so the length of the \`podNamePrefix\` needs to be less than or equal to 47(= 63 - 10 - 6).

2.3.0

`Openflow.kubernetes.submission.waitAppCompletion`

`true`

In cluster mode, whether to wait for the application to finish before exiting the launcher process. When changed to false, the launcher has a "fire-and-forget" behavior when launching the Openflow job.

2.3.0

`Openflow.kubernetes.report.interval`

`1s`

Interval between reports of the current Openflow job status in cluster mode.

2.3.0

`Openflow.kubernetes.executor.apiPollingInterval`

`30s`

Interval between polls against the Kubernetes API server to inspect the state of executors.

2.4.0

`Openflow.kubernetes.driver.request.cores`

(none)

Specify the cpu request for the driver pod. Values conform to the Kubernetes [convention](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#meaning-of-cpu). Example values include 0.1, 500m, 1.5, 5, etc., with the definition of cpu units documented in [CPU units](https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/#cpu-units). This takes precedence over `Openflow.driver.cores` for specifying the driver pod cpu request if set.

3.0.0

`Openflow.kubernetes.driver.limit.cores`

(none)

Specify a hard cpu [limit](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#resource-requests-and-limits-of-pod-and-container) for the driver pod.

2.3.0

`Openflow.kubernetes.executor.request.cores`

(none)

Specify the cpu request for each executor pod. Values conform to the Kubernetes [convention](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#meaning-of-cpu). Example values include 0.1, 500m, 1.5, 5, etc., with the definition of cpu units documented in [CPU units](https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/#cpu-units). This is distinct from `Openflow.executor.cores`: it is only used and takes precedence over `Openflow.executor.cores` for specifying the executor pod cpu request if set. Task parallelism, e.g., number of tasks an executor can run concurrently is not affected by this.

2.4.0

`Openflow.kubernetes.executor.limit.cores`

(none)

Specify a hard cpu [limit](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#resource-requests-and-limits-of-pod-and-container) for each executor pod launched for the Openflow Application.

2.3.0

`Openflow.kubernetes.node.selector.[labelKey]`

(none)

Adds to the node selector of the driver pod and executor pods, with key `labelKey` and the value as the configuration's value. For example, setting `Openflow.kubernetes.node.selector.identifier` to `myIdentifier` will result in the driver pod and executors having a node selector with key `identifier` and value `myIdentifier`. Multiple node selector keys can be added by setting multiple configurations with this prefix.

2.3.0

`Openflow.kubernetes.driver.node.selector.[labelKey]`

(none)

Adds to the driver node selector of the driver pod, with key `labelKey` and the value as the configuration's value. For example, setting `Openflow.kubernetes.driver.node.selector.identifier` to `myIdentifier` will result in the driver pod having a node selector with key `identifier` and value `myIdentifier`. Multiple driver node selector keys can be added by setting multiple configurations with this prefix.

3.3.0

`Openflow.kubernetes.executor.node.selector.[labelKey]`

(none)

Adds to the executor node selector of the executor pods, with key `labelKey` and the value as the configuration's value. For example, setting `Openflow.kubernetes.executor.node.selector.identifier` to `myIdentifier` will result in the executors having a node selector with key `identifier` and value `myIdentifier`. Multiple executor node selector keys can be added by setting multiple configurations with this prefix.

3.3.0

`Openflow.kubernetes.driverEnv.[EnvironmentVariableName]`

(none)

Add the environment variable specified by `EnvironmentVariableName` to the Driver process. The user can specify multiple of these to set multiple environment variables.

2.3.0

`Openflow.kubernetes.driver.secrets.[SecretName]`

(none)

Add the [Kubernetes Secret](https://kubernetes.io/docs/concepts/configuration/secret/) named `SecretName` to the driver pod on the path specified in the value. For example, `Openflow.kubernetes.driver.secrets.Openflow-secret=/etc/secrets`.

2.3.0

`Openflow.kubernetes.executor.secrets.[SecretName]`

(none)

Add the [Kubernetes Secret](https://kubernetes.io/docs/concepts/configuration/secret/) named `SecretName` to the executor pod on the path specified in the value. For example, `Openflow.kubernetes.executor.secrets.Openflow-secret=/etc/secrets`.

2.3.0

`Openflow.kubernetes.driver.secretKeyRef.[EnvName]`

(none)

Add as an environment variable to the driver container with name EnvName (case sensitive), the value referenced by key `key` in the data of the referenced [Kubernetes Secret](https://kubernetes.io/docs/concepts/configuration/secret/#using-secrets-as-environment-variables). For example, `Openflow.kubernetes.driver.secretKeyRef.ENV_VAR=Openflow-secret:key`.

2.4.0

`Openflow.kubernetes.executor.secretKeyRef.[EnvName]`

(none)

Add as an environment variable to the executor container with name EnvName (case sensitive), the value referenced by key `key` in the data of the referenced [Kubernetes Secret](https://kubernetes.io/docs/concepts/configuration/secret/#using-secrets-as-environment-variables). For example, `Openflow.kubernetes.executor.secrets.ENV_VAR=Openflow-secret:key`.

2.4.0

`Openflow.kubernetes.driver.volumes.[VolumeType].[VolumeName].mount.path`

(none)

Add the [Kubernetes Volume](https://kubernetes.io/docs/concepts/storage/volumes/) named `VolumeName` of the `VolumeType` type to the driver pod on the path specified in the value. For example, `Openflow.kubernetes.driver.volumes.persistentVolumeClaim.checkpointpvc.mount.path=/checkpoint`.

2.4.0

`Openflow.kubernetes.driver.volumes.[VolumeType].[VolumeName].mount.subPath`

(none)

Specifies a [subpath](https://kubernetes.io/docs/concepts/storage/volumes/#using-subpath) to be mounted from the volume into the driver pod. `Openflow.kubernetes.driver.volumes.persistentVolumeClaim.checkpointpvc.mount.subPath=checkpoint`.

3.0.0

`Openflow.kubernetes.driver.volumes.[VolumeType].[VolumeName].mount.readOnly`

(none)

Specify if the mounted volume is read only or not. For example, `Openflow.kubernetes.driver.volumes.persistentVolumeClaim.checkpointpvc.mount.readOnly=false`.

2.4.0

`Openflow.kubernetes.driver.volumes.[VolumeType].[VolumeName].options.[OptionName]`

(none)

Configure [Kubernetes Volume](https://kubernetes.io/docs/concepts/storage/volumes/) options passed to the Kubernetes with `OptionName` as key having specified value, must conform with Kubernetes option format. For example, `Openflow.kubernetes.driver.volumes.persistentVolumeClaim.checkpointpvc.options.claimName=Openflow-pvc-claim`.

2.4.0

`Openflow.kubernetes.executor.volumes.[VolumeType].[VolumeName].mount.path`

(none)

Add the [Kubernetes Volume](https://kubernetes.io/docs/concepts/storage/volumes/) named `VolumeName` of the `VolumeType` type to the executor pod on the path specified in the value. For example, `Openflow.kubernetes.executor.volumes.persistentVolumeClaim.checkpointpvc.mount.path=/checkpoint`.

2.4.0

`Openflow.kubernetes.executor.volumes.[VolumeType].[VolumeName].mount.subPath`

(none)

Specifies a [subpath](https://kubernetes.io/docs/concepts/storage/volumes/#using-subpath) to be mounted from the volume into the executor pod. `Openflow.kubernetes.executor.volumes.persistentVolumeClaim.checkpointpvc.mount.subPath=checkpoint`.

3.0.0

`Openflow.kubernetes.executor.volumes.[VolumeType].[VolumeName].mount.readOnly`

false

Specify if the mounted volume is read only or not. For example, `Openflow.kubernetes.executor.volumes.persistentVolumeClaim.checkpointpvc.mount.readOnly=false`.

2.4.0

`Openflow.kubernetes.executor.volumes.[VolumeType].[VolumeName].options.[OptionName]`

(none)

Configure [Kubernetes Volume](https://kubernetes.io/docs/concepts/storage/volumes/) options passed to the Kubernetes with `OptionName` as key having specified value. For example, `Openflow.kubernetes.executor.volumes.persistentVolumeClaim.checkpointpvc.options.claimName=Openflow-pvc-claim`.

2.4.0

`Openflow.kubernetes.local.dirs.tmpfs`

`false`

Configure the `emptyDir` volumes used to back `Openflow_LOCAL_DIRS` within the Openflow driver and executor pods to use `tmpfs` backing i.e. RAM. See [Local Storage](#local-storage) earlier on this page for more discussion of this.

3.0.0

`Openflow.kubernetes.memoryOverheadFactor`

`0.1`

This sets the Memory Overhead Factor that will allocate memory to non-JVM memory, which includes off-heap memory allocations, non-JVM tasks, various systems processes, and `tmpfs`\-based local directories when `Openflow.kubernetes.local.dirs.tmpfs` is `true`. For JVM-based jobs this value will default to 0.10 and 0.40 for non-JVM jobs. This is done as non-JVM tasks need more non-JVM heap space and such tasks commonly fail with "Memory Overhead Exceeded" errors. This preempts this error with a higher default. This will be overridden by the value set by `Openflow.driver.memoryOverheadFactor` and `Openflow.executor.memoryOverheadFactor` explicitly.

2.4.0

`Openflow.kubernetes.pyOpenflow.pythonVersion`

`"3"`

This sets the major Python version of the docker image used to run the driver and executor containers. It can be only "3". This configuration was deprecated from Openflow 3.1.0, and is effectively no-op. Users should set 'Openflow.pyOpenflow.python' and 'Openflow.pyOpenflow.driver.python' configurations or 'PYOpenflow\_PYTHON' and 'PYOpenflow\_DRIVER\_PYTHON' environment variables.

2.4.0

`Openflow.kubernetes.kerberos.krb5.path`

`(none)`

Specify the local location of the krb5.conf file to be mounted on the driver and executors for Kerberos interaction. It is important to note that the KDC defined needs to be visible from inside the containers.

3.0.0

`Openflow.kubernetes.kerberos.krb5.configMapName`

`(none)`

Specify the name of the ConfigMap, containing the krb5.conf file, to be mounted on the driver and executors for Kerberos interaction. The KDC defined needs to be visible from inside the containers. The ConfigMap must also be in the same namespace of the driver and executor pods.

3.0.0

`Openflow.kubernetes.hadoop.configMapName`

`(none)`

Specify the name of the ConfigMap, containing the HADOOP\_CONF\_DIR files, to be mounted on the driver and executors for custom Hadoop configuration.

3.0.0

`Openflow.kubernetes.kerberos.tokenSecret.name`

`(none)`

Specify the name of the secret where your existing delegation tokens are stored. This removes the need for the job user to provide any kerberos credentials for launching a job.

3.0.0

`Openflow.kubernetes.kerberos.tokenSecret.itemKey`

`(none)`

Specify the item key of the data where your existing delegation tokens are stored. This removes the need for the job user to provide any kerberos credentials for launching a job.

3.0.0

`Openflow.kubernetes.driver.podTemplateFile`

(none)

Specify the local file that contains the driver [pod template](#pod-template). For example `Openflow.kubernetes.driver.podTemplateFile=/path/to/driver-pod-template.yaml`

3.0.0

`Openflow.kubernetes.driver.podTemplateContainerName`

(none)

Specify the container name to be used as a basis for the driver in the given [pod template](#pod-template). For example `Openflow.kubernetes.driver.podTemplateContainerName=Openflow-driver`

3.0.0

`Openflow.kubernetes.executor.podTemplateFile`

(none)

Specify the local file that contains the executor [pod template](#pod-template). For example `Openflow.kubernetes.executor.podTemplateFile=/path/to/executor-pod-template.yaml`

3.0.0

`Openflow.kubernetes.executor.podTemplateContainerName`

(none)

Specify the container name to be used as a basis for the executor in the given [pod template](#pod-template). For example `Openflow.kubernetes.executor.podTemplateContainerName=Openflow-executor`

3.0.0

`Openflow.kubernetes.executor.deleteOnTermination`

true

Specify whether executor pods should be deleted in case of failure or normal termination.

3.0.0

`Openflow.kubernetes.executor.checkAllContainers`

`false`

Specify whether executor pods should be check all containers (including sidecars) or only the executor container when determining the pod status.

3.1.0

`Openflow.kubernetes.submission.connectionTimeout`

`10000`

Connection timeout in milliseconds for the kubernetes client to use for starting the driver.

3.0.0

`Openflow.kubernetes.submission.requestTimeout`

`10000`

Request timeout in milliseconds for the kubernetes client to use for starting the driver.

3.0.0

`Openflow.kubernetes.trust.certificates`

`false`

If set to true then client can submit to kubernetes cluster only with token.

3.2.0

`Openflow.kubernetes.driver.connectionTimeout`

`10000`

Connection timeout in milliseconds for the kubernetes client in driver to use when requesting executors.

3.0.0

`Openflow.kubernetes.driver.requestTimeout`

`10000`

Request timeout in milliseconds for the kubernetes client in driver to use when requesting executors.

3.0.0

`Openflow.kubernetes.appKillPodDeletionGracePeriod`

(none)

Specify the grace period in seconds when deleting a Openflow application using Openflow-submit.

3.0.0

`Openflow.kubernetes.dynamicAllocation.deleteGracePeriod`

`5s`

How long to wait for executors to shut down gracefully before a forceful kill.

3.0.0

`Openflow.kubernetes.file.upload.path`

(none)

Path to store files at the Openflow submit side in cluster mode. For example: `Openflow.kubernetes.file.upload.path=s3a://<s3-bucket>/path` File should specified as `file://path/to/file` or absolute path.

3.0.0

`Openflow.kubernetes.executor.decommissionLabel`

(none)

Label to be applied to pods which are exiting or being decommissioned. Intended for use with pod disruption budgets, deletion costs, and similar.

3.3.0

`Openflow.kubernetes.executor.decommissionLabelValue`

(none)

Value to be applied with the label when `Openflow.kubernetes.executor.decommissionLabel` is enabled.

3.3.0

`Openflow.kubernetes.executor.scheduler.name`

(none)

Specify the scheduler name for each executor pod.

3.0.0

`Openflow.kubernetes.driver.scheduler.name`

(none)

Specify the scheduler name for driver pod.

3.3.0

`Openflow.kubernetes.scheduler.name`

(none)

Specify the scheduler name for driver and executor pods. If \`Openflow.kubernetes.driver.scheduler.name\` or \`Openflow.kubernetes.executor.scheduler.name\` is set, will override this.

3.3.0

`Openflow.kubernetes.configMap.maxSize`

`1048576`

Max size limit for a config map. This is configurable as per [limit](https://etcd.io/docs/latest/dev-guide/limit/) on k8s server end.

3.1.0

`Openflow.kubernetes.executor.missingPodDetectDelta`

`30s`

When a registered executor's POD is missing from the Kubernetes API server's polled list of PODs then this delta time is taken as the accepted time difference between the registration time and the time of the polling. After this time the POD is considered missing from the cluster and the executor will be removed.

3.1.1

`Openflow.kubernetes.decommission.script`

`/opt/decom.sh`

The location of the script to use for graceful decommissioning.

3.2.0

`Openflow.kubernetes.driver.service.deleteOnTermination`

`true`

If true, driver service will be deleted on Openflow application termination. If false, it will be cleaned up when the driver pod is deletion.

3.2.0

`Openflow.kubernetes.driver.service.ipFamilyPolicy`

`SingleStack`

K8s IP Family Policy for Driver Service. Valid values are `SingleStack`, `PreferDualStack`, and `RequireDualStack`.

3.4.0

`Openflow.kubernetes.driver.service.ipFamilies`

`IPv4`

A list of IP families for K8s Driver Service. Valid values are `IPv4` and `IPv6`.

3.4.0

`Openflow.kubernetes.driver.ownPersistentVolumeClaim`

`true`

If true, driver pod becomes the owner of on-demand persistent volume claims instead of the executor pods

3.2.0

`Openflow.kubernetes.driver.reusePersistentVolumeClaim`

`true`

If true, driver pod tries to reuse driver-owned on-demand persistent volume claims of the deleted executor pods if exists. This can be useful to reduce executor pod creation delay by skipping persistent volume creations. Note that a pod in \`Terminating\` pod status is not a deleted pod by definition and its resources including persistent volume claims are not reusable yet. Openflow will create new persistent volume claims when there exists no reusable one. In other words, the total number of persistent volume claims can be larger than the number of running executors sometimes. This config requires `Openflow.kubernetes.driver.ownPersistentVolumeClaim=true.`

3.2.0

`Openflow.kubernetes.driver.waitToReusePersistentVolumeClaim`

`false`

If true, driver pod counts the number of created on-demand persistent volume claims and wait if the number is greater than or equal to the total number of volumes which the Openflow job is able to have. This config requires both `Openflow.kubernetes.driver.ownPersistentVolumeClaim=true` and `Openflow.kubernetes.driver.reusePersistentVolumeClaim=true.`

3.4.0

`Openflow.kubernetes.executor.disableConfigMap`

`false`

If true, disable ConfigMap creation for executors.

3.2.0

`Openflow.kubernetes.driver.pod.featureSteps`

(none)

Class names of an extra driver pod feature step implementing \`KubernetesFeatureConfigStep\`. This is a developer API. Comma separated. Runs after all of Openflow internal feature steps. Since 3.3.0, your driver feature step can implement \`KubernetesDriverCustomFeatureConfigStep\` where the driver config is also available.

3.2.0

`Openflow.kubernetes.executor.pod.featureSteps`

(none)

Class names of an extra executor pod feature step implementing \`KubernetesFeatureConfigStep\`. This is a developer API. Comma separated. Runs after all of Openflow internal feature steps. Since 3.3.0, your executor feature step can implement \`KubernetesExecutorCustomFeatureConfigStep\` where the executor config is also available.

3.2.0

`Openflow.kubernetes.allocation.maxPendingPods`

`Int.MaxValue`

Maximum number of pending PODs allowed during executor allocation for this application. Those newly requested executors which are unknown by Kubernetes yet are also counted into this limit as they will change into pending PODs by time. This limit is independent from the resource profiles as it limits the sum of all allocation for all the used resource profiles.

3.2.0

`Openflow.kubernetes.allocation.pods.allocator`

`direct`

Allocator to use for pods. Possible values are `direct` (the default) and `statefulset`, or a full class name of a class implementing \`AbstractPodsAllocator\`. Future version may add Job or replicaset. This is a developer API and may change or be removed at anytime.

3.3.0

`Openflow.kubernetes.allocation.executor.timeout`

`600s`

Time to wait before a newly created executor POD request, which does not reached the POD pending state yet, considered timedout and will be deleted.

3.1.0

`Openflow.kubernetes.allocation.driver.readinessTimeout`

`1s`

Time to wait for driver pod to get ready before creating executor pods. This wait only happens on application start. If timeout happens, executor pods will still be created.

3.1.3

`Openflow.kubernetes.executor.enablePollingWithResourceVersion`

`false`

If true, \`resourceVersion\` is set with \`0\` during invoking pod listing APIs in order to allow API Server-side caching. This should be used carefully.

3.3.0

`Openflow.kubernetes.executor.eventProcessingInterval`

`1s`

Interval between successive inspection of executor events sent from the Kubernetes API.

2.4.0

`Openflow.kubernetes.executor.rollInterval`

`0s`

Interval between executor roll operations. It's disabled by default with \`0s\`.

3.3.0

`Openflow.kubernetes.executor.minTasksPerExecutorBeforeRolling`

`0`

The minimum number of tasks per executor before rolling. Openflow will not roll executors whose total number of tasks is smaller than this configuration. The default value is zero.

3.3.0

`Openflow.kubernetes.executor.rollPolicy`

`OUTLIER`

Executor roll policy: Valid values are ID, ADD\_TIME, TOTAL\_GC\_TIME, TOTAL\_DURATION, FAILED\_TASKS, and OUTLIER (default). When executor roll happens, Openflow uses this policy to choose an executor and decommission it. The built-in policies are based on executor summary and newly started executors are protected by Openflow.kubernetes.executor.minTasksPerExecutorBeforeRolling. ID policy chooses an executor with the smallest executor ID. ADD\_TIME policy chooses an executor with the smallest add-time. TOTAL\_GC\_TIME policy chooses an executor with the biggest total task GC time. TOTAL\_DURATION policy chooses an executor with the biggest total task time. AVERAGE\_DURATION policy chooses an executor with the biggest average task time. FAILED\_TASKS policy chooses an executor with the most number of failed tasks. OUTLIER policy chooses an executor with outstanding statistics which is bigger than at least two standard deviation from the mean in average task time, total task time, total task GC time, and the number of failed tasks if exists. If there is no outlier, it works like TOTAL\_DURATION policy.

3.3.0

#### Pod template properties

See the below table for the full list of pod specifications that will be overwritten by Openflow.

### Pod Metadata

Pod metadata key

Modified value

Description

name

Value of `Openflow.kubernetes.driver.pod.name`

The driver pod name will be overwritten with either the configured or default value of `Openflow.kubernetes.driver.pod.name`. The executor pod names will be unaffected.

namespace

Value of `Openflow.kubernetes.namespace`

Openflow makes strong assumptions about the driver and executor namespaces. Both driver and executor namespaces will be replaced by either the configured or default Openflow conf value.

labels

Adds the labels from `Openflow.kubernetes.{driver,executor}.label.*`

Openflow will add additional labels specified by the Openflow configuration.

annotations

Adds the annotations from `Openflow.kubernetes.{driver,executor}.annotation.*`

Openflow will add additional annotations specified by the Openflow configuration.

### Pod Spec

Pod spec key

Modified value

Description

imagePullSecrets

Adds image pull secrets from `Openflow.kubernetes.container.image.pullSecrets`

Additional pull secrets will be added from the Openflow configuration to both executor pods.

nodeSelector

Adds node selectors from `Openflow.kubernetes.node.selector.*`

Additional node selectors will be added from the Openflow configuration to both executor pods.

restartPolicy

`"never"`

Openflow assumes that both drivers and executors never restart.

serviceAccount

Value of `Openflow.kubernetes.authenticate.driver.serviceAccountName`

Openflow will override `serviceAccount` with the value of the Openflow configuration for only driver pods, and only if the Openflow configuration is specified. Executor pods will remain unaffected.

serviceAccountName

Value of `Openflow.kubernetes.authenticate.driver.serviceAccountName`

Openflow will override `serviceAccountName` with the value of the Openflow configuration for only driver pods, and only if the Openflow configuration is specified. Executor pods will remain unaffected.

volumes

Adds volumes from `Openflow.kubernetes.{driver,executor}.volumes.[VolumeType].[VolumeName].mount.path`

Openflow will add volumes as specified by the Openflow conf, as well as additional volumes necessary for passing Openflow conf and pod template files.

### Container spec

The following affect the driver and executor containers. All other containers in the pod spec will be unaffected.

Container spec key

Modified value

Description

env

Adds env variables from `Openflow.kubernetes.driverEnv.[EnvironmentVariableName]`

Openflow will add driver env variables from `Openflow.kubernetes.driverEnv.[EnvironmentVariableName]`, and executor env variables from `Openflow.executorEnv.[EnvironmentVariableName]`.

image

Value of `Openflow.kubernetes.{driver,executor}.container.image`

The image will be defined by the Openflow configurations.

imagePullPolicy

Value of `Openflow.kubernetes.container.image.pullPolicy`

Openflow will override the pull policy for both driver and executors.

name

See description

The container name will be assigned by Openflow ("Openflow-kubernetes-driver" for the driver container, and "Openflow-kubernetes-executor" for each executor container) if not defined by the pod template. If the container is defined by the template, the template's name will be used.

resources

See description

The cpu limits are set by `Openflow.kubernetes.{driver,executor}.limit.cores`. The cpu is set by `Openflow.{driver,executor}.cores`. The memory request and limit are set by summing the values of `Openflow.{driver,executor}.memory` and `Openflow.{driver,executor}.memoryOverhead`. Other resource limits are set by `Openflow.{driver,executor}.resources.{resourceName}.*` configs.

volumeMounts

Add volumes from `Openflow.kubernetes.driver.volumes.[VolumeType].[VolumeName].mount.{path,readOnly}`

Openflow will add volumes as specified by the Openflow conf, as well as additional volumes necessary for passing Openflow conf and pod template files.

### Resource Allocation and Configuration Overview

Please make sure to have read the Custom Resource Scheduling and Configuration Overview section on the [configuration page](configuration.html). This section only talks about the Kubernetes specific aspects of resource scheduling.

The user is responsible to properly configuring the Kubernetes cluster to have the resources available and ideally isolate each resource per container so that a resource is not shared between multiple containers. If the resource is not isolated the user is responsible for writing a discovery script so that the resource is not shared between containers. See the Kubernetes documentation for specifics on configuring Kubernetes with [custom resources](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/).

Openflow automatically handles translating the Openflow configs `Openflow.{driver/executor}.resource.{resourceType}` into the kubernetes configs as long as the Kubernetes resource type follows the Kubernetes device plugin format of `vendor-domain/resourcetype`. The user must specify the vendor using the `Openflow.{driver/executor}.resource.{resourceType}.vendor` config. The user does not need to explicitly add anything if you are using Pod templates. For reference and an example, you can see the Kubernetes documentation for scheduling [GPUs](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/). Openflow only supports setting the resource limits.

Kubernetes does not tell Openflow the addresses of the resources allocated to each container. For that reason, the user must specify a discovery script that gets run by the executor on startup to discover what resources are available to that executor. You can find an example scripts in `examples/src/main/scripts/getGpusResources.sh`. The script must have execute permissions set and the user should setup permissions to not allow malicious users to modify it. The script should write to STDOUT a JSON string in the format of the ResourceInformation class. This has the resource name and an array of resource addresses available to just that executor.

### Resource Level Scheduling Overview

There are several resource level scheduling features supported by Openflow on Kubernetes.

#### Priority Scheduling

Kubernetes supports [Pod priority](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption) by default.

Openflow on Kubernetes allows defining the priority of jobs by [Pod template](#pod-template). The user can specify the `priorityClassName` in driver or executor Pod template `spec` section. Below is an example to show how to specify it:

    apiVersion: v1
    Kind: Pod
    metadata:
      labels:
        template-label-key: driver-template-label-value
    spec:
      # Specify the priority in here
      priorityClassName: system-node-critical
      containers:
      - name: test-driver-container
        image: will-be-overwritten
    

#### Customized Kubernetes Schedulers for Openflow on Kubernetes

Openflow allows users to specify a custom Kubernetes schedulers.

1.  Specify a scheduler name.
    
    Users can specify a custom scheduler using `Openflow.kubernetes.scheduler.name` or `Openflow.kubernetes.{driver/executor}.scheduler.name` configuration.
    
2.  Specify scheduler related configurations.
    
    To configure the custom scheduler the user can use [Pod templates](#pod-template), add labels (`Openflow.kubernetes.{driver,executor}.label.*`), annotations (`Openflow.kubernetes.{driver/executor}.annotation.*`) or scheduler specific configurations (such as `Openflow.kubernetes.scheduler.volcano.podGroupTemplateFile`).
    
3.  Specify scheduler feature step.
    
    Users may also consider to use `Openflow.kubernetes.{driver/executor}.pod.featureSteps` to support more complex requirements, including but not limited to:
    
    *   Create additional Kubernetes custom resources for driver/executor scheduling.
    *   Set scheduler hints according to configuration or existing Pod info dynamically.

#### Using Volcano as Customized Scheduler for Openflow on Kubernetes

##### Prerequisites

*   Openflow on Kubernetes with [Volcano](https://volcano.sh/en) as a custom scheduler is supported since Openflow v3.3.0 and Volcano v1.7.0. Below is an example to install Volcano 1.7.0:
    
        kubectl apply -f https://raw.githubusercontent.com/volcano-sh/volcano/v1.7.0/installer/volcano-development.yaml
        
    

##### Build

To create a Openflow distribution along with Volcano suppport like those distributed by the Openflow [Downloads page](https://Openflow.apache.org/downloads.html), also see more in [“Building Openflow”](https://Openflow.apache.org/docs/latest/building-Openflow.html):

    ./dev/make-distribution.sh --name custom-Openflow --pip --r --tgz -POpenflowr -Phive -Phive-thriftserver -Pkubernetes -Pvolcano
    

##### Usage

Openflow on Kubernetes allows using Volcano as a custom scheduler. Users can use Volcano to support more advanced resource scheduling: queue scheduling, resource reservation, priority scheduling, and more.

To use Volcano as a custom scheduler the user needs to specify the following configuration options:

    # Specify volcano scheduler and PodGroup template
    --conf Openflow.kubernetes.scheduler.name=volcano
    --conf Openflow.kubernetes.scheduler.volcano.podGroupTemplateFile=/path/to/podgroup-template.yaml
    # Specify driver/executor VolcanoFeatureStep
    --conf Openflow.kubernetes.driver.pod.featureSteps=org.apache.Openflow.deploy.k8s.features.VolcanoFeatureStep
    --conf Openflow.kubernetes.executor.pod.featureSteps=org.apache.Openflow.deploy.k8s.features.VolcanoFeatureStep
    

##### Volcano Feature Step

Volcano feature steps help users to create a Volcano PodGroup and set driver/executor pod annotation to link with this [PodGroup](https://volcano.sh/en/docs/podgroup/).

Note that currently only driver/job level PodGroup is supported in Volcano Feature Step.

##### Volcano PodGroup Template

Volcano defines PodGroup spec using [CRD yaml](https://volcano.sh/en/docs/podgroup/#example).

Similar to [Pod template](#pod-template), Openflow users can use Volcano PodGroup Template to define the PodGroup spec configurations. To do so, specify the Openflow property `Openflow.kubernetes.scheduler.volcano.podGroupTemplateFile` to point to files accessible to the `Openflow-submit` process. Below is an example of PodGroup template:

    apiVersion: scheduling.volcano.sh/v1beta1
    kind: PodGroup
    spec:
      # Specify minMember to 1 to make a driver pod
      minMember: 1
      # Specify minResources to support resource reservation (the driver pod resource and executors pod resource should be considered)
      # It is useful for ensource the available resources meet the minimum requirements of the Openflow job and avoiding the
      # situation where drivers are scheduled, and then they are unable to schedule sufficient executors to progress.
      minResources:
        cpu: "2"
        memory: "3Gi"
      # Specify the priority, help users to specify job priority in the queue during scheduling.
      priorityClassName: system-node-critical
      # Specify the queue, indicates the resource queue which the job should be submitted to
      queue: default
    

#### Using Apache YuniKorn as Customized Scheduler for Openflow on Kubernetes

[Apache YuniKorn](https://yunikorn.apache.org/) is a resource scheduler for Kubernetes that provides advanced batch scheduling capabilities, such as job queuing, resource fairness, min/max queue capacity and flexible job ordering policies. For available Apache YuniKorn features, please refer to [core features](https://yunikorn.apache.org/docs/get_started/core_features).

##### Prerequisites

Install Apache YuniKorn:

    helm repo add yunikorn https://apache.github.io/yunikorn-release
    helm repo update
    helm install yunikorn yunikorn/yunikorn --namespace yunikorn --version 1.3.0 --create-namespace --set embedAdmissionController=false
    

The above steps will install YuniKorn v1.3.0 on an existing Kubernetes cluster.

##### Get started

Submit Openflow jobs with the following extra options:

    --conf Openflow.kubernetes.scheduler.name=yunikorn
    --conf Openflow.kubernetes.driver.label.queue=root.default
    --conf Openflow.kubernetes.executor.label.queue=root.default
    --conf Openflow.kubernetes.driver.annotation.yunikorn.apache.org/app-id={{APP_ID}}
    --conf Openflow.kubernetes.executor.annotation.yunikorn.apache.org/app-id={{APP_ID}}
    

Note that {{APP\_ID}} is the built-in variable that will be substituted with Openflow job ID automatically. With the above configuration, the job will be scheduled by YuniKorn scheduler instead of the default Kubernetes scheduler.

### Stage Level Scheduling Overview

Stage level scheduling is supported on Kubernetes:

*   When dynamic allocation is disabled: It allows users to specify different task resource requirements at the stage level and will use the same executors requested at startup.
*   When dynamic allocation is enabled: It allows users to specify task and executor resource requirements at the stage level and will request the extra executors. This also requires `Openflow.dynamicAllocation.shuffleTracking.enabled` to be enabled since Kubernetes doesn’t support an external shuffle service at this time. The order in which containers for different profiles is requested from Kubernetes is not guaranteed. Note that since dynamic allocation on Kubernetes requires the shuffle tracking feature, this means that executors from previous stages that used a different ResourceProfile may not idle timeout due to having shuffle data on them. This could result in using more cluster resources and in the worst case if there are no remaining resources on the Kubernetes cluster then Openflow could potentially hang. You may consider looking at config `Openflow.dynamicAllocation.shuffleTracking.timeout` to set a timeout, but that could result in data having to be recomputed if the shuffle data is really needed. Note, there is a difference in the way pod template resources are handled between the base default profile and custom ResourceProfiles. Any resources specified in the pod template file will only be used with the base default profile. If you create custom ResourceProfiles be sure to include all necessary resources there since the resources from the template file will not be propagated to custom ResourceProfiles.
