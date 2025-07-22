# Introduction

The easiest way to get started with "full blown" OpenShift (OCP) is using the Assisted installer to deploy Single-node OpenShift (SNO). This guide will walk you through the basic procedure. It's written using HPE bare metal as an example, but could be replicated on other bare metal or virtual deployments. Works fine on QEMU/KVM.

For customers and partners interested in customizing RHEL CoreOS (RHCOS) there is also a walkthrough of setting up on-cluster image mode (also known as on-cluster layering) to deploy some HPE binaries. The same basic procedure would apply for adding additional RHEL content (those repos are available automatically) or non-RHEL content.

## Version Compatibility

**This guide is written for OpenShift 4.19.** On-cluster image mode is GA (Generally Available) in OpenShift 4.19 and will be GA in an upcoming OpenShift 4.18.z release. For earlier versions (4.17 and earlier 4.18.x releases), additional feature gate configuration is required.

# Does On-cluster Layering have anything to do with `bootc` and Image mode?

Absolutely. `bootc` was originally an enhancement to rpm-ostree called [OSTree Native containers](https://coreos.github.io/rpm-ostree/container/). Today, they have essentially the same functionality, i.e., the ability to customize operating system images with a familiar container build process. Eventually OpenShift will converge on bootc as the underlying technology but operationally they act the same already. Migration should be a non-event for users.

# LINKS

* [https://console.redhat.com/openshift/assisted-installer/clusters](https://console.redhat.com/openshift/assisted-installer/clusters)
* [Single Node OpenShift Requirements](https://docs.openshift.com/container-platform/4.18/installing/installing_sno/install-sno-preparing-to-install-sno.html)

# Install SNO via AI & Set up image registry

Go to console.redhat.com and login
  * Click on “OpenShift →”
  * Click “Create Cluster” from the “Red Hat OpenShift Container Platform” tile
  * It will show “Select an OpenShift cluster type to create”
  * Click the “Datacenter” tab
  * Click “Create cluster” under “Assisted Installer”

Create cluster
  * Name cluster and domain
  * Select OpenShift version & architecture
    * Note: on-cluster image mode is only supported on amd64 today
  * Check SNO box
  * Leave the rest as default and click next
  * Select additional operators. We’ll keep it simple and not include any
  * Hit next
  * Click “Add host”
  * In the next dialog box:
    * Choose the type of ISO (I recommend minimal for iLO attached boot media)
    * Add ssh public key
    * Click “Generate Discovery ISO”
    * You can download the iso or in our case here we are just going to grab the URL

Over to the iLO
  * Set up virtual media URL and set to boot up
  * Boot, get coffee

Back to the console
  * Wait until the node shows up as Ready in the console
  * Next\!
  * Next\!
  * Networking \- we will leave defaults here
  * Next\!
  * Install cluster
  * Go for a quick lunch
  * Download kubeconfig & store it somewhere (the service will keep it for 20 days)
  * Grab kubeadmin password
  * Wait, I can’t connect to the web console!
    * Back to Assited installer page
    * Look for "Not able to access the Web Console?" and click it to get DNS or `/etc/hosts` info and apply it to your workstation or DNS.
    * Login to web console

Let's login to the command line
  * Over to the terminal
    * `export KUBECONFIG=<your kubeconfig>` (better to use absolute path in case you change directories while working with the cluster)
  * Login - one of these will do it!
    * `oc login --web`
    * `oc login`
    * `oc login --insecure-skip-tls-verify=false`

# Enable on-board internal registry (optional)

Let’s enable the internal registry, which is disabled by default on bare metal. If you want to use your own container registry, that is fully supported as well.

* [DOCS LINK](https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/registry/index)

Set strategy for 1 replica for single node (not in docs. Source: [Access article](https://access.redhat.com/solutions/7063960) which also tells you how to set up LVM storage for the registry if desired \- very optional)

```
oc patch config.imageregistry.operator.openshift.io/cluster --type=merge -p '{"spec":{"rolloutStrategy":"Recreate","replicas":1}}'
```

Change image registry’s management state from Removed to Managed

```
oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"managementState":"Managed"}}'
```

Set storage to emptyDir (fine for us here, not for prod use, the storage will be lost on service restart)

```
oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"storage":{"emptyDir":{}}}}'
```

Now the image registry pod should restart

```
oc get pods -n openshift-image-registry
```

Next we need to expose the registry to the host network

```
oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge
```

To check to see if the default route is live
```
oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}'
```


# Enable On-cluster Image Mode

[Official Docs](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/machine_configuration/mco-coreos-layering)

*Note: The upstream documentation still uses "on-cluster layering" terminology.*

Most of these commands will be copied and pasted and will contain explicit namespace references, but to save yourself some typing later you can explicitly change to the MCO namespace. Super helpful while troubleshooting.

```
kubectl config set-context --current --namespace=openshift-machine-config-operator

OR

oc project openshift-machine-config-operator
```

## One-time setup



We are going to use the internal registry so let’s create an imagestream

```
oc create imagestream os-images -n openshift-machine-config-operator
```

Check for image registry pullspec

```
oc get imagestream/os-images  -n openshift-machine-config-operator -o=jsonpath='{.status.dockerImageRepository}'
```

This should return: "image-registry.openshift-image-registry.svc:5000/openshift-machine-config-operator/os-images"

Since we are using an imagestream we need to get the push secret:

```
oc get secrets -o name -n openshift-machine-config-operator -o=jsonpath='{.items[?(@.metadata.annotations.openshift\.io\/internal-registry-auth-token\.service-account=="builder")].metadata.name}'
```

Now we are going to deviate from the quickstart guide. Because this is SNO, we are going to apply this to the master pool and not create a new one. While the node is technically a control plane node and a compute node, the control plane configuration takes precedence.


## Let’s look at our Containerfile/dockerfile

Because there is no local build context we have to use another method of injecting repo definitions for 3rd party integrations. One method is creating a multi-stage build. In this example, we'll use a heredoc to write the file inline.

```
# On-cluster image mode is a multi-stage build where "configs" is the stock image plus
# machineconfig content and "final" is the final image
FROM configs AS final

RUN rpm --import https://downloads.linux.hpe.com/repo/spp/GPG-KEY-spp
RUN rpm --import https://downloads.linux.hpe.com/repo/spp/GPG-KEY2-spp

# Create repo file for Gen 11 repo
RUN cat <<EOF > /etc/yum.repos.d/hpe-sdr.repo
[spp]
name=Service Pack for ProLiant
baseurl=https://downloads.linux.hpe.com/repo/spp-gen11/redhat/9/x86_64/current
enabled=1
gpgcheck=1
gpgkey=https://downloads.linux.hpe.com/repo/spp/GPG-KEY-spp,https://downloads.linux.hpe.com/repo/spp/GPG-KEY2-spp
EOF

# Create repo file for iLOREST
RUN cat <<EOF > /etc/yum.repos.d/ilorest.repo
[ilorest]
name=hpe restful interface tool
baseurl=http://downloads.linux.hpe.com/SDR/repo/ilorest/rhel/9/x86_64/current
enabled=1
gpgcheck=1
gpgkey=https://downloads.linux.hpe.com/repo/spp/GPG-KEY-spp,https://downloads.linux.hpe.com/repo/spp/GPG-KEY2-spp
EOF

# Create directory to satisfy amsd & install packages
RUN mkdir /var/opt && \
    dnf install -y amsd ilorest

# Move the /opt content to the system partition
RUN mkdir /usr/share/amsd && mv /var/opt/amsd/amsd.license /usr/share/amsd/amsd.license && \
    bootc container lint
    # NOTE: on 4.18 use `ostree container commit` in place of
    # `bootc container lint`
```

As noted in the guide:

* Multiple build stages are allowed
* One must use the configs image target (FROM configs AS final) to inject content into it and it must be the last image in the build

The new object in etcd for on-cluster image mode is called the MachineOSConfig. First we create a template for creating ours with this simple script.

```
#!/usr/bin/env bash

# Write the sample MachineOSConfig to a YAML file:
cat << EOF > cluster-image-config.yaml
---
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineOSConfig
metadata:
  name: cluster-image-config
spec:
  machineConfigPool:
    name: master
  containerFile:
  - content: |
      <containerfile contents>

  # Here is where you can select an image builder type. For now, we only
  # support the "Job" type that we maintain ourselves. Future
  # integrations can / will include other build system integrations.
  imageBuilder:
    imageBuilderType: Job

  # Here is where you specify the name of the push secret you use to push
  # your newly-built image to.
  renderedImagePushSecret:
    name: builder-dockercfg-rfl85

  # Here is where you specify the image registry to push your newly-built
  # images to.
  renderedImagePushSpec: image-registry.openshift-image-registry.svc:5000/openshift-machine-config-operator/os-images:latest
EOF
```

Run that to create the template. Then we'll export our variables and update our yaml using `yq`. Remember to replace `builder-dockercfg-rfl85` with the output of the previous command.

Note: [yq](https://github.com/mikefarah/yq) is available in Fedora and Brew. It is not shipped in RHEL.

```
export pushSecretName="builder-dockercfg-rfl85"
export containerfileContents="$(cat Containerfile)"

yq -i e '.spec.renderedImagePushSecret.name = strenv(pushSecretName)' ./cluster-image-config.yaml
yq -i e '.spec.containerFile[0].content = strenv(containerfileContents)' ./cluster-image-config.yaml
```

You can always edit the yaml by hand, but this method pays off when you do it more than a couple of times!


Alright, we’re ready, let’s do this\!

```
oc apply -f cluster-image-config.yaml
```

# Build time

Check to see that the builder controller pod has appeared. You can watch in the web console or at the CLI

```
watch oc get pods -n openshift-machine-config-operator
```

First you will see a new pod for the Machine OS Builder controller, and moment or two after that, a rendered build pod will appear in the list. This is our container build and should look similar to this one

```
build-rendered-master-eeb245f06b1a2ad6f151e282b7866099
```

We can watch the build in progress by tailing the pod logs. This might be our first GA release so be prepared for a LOT of detail\!

```
oc logs -n machine-config-operator <buildpod> -f
```

# Let’s see if it worked

Once the build has succeeded, wait until the cluster restarts itself with the new image

```
$ oc get nodes
NAME                                               STATUS   ROLES                         AGE    VERSION
hpe-dl365gen10plus-01.khw.eng.rdu2.dc.redhat.com   Ready    control-plane,master,worker   3h2m   v1.30.7

$ oc debug node/hpe-dl365gen10plus-01.khw.eng.rdu2.dc.redhat.com
Starting pod/hpe-dl365gen10plus-01khwengrdu2dcredhatcom-debug-qbcw9 ...
To use host binaries, run `chroot /host`
Pod IP: 10.6.10.193
If you don't see a command prompt, try pressing enter.
sh-5.1# chroot /host
sh-5.1# systemctl status amsd
● amsd.service - Agentless Management Service daemon
     Loaded: loaded (/usr/lib/systemd/system/amsd.service; enabled; preset: enabled)
     Active: active (running) since Sat 2025-01-11 19:19:25 UTC; 27min ago
   Main PID: 5144 (amsd)
      Tasks: 1 (limit: 822244)
     Memory: 67.7M
        CPU: 9.440s
     CGroup: /system.slice/amsd.service
             └─5144 /sbin/amsd -f

Jan 11 19:19:24 hpe-dl365gen10plus-01.khw.eng.rdu2.dc.redhat.com systemd[1]: Starting Agentless Management Service daemon...
Jan 11 19:19:25 hpe-dl365gen10plus-01.khw.eng.rdu2.dc.redhat.com amsd[5144]: amsd Started . .
Jan 11 19:19:25 hpe-dl365gen10plus-01.khw.eng.rdu2.dc.redhat.com systemd[1]: Started Agentless Management Service daemon.
sh-5.1# ilorest --version
RESTful Interface Tool 5.3.0.0
sh-5.1# 
```

We can also see that amsd is running back on the iLO.
