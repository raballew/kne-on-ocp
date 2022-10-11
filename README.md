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

First, clone the source code required for this guide:

```bash
git clone git@github.com:raballew/kne-on-ocp.git
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
nodes. For cEOS Arista provides a controller.

Deploy the Arista controller:

```bash
oc apply -f https://raw.githubusercontent.com/aristanetworks/arista-ceoslab-operator/v1.0.2/config/kustomized/manifest.yaml
```

Make sure the correct `image` is set for `ARISTA_CEOS` nodes in
[3-node-ceos.pb.txt](/topologies/3-node-ceos.pb.txt) if you use a different
version of cEOS.

Deploy the topology into a new namespace:

```bash
namespace=3-node-ceos
oc create namespace $namespace
tmp_dir=$(mktemp -d)
cp -r topologies/ $tmp_dir
echo "name: \"$namespace\"" >> $tmp_dir/topologies/3-node-ceos.pb.txt
kne create $tmp_dir/topologies/3-node-ceos.pb.txt --kubecfg $KUBECONFIG
```

Verify that the virtual instances are working:

```bash
oc exec -it -n $namespace r1 -- Cli -c "show bgp instance"
oc exec -it -n $namespace r2 -- Cli -c "show bgp instance"
oc exec -it -n $namespace r3 -- Cli -c "show bgp instance"
```

Delete the topology:

```bash
oc delete namespace $namespace
```

Where:

* `$KUBECONFIG` - List of paths to configuration files used to configure access
  to a cluster

## Traffic generation

IXIA offers an operator that through a CRD allows the deployment of a modern,
powerful and API-driven traffic generator.

Install the IXIA traffic generator:

```bash
oc apply -f https://github.com/open-traffic-generator/ixia-c-operator/releases/download/v0.2.2/ixiatg-operator.yaml
# Fixes https://github.com/open-traffic-generator/ixia-c-operator/issues/15
oc set resources deployment ixiatg-op-controller-manager -n ixiatg-op-system --limits memory=200Mi
```

> For the deployment of IXIA with custom images the version for nodes of type
> `IXIA_TG` specified in
> [3-node-ceos-with-traffic.pb.txt](/topologies/3-node-ceos-with-traffic.pb.txt)
> needs to match the release value at `.spec.data.versions` in
> [config.yaml](/manifests/ixiatg/config.yaml). If you want to use a different
> version, make sure to adjust both files accordingly before applying them to
> the cluster. The latest upstream version of this configuration are published
> via [ixia-c-operator
> releases](https://github.com/open-traffic-generator/ixia-c/releases/).

Configure the usage of public available container images:

```bash
oc apply -f manifests/ixiatg/config.yaml
```

Deploy a topology with traffic generation into a new namespace:

```bash
namespace=3-node-ceos-with-traffic
oc create namespace $namespace
# Fixes https://github.com/open-traffic-generator/ixia-c-operator/issues/18
# Fixes https://github.com/open-traffic-generator/ixia-c-operator/issues/19
oc apply -f manifests/ixiatg/rbac.yaml -n $namespace
tmp_dir=$(mktemp -d)
cp -r topologies/ $tmp_dir
echo "name: \"$namespace\"" >> $tmp_dir/topologies/3-node-ceos-with-traffic.pb.txt
kne create $tmp_dir/topologies/3-node-ceos-with-traffic.pb.txt --kubecfg $KUBECONFIG
```

Where:

* `$KUBECONFIG` - List of paths to configuration files used to configure access
  to a cluster

Accessing the nodes:

```bash
oc get services -n $namespace
NAME                           TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                     AGE
service-gnmi-otg-controller    LoadBalancer   172.30.189.156   <REDACTED>    50051:31073/TCP                             1m
service-grpc-otg-controller    LoadBalancer   172.30.85.47     <REDACTED>    40051:30137/TCP                             1m
service-https-otg-controller   LoadBalancer   172.30.125.200   <REDACTED>    443:30903/TCP                               1m
service-otg-port-eth1          LoadBalancer   172.30.82.179    <REDACTED>    5555:30081/TCP,50071:31381/TCP              1m
service-otg-port-eth2          LoadBalancer   172.30.30.247    <REDACTED>    5555:31032/TCP,50071:32591/TCP              1m
service-r1                     LoadBalancer   172.30.137.76    <REDACTED>    443:32511/TCP,22:31692/TCP,6030:32185/TCP   1m
service-r2                     LoadBalancer   172.30.197.125   <REDACTED>    443:31446/TCP,22:30730/TCP,6030:32575/TCP   1m
service-r3                     LoadBalancer   172.30.135.159   <REDACTED>    443:30206/TCP,22:30048/TCP,6030:30450/TCP   1m
```

You can access each node and the traffic generator through its `CLUSTER-IP` and
`EXTERNAL-IP` depending on your setup. Additionally you could use the Kubernetes
DNS service to access each service via its DNS A record
(`<svc>.<namespace>.svc.cluster.local`).

As of now a topology consisting of several switches and a reference
implementation of the [Open Traffic Generator
API](https://github.com/open-traffic-generator/models) has been deployed with
the [IXIA-C traffic
generator](https://github.com/open-traffic-generator/ixia-c). In order to
generate traffic test scripts are written in
[snappi](https://github.com/open-traffic-generator/snappi) and executed against
the IXIA-C service (`*-otg-controller` services as shown above). Another way of
running tests is by using the [Open Traffic Generator CLI
Tool](https://github.com/open-traffic-generator/otgen), which we for simplicity
will use in the next step.

Deploy a job that triggers a workflow for generating traffic:

```bash
oc apply -f manifests/ixiatg-test.yaml -n $namespace
```

Delete the topology:

```bash
oc delete namespace $namespace
```

## Limitations

IXIA traffic generation as described in the [KNE examples][kne-examples] does
not seem to work on OpenShift out of the box as IXIA requires additional
privileges and seem to have problems handling arbitrary UID which are enforced
by OpenShift. Due to this, containers images have been rebuilt and a bunch of
patches have been applied to overcome these shortcomings. Keep in mind, that
before considering going into production with this a bunch of changes need to
happen upstream first.

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
[kne-examples]:
    https://github.com/openconfig/kne/blob/main/examples/3node-withtraffic.pb.txt
