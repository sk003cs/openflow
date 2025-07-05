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
