# LINKS

* [https://console.redhat.com/openshift/assisted-installer/clusters](https://console.redhat.com/openshift/assisted-installer/clusters)

# Install SNO via AI & Set up image registry

* Go to console.redhat.com and login
  * Click on “OpenShift →”
  * Click “Create Cluster” from the “Red Hat OpenShift Container Platform” tile
  * It will show “Select an OpenShift cluster type to create”
  * Click the “Datacenter” tab
  * Click “Create cluster” under “Assisted Installer”
* Create cluster
  * Name cluster and domain
  * Select OpenShift version & architecture
    * Note: layering is only supported on amd64 today
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
* Over to the iLO
  * Set up virtual media URL and set to boot up
  * Boot, get coffee
* Back to the console
  * Wait until the node shows up as Ready in the console
  * Next\!
  * Next\!
  * Networking \- we will leave defaults here
  * Next\!
  * Install cluster
  * Go for a quick lunch
  * Download kubeconfig & store it somewhere (the service will keep it for 20 days)
  * Grab kubeadmin password
  * Over to the terminal
    * `export KUBECONFIG=`
    * `oc login`
  * Wait, can’t connect
    * Back to console and copy DNS info
  * Login - one of these will do it!
    * `oc login --web`
    * `oc login`
    * `oc login --insecure-skip-tls-verify=false`

# Enable on-board internal registry (optional)

Let’s enable the internal registry, which is disabled by default on bare metal, and other non-cloud deployments. If you want to use your own container registry, that is fully supported as well.

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

* Now the image registry pod should start up
  * `oc get pods -n openshift-image-registry`
  * Next we need to expose the registry to the host network

```
oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge
```

# Enable feature gate to enable tech preview features (4.17)

[Docs](https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/hosted_control_planes/hcp-using-feature-gates#hcp-enable-feature-sets_hcp-using-feature-gates)

enable-feature-gate.yaml

```
apiVersion: config.openshift.io/v1
kind: FeatureGate
metadata:
  name: cluster
# ...
spec:
  featureSet: TechPreviewNoUpgrade
```

Note: enabling TechPreviewNoUpgrade does exactly what it says: it permanently prevents this machine from updating to newer builds

```
oc apply -f enable-feature-gate.yaml
```

# Enable OCL

[Full Quickstart Guide](https://github.com/openshift/machine-config-operator/blob/master/docs/onclusterlayering-quickstart.md)  
[OCL Troubleshooting](https://github.com/openshift/machine-config-operator/blob/master/docs/onclusterlayering-troubleshooting.md)

Most of these commands will be copied and pasted and will contain explicit namespace references, but to save yourself some typing later you can explicitly change to the MCO namespace. Super helpful while troubleshooting.

```
kubectl config set-context --current --namespace=openshift-machine-config-operator

OR

oc project openshift-machine-config-operator
```

Power users might want to look into a free tool called [K9s](https://k9scli.io/) (not shipped by Red Hat, just a suggestion\!)

## One-time setup

Copy cluster pull secret to the MCO namespace (make sure you use bash, it definitely doesn't work with `/usr/bin/fish`)

```
oc create secret docker-registry global-pull-secret-copy \
  --namespace "openshift-machine-config-operator" \
  --from-file=.dockerconfigjson=<(oc get secret/pull-secret -n openshift-config \
  -o go-template='{{index .data ".dockerconfigjson" | base64decode}}')
```

Copy RHEL entitlement to MCO namespace (also bash)

```
oc create secret generic etc-pki-entitlement \
  --namespace "openshift-machine-config-operator" \
  --from-file=entitlement.pem=<(oc get secret/etc-pki-entitlement \
  -n openshift-config-managed -o go-template='{{index .data "entitlement.pem" | base64decode }}') \
  --from-file=entitlement-key.pem=<(oc get secret/etc-pki-entitlement \
  -n openshift-config-managed -o go-template='{{index .data "entitlement-key.pem" | base64decode }}')
```

We are going to use the internal registry so let’s create an imagestream

```
oc create imagestream os-images -n openshift-machine-config-operator
```

Check for image registry pullspec

```
oc get imagestream/os-images  -n openshift-machine-config-operator -o=jsonpath='{.status.dockerImageRepository}'
```

This should return: "image-registry.openshift-image-registry.svc:5000/openshift-machine-config-operator/os-images"

Since we are using an imagestream we need to get the push and pull secret:

```
oc get secrets -o name -n openshift-machine-config-operator -o=jsonpath='{.items[?(@.metadata.annotations.openshift\.io\/internal-registry-auth-token\.service-account=="builder")].metadata.name}'
```

Now we are going to deviate from the quickstart guide. Because this is SNO, we are going to apply this to the master pool and not create a new one. While the node is technically a control plane node and a compute node, the control plane configuration takes precedence.

Let’s look at our Containerfile/dockerfile

```
FROM configs AS final

RUN rpm --import https://downloads.linux.hpe.com/repo/spp/GPG-KEY-spp
RUN rpm --import https://downloads.linux.hpe.com/repo/spp/GPG-KEY2-spp
RUN rpm --import https://downloads.linux.hpe.com/SDR/hpePublicKey2048_key1.pub
RUN rpm --import https://downloads.linux.hpe.com/SDR/hpePublicKey2048_key2.pub

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
gpgcheck=0
gpgkey=https://downloads.linux.hpe.com/repo/spp/GPG-KEY-spp,https://downloads.linux.hpe.com/repo/spp/GPG-KEY2-spp
EOF

RUN mkdir /var/opt
RUN dnf install -y amsd ilorest

RUN ostree container commit
```

As noted in the guide:

* Multiple build stages are allowed
* One must use the configs image target (FROM configs AS final) to inject content into it and it must be the last image in the build

The new object in etcd for the layered build is called the MachineOSConfig. First we create a template for creating ours.

```
#!/usr/bin/env bash

# Write the sample MachineOSConfig to a YAML file:
cat << EOF > layered-machineosconfig.yaml
---
apiVersion: machineconfiguration.openshift.io/v1alpha1
kind: MachineOSConfig
metadata:
  name: layered
spec:
  # Here is where you refer to the MachineConfigPool that you want your built
  # image to be deployed to.
  machineConfigPool:
    name: layered
  buildInputs:
    containerFile:
    # Here is where you can set the Containerfile for your MachineConfigPool.
    # You'll note that you need to provide an architecture. This is because this
    # will eventually support multiarch clusters. For now, only noArch is
    # supported.
    - containerfileArch: noarch
      content: |-
        <containerfile contents>
    # Here is where you can select an image builder type. For now, we only
    # support the "PodImageBuilder" type that we maintain ourselves. Future
    # integrations can / will include other build system integrations.
    imageBuilder:
      imageBuilderType: PodImageBuilder
    # The Machine OS Builder needs to know what pull secret it can use to pull
    # the base OS image.
    baseImagePullSecret:
      name: <secret-name>
    # Here is where you specify the name of the push secret you use to push
    # your newly-built image to.
    renderedImagePushSecret:
      name: <secret-name>
    # Here is where you specify the image registry to push your newly-built
    # images to.
    renderedImagePushspec: <final image pullspec>
  buildOutputs:
    # Here is where you specify what image will be used on all of your nodes to
    # pull the newly-built image.
    currentImagePullSecret:
      name: <secret-name>
EOF
```

Now export the variables we want and update our yaml, remember to replace `builder-dockercfg-123` with the output of `oc get secrets -o name -n openshift-machine-config-operator -o=jsonpath='{.items[?(@.metadata.annotations.openshift\.io\/internal-registry-auth-token\.service-account=="builder")].metadata.name}'`

Note: [yq](https://github.com/mikefarah/yq) is available in Fedora and Brew. It is not shipped in RHEL.

```
export pushSecretName="builder-dockercfg-123"
export pullSecretName="builder-dockercfg-123"
export baseImagePullSecretName="global-pull-secret-copy"
export containerfileContents="$(cat Containerfile)"
export imageRegistryPullspec="image-registry.openshift-image-registry.svc:5000/openshift-machine-config-operator/os-images:latest"

yq -i e '.spec.buildInputs.baseImagePullSecret.name = strenv(baseImagePullSecretName)' ./layered-machineosconfig.yaml
yq -i e '.spec.buildInputs.renderedImagePushSecret.name = strenv(pushSecretName)' ./layered-machineosconfig.yaml
yq -i e '.spec.buildOutputs.currentImagePullSecret.name = strenv(pullSecretName)' ./layered-machineosconfig.yaml
yq -i e '.spec.buildInputs.containerFile[0].content = strenv(containerfileContents)' ./layered-machineosconfig.yaml
yq -i e '.spec.buildInputs.renderedImagePushspec = strenv(imageRegistryPullspec)' ./layered-machineosconfig.yaml
```

You can always edit the yaml by hand, but this method pays off when you do it more than a couple of times\!

Let’s take a look at the final yaml before we apply it. Ok, we need to make one more **important** change, in the “spec” section look for

```
  machineConfigPool:
    name: layered
```

Change “layered” to “master” because we are targeting the master pool. In SNO, the node is both worker and control-plane, but the control-plane configuration overrides “worker”.

Alright, we’re ready, let’s do this\!

```
oc apply -f layered-machineosconfig.yaml
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

We can watch the build in progress by tailing the pod logs. This is our first GA release so be prepared for a LOT of detail\!

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

# Recovery

More info in the troubleshooting guide, but these are the commands you need to clear the objects created during a failed build.

```
oc get configmaps -n openshift-machine-config-operator -l 'machineconfiguration.openshift.io/on-cluster-layering=' \
-o name | xargs oc delete -n openshift-machine-config-operator

oc get secrets -n openshift-machine-config-operator -l 'machineconfiguration.openshift.io/on-cluster-layering=' \
-o name | xargs oc delete -n openshift-machine-config-operator

oc get pods -n openshift-machine-config-operator -l 'machineconfiguration.openshift.io/on-cluster-layering=' \
-o name | xargs oc delete -n openshift-machine-config-operator
```

Delete the machineconfig, fix it, reapply it.

```
oc delete machineosconfig/layered
```

