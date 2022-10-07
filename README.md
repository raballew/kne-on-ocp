# Kubernetes Network Emulation on OpenShift

Network emulation is the process of imitating certain aspects of the behavior of
network equipment without actually using the target real-world networking
hardware. While there are many different use cases for network emulation one of
the more interesting ones is virtual testing. Virtual testing is the simulation
of a physical test environment. In this context it means removing the bottleneck
of dedicated hardware for developers by executing integration tests against
virtual instances of a production environment to pre-validate hardware tests.
This frees up valuable and often limited hardware resources as certain testing
scopes and most parts of the development process can be offloaded to virtual
instances. Additionally, virtual instances often provide an easier to use
environment that can be changed rapidly to represent any number of production
topologies without anyone needing to physically setup the network.

[Kubernetes Network Emulation (KNE)][kne] lets you run virtual network
topologies in Kubernetes. It does so by running various device OSes in
containers. For anyone interested in a more in-depth view into the inner
workings of KNE I recommend using the projects source code as reference since
the official documentation is a bit lacking.

This documents will guide you through setting up KNE on [OpenShift
(OCP)][openshift-docs] (or on [OKD][okd-docs] respectively) in the first step.
Then you will use the KNE cluster to create and interact with your own topology
based on Arista cEOS. In the last step you will set up a minimal CI workflow
based on [Tekton][tekton] using a virtual instance to pre-validate hardware
tests.

If you struggling to setup an OKD cluster, you might want to follow [this
guide][okd-the-hard-way] to learn it the hard way.

> With minor changes all steps mentioned in this guide are also be usable on
> other Kubernetes distributions.

## Prerequisites

The following system specifications are recommended for the cluster if you plan
to run some workload beyond the scope of this guide:

* x86_64 cluster system architecture
* Dynamic storage provisioning configured
* At least 3 worker nodes
* At least 8 CPU cores per worker node
* At least 32 GB of RAM per worker node
* At least 1 GBit/s network interfaces on all nodes
* Cluster administrator permissions

You also will be able to use a much smaller setup but it would significantly
impact performance and should only be used for evaluation.

First clone this repository:

```bash

```

## Deploy meshnet CNI

`meshnet` is a Kubernetes CNI plugin that allows you to create arbitrary virtual
network topologies. It interconnects pods via direct point-to-point links
according to pre-defined topologies.

Deploy `meshnet`:

```bash
oc apply -k manifests/meshnet/overlays/openshift
```

Wait until `meshnet` is ready:

```bash
oc rollout status daemonset meshnet -n meshnet
```

Test if `meshnet` works:

```bash
oc create namespace meshnet-test
oc apply -f manifests/meshnet-test.yaml -n meshnet-test
oc exec r1 -n meshnet-test -- ping -c 1 12.12.12.2
oc delete namespace meshnet-test
```

## Deploy MetalLB

> If you have already set up a load balancer so that Services of type `Load
> Balancer` with external IP addresses can be used or if no cluster external
> access to the virtual environment is required this skip can be skipped.

Follow the instructions in the [official MetalLB documentation][metallb-docs]
and its [notes for OCP and OKD][metallb-notes].

## Getting cEOS image

Arista requires its users to register with
[arista.com][arista-software-download] before downloading any images.

> Make sure to register as with the user role Partner or Customer (do not use
Guest role) because otherwise you might not be able to download the required
artifacts. If you are already registered you can change the role in your profile
settings.

Once you created an account and logged in, go to the software downloads section
and download a 64-bit release of cEOS-lab.

> This guide has been tested with
> `cEOS-lab/EOS-4.28.3M/cEOS64-lab-4.28.3M.tar.xz`. When you download this file
> the `.xz` suffix will be missing.

Then expose the internal registry and push the container image into it. Finally,
set the internal registry back to private.

```bash
oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge
HOST=$(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')
podman import cEOS64-lab-4.28.3M.tar ceos64:4.28.3M
podman tag localhost/ceos64:4.28.3M $HOST/openshift/ceos64:4.28.3M
podman login -u kubeadmin -p $(oc whoami -t) $HOST
podman push $HOST/openshift/ceos64:4.28.3M
oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":false}}' --type=merge
```

## Setup KNE

Follow the instructions in the [official KNE documentation][kne-docs] on how to
setup KNE.

## Create topology

Some vendors provide a controller that handles the pod lifecycle for their
nodes. If you did not specify these in your deployment configuration, you will
need to manually create them before deploying a topology. For cEOS Arista
provides a controller that handles the pod lifecycle for their nodes.

Deploy the Arista controller:

```bash
oc apply -f https://raw.githubusercontent.com/aristanetworks/arista-ceoslab-operator/main/config/kustomized/manifest.yaml
```

Additionally, Keysight provides an operator for traffic generation as well.

Make sure that the version of the `IXIA_TG` node (local-latest) in
[3-node-ceos-withtraffic.pb.txt](/topologies/3-node-ceos-withtraffic.pb.txt) is
similar to the version of specified in
[configmap.yaml](/manifests/ixiatg/configmap.yaml) that is going to be applied
in the next step.

Deploy the Ixia-C operator:

```bash
oc apply -f https://github.com/open-traffic-generator/ixia-c-operator/releases/download/v0.2.2/ixiatg-operator.yaml
oc patch deploy ixiatg-op-controller-manager -n ixiatg-op-system --type json -p='[{"op": "replace", "path": "/spec/template/spec/containers/1/resources/limits/memory", "value":"200Mi"}]'
oc apply -f manifests/ixiatg/configmap.yaml
```

Make sure the correct `image` is set for `ARISTA_CEOS` nodes in
[3-node-ceos-withtraffic.pb.txt](/topologies/3-node-ceos-withtraffic.pb.txt) if
you use a different version of cEOS.

Deploy the topology into a new namespace:

```bash
namespace=3-node-ceos-withtraffic
oc create namespace $namespace
tmp_dir=$(mktemp -d)
cp -r topologies/ $tmp_dir
echo "name: \"$namespace\"" >> $tmp_dir/topologies/3-node-ceos-withtraffic.pb.txt
kne create $tmp_dir/topologies/3-node-ceos-withtraffic.pb.txt --kubecfg $KUBECONFIG
```

Verify that the virtual instances are working:

```bash
oc exec -it -n $namespace r1 -- Cli
oc exec -it -n $namespace r2 -- Cli
oc exec -it -n $namespace r3 -- Cli
```

Push a configuration to a virtual instance:

```bash

```

Delete the topology:

```bash
oc delete namespace $namespace
```

Where:

* `$KUBECONFIG` - List of paths to configuration files used to configure access
  to a cluster

[kne]: https://github.com/openconfig/kne
[kne-docs]: https://github.com/openconfig/kne/blob/main/docs/setup.md
[okd-docs]: https://docs.okd.io/
[openshift-docs]: https://docs.openshift.com/
[okd-the-hard-way]: https://github.com/raballew/okd-the-hard-way
[metallb-docs]: https://metallb.universe.tf/installation/
[metallb-notes]:
    https://metallb.universe.tf/installation/clouds/#metallb-on-openshift-ocp
[tekton]: https://tekton.dev/docs/
[arista-software-download]: https://www.arista.com/en/support/software-download
