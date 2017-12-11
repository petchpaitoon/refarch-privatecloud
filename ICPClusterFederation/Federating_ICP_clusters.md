Federating ICP v2.1 Clusters
=======================================================
# Introduction

This is a guide for federating two or more ICP v2.1 clusters.

TBD - Some of the documentation sources on Kubernetes cluster federation makes a distinction that the clusters to be federated are on-premises as opposed to deployed in a public cloud.  It would be good to document why that matters.

The information in this guide is based on federating on-premises ICP clusters.  ICP is running Kubernetes.  The federation of ICP clusters is essentially the federation of Kubernetes clusters.

This guide is based on using two ICP v2.1 clusters.  For purposes of this document, one cluster is referred to as FC01 and the other cluster is referred to as FC02.

The federation work is done on the bootmaster VM of each ICP cluster:
```
fc01-bootmaster01
fc02-bootmaster01
```

The ICP home directory on the bootmaster VM is referred to as `ICP_HOME` in this document.  (The actual directory may appear in some code samples, and it is: `/opt/icp`.)

# Getting started
This section describes some preliminary steps required to get prepared to do the federation.

## Install kubectl

You need kubectl for much of the configuration work.  On a machine where ICP has been installed, you can simply copy the kubectl executable from the Kubernetes container.

```
docker run -e LICENSE=accept --net=host -v /usr/local/bin:/data ibmcom/kubernetes:v1.7.3-ee cp /kubectl /data
```

The above command copies the kubectl executable to /usr/local/bin.

You can run something simple from the command line like `kubectl version`, to confirm that kubectl is available in your shell.
```
# kubectl version
Client Version: version.Info{Major:"1", Minor:"7+", GitVersion:"v1.7.3-11+f747daa02c9ffb", GitCommit:"f747daa02c9ffba3fabc9f679b5003b3f7fcb2c0", GitTreeState:"clean", BuildDate:"2017-08-28T07:04:45Z", GoVersion:"go1.8.3", Compiler:"gc", Platform:"linux/amd64"}
Error from server (NotFound): the server could not find the requested resource
```
The error message at the end of the version information is because no _context_ has been established.  The section below describes the steps for configuring a Kubernetes context for using kubectl.

## Create a kubernetes context for interacting with the current cluster

(TBD - I need to confirm this definition of a context.) Kubernetes uses the term "context" to mean the collection of configuration information that establishes which Kubernetes cluster is to be referenced by kubectl commands. The context information may be defined in what is referred to as a _kubeconfig_ file. The context information includes a connection URL and an authentication token among numerous other things.

To set up a Kubernetes context for use with the cluster in ICP do the following:
- Login to the ICP console.
- Click on your user name in the upper right corner of the console window.  (See figure below.)
![Configure Kubectl Client from ICP Console](images/01_ConfigureKubectlClientFromICPConsole.png)
- A pop-up window should appear with the kubectl commands listed that are needed to configure a kubectl context for your ICP cluster.  (See figure below.)
![Getting Kubeconfig from ICP Console](images/02_GettingKubeconfigFromICPConsole.png)
You can copy the command block into the clipboard by clicking in the icon in the upper right corner of the kubeconfig pop-up window that looks like two overlapping sheets of paper.  (See figure above.)

- Paste the the kubectl commands into a shell and they will execute.  At that point you will have a context established for the Kubernetes cluster for the ICP instance where you were logged in.

You can view the defined contexts using the `kubectl config get-contexts` command.
```
# kubectl config get-contexts
[root@fc02-bootmaster01 federation]# kubectl config get-contexts
CURRENT   NAME                    CLUSTER         AUTHINFO             NAMESPACE
*         mycluster.icp-context   mycluster.icp   mycluster.icp-user   default
```
Once a context has been established for the kubectl client, you an run the `kubectl version` command and you should not see an error message.

You can view the details of the current configuration with the command: `kubectl config view`

## Create a work area directory in the file system

In order to keep all of the federation related artifacts organized, create a work area directory named `federation` in `ICP_HOME`.
```
# cd /opt/icp
# mkdir federation
# cd federation
```
The steps in the process that create a file system artifact are executed in `ICP_HOME/federation`.

## Save kubeconfig for the context of each ICP cluster to a file

On the boot-master host for each cluster (where you have a shell with the cluster context defined) run the `kubectl config view` command with output directed to a file.

```
kubectl config view > fc01-kubeconfig.yml
```
Then on the other boot-master host:
```
kubectl config view > fc01-kubeconfig.yml
```

## Merge the kubeconfig files from all clusters

- Copy all of the kubeconfig files to the federation host into its `ICP_HOME/federation` directory.

- Edit each kubeconfig file and do a global replace of mycluster with something that uniquely identifies the cluster.  In this example we replaced mycluster with fc01 for the kubeconfig YAML file from cluster 01, and mycluster with fc02 for the kubeconfig YAML file from cluster 02.

- Merge the kubeconfig files from each cluster by appending them into a single file.  In this example the merged kubeonfig file is named `fc-kubeconfig.yml`.

- In the merged kubeconfig file (`fc-kubeconfig.yml`) combine the `clusters`, `contexts` and `users` sections.  Remove redundancies for attributes such as `apiVersion`, `kind`, `preferences`.

The resulting fc-kubeconfig.yml file looks like:
```
apiVersion: v1
clusters:
- cluster:
    insecure-skip-tls-verify: true
    server: https://10.11.12.1:8001
  name: fc01.icp
- cluster:
    insecure-skip-tls-verify: true
    server: https://10.11.12.1:8001
  name: fc02.icp
contexts:
- context:
    cluster: fc01.icp
    namespace: kube-system
    user: fc01.icp-user
  name: fc01.icp-context
- context:
    cluster: fc02.icp
    namespace: default
    user: fc02.icp-user
  name: fc02.icp-context
current-context: fc01.icp-context
users:
- name: fc01.icp-user
  user:
    token: REDACTED
- name: fc02.icp-user
  user:
    token: REDACTED
kind: Config
preferences: {}
```

- Set the KUBECONFIG environment variable to the combined kubeconfig file, e.g., `fc-kubeconfig.yml`.
```
[root@fc01-bootmaster01 federation]# export KUBECONFIG=./fc-kubeconfig.yml
[root@fc01-bootmaster01 federation]# kubectl config get-contexts
CURRENT   NAME               CLUSTER    AUTHINFO        NAMESPACE
*         fc01.icp-context   fc01.icp   fc01.icp-user   kube-system
          fc02.icp-context   fc02.icp   fc02.icp-user   default

```

## Label the nodes



# References
- [Kubernetes Federation for on-premises clusters](https://github.com/ufcg-lsd/k8s-onpremise-federation) This document may prove to be useful. Jeff Kwong referenced it and I found it helpful as background.
- Kubernetes documentation: [Configure Access to Multiple Clusters](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)