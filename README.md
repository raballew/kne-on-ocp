# Kubernetes Network Emulation on OpenShift

Network emulation is the process of imitating certain aspects of the behavior of
network equipment without actually using any target real-world networking
hardware. While there are many different use cases for network emulation one of
the more interesting ones is virtual testing. Virtual testing is the simulation
of a physical test environment. In this context it means removing the bottleneck
of dedicated hardware for developers by executing integration tests against
virtual instances of a production environment to pre-validate hardware tests.
This frees up valuable and often limited hardware resources as certain testing
scopes and most parts of the development process can be offloaded to virtual
instances. Additionally, virtual instances often provide an easier to use
environment that can be changed rapidly to represent any production topology
without anyone needing to physically setup the network.

[Kubernetes Network Emulation (KNE)][kne] lets you run virtual network
topologies in Kubernetes. It does so by running various device operating systems
in containers. For anyone interested in a more in-depth view into the inner
workings of KNE the projects source code should act as the reference since the
documentation is a bit lacking.

This document will guide you through setting up KNE on [OpenShift
(OCP)][openshift-docs] (or on [OKD][okd-docs] respectively) in the first step.
Then you will use the KNE cluster to create and interact with a topology based
on [Arista cEOS][arista-ceos]. In the last step you will set up a minimal
workflow using the same topology attached to virtual instances of the [Ixia-C
Open Traffic Generator][ixia-c-otg] to pre-validate hardware tests. This should
give you a good understanding of what virtual testing is all about and allow you
to adopt this concept to more complex CI workflows leveraging your choice of
tools.

If you are struggling to setup an OKD cluster, you might want to follow [this
guide][okd-the-hard-way] to learn it the hard way.

> With a few changes all steps mentioned in this guide are also usable on other
> Kubernetes distributions.

## Prerequisites

This guide assumes that you are familiar with a few technologies on a
fundamental level:

* OpenShift
* Operators
* Networking

The following system specifications are required for the cluster in order to
follow the guide and run some additional workload:

* x86_64 cluster system architecture
* At least 3 worker nodes
* At least 8 CPU cores per worker node
* At least 16 GB of RAM per worker node
* At least 1 GBit/s network interfaces on all nodes
* Cluster administrator permissions
* Internet access

If the cluster worker nodes are hosted on a hypervisor, make sure to pass-trough
the CPU information as features such as SSSE3 are required to run traffic
generation successfully. A smaller cluster might be feasible as well but it
could significantly impact performance. For the development of this guide an
OpenShift 4.11 cluster has been used.

To get started clone the source code required for this guide:

```bash
git clone https://github.com/raballew/kne-on-ocp.git
cd kne-on-ocp
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

`meshnet` is successfully configured, if you have been able to ping `12.12.12.2`
from pod `r1`.

## Deploy MetalLB

> This step is optional if you have already set up a load balancer so that
> services of type `LoadBalancer` with external IP addresses can be used or if
> no cluster external access to the virtual environment is required.

Follow the instructions in the [official MetalLB documentation][metallb-docs]
and its [notes for OCP and OKD][metallb-notes].

## Getting cEOS image

Arista requires its users to register at [arista.com][arista-software-download]
before downloading any container images.

> Make sure to register with the user role Partner or Customer (do not use Guest
> role) because otherwise you might not be able to download the required
> artifacts. If you are already registered you can change the role in your
> profile settings.

Once you created an account and logged in, go to the software downloads section
and download a 64-bit release of cEOS-lab.

> This guide has been tested with
> `cEOS-lab/EOS-4.28.3M/cEOS64-lab-4.28.3M.tar.xz`. When you download this file
> the `.xz` suffix will be missing.

Expose the internal registry and push the container image into it. Finally, set
the internal registry back to private.

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
nodes. Arista provides a controller for cEOS nodes.

Deploy the Arista controller:

```bash
oc apply -f https://raw.githubusercontent.com/aristanetworks/arista-ceoslab-operator/v2.0.1/config/kustomized/manifest.yaml
```

Deploy a basic topology with three cEOS virtual instances (`r1`, `r2`, `r3`)
into a new namespace.

```txt
      ┌────┐                  ┌────┐                  ┌────┐
      │eth3│                  │eth4│                  │eth5│
    ┌─┴────┴──┐             ┌─┴────┴──┐             ┌─┴────┴──┐
    │   r1    │             │   r2    │             │   r3    │
    ├────┬────┤             ├────┬────┤             ├────┬────┤
    │eth1│eth2│             │eth1│eth2│             │eth1│eth2│
    └┬───┴──┬─┘             └─┬──┴───┬┘             └─┬──┴───┬┘
     │      │                 │      │                │      │
     │ 1.2.0.1/30      1.2.0.2/30  1.2.0.9/30    1.2.0.10/30 │
     │      │                 │      │                │      │
     │      └─────────────────┘      └────────────────┘      │
1.2.0.5/30                                              1.2.0.6/30
     │                                                       │
     └───────────────────────────────────────────────────────┘

r1: AS1, 2.2.2.1/32
r2: AS1, 2.2.2.2/32
r3: AS1, 2.2.2.3/32
```

> Make sure the correct `image` is set for `ARISTA` nodes in
> [3-node-ceos.pb.txt](topologies/3-node-ceos.pb.txt) if you use a different
> version of cEOS.

```bash
namespace=3-node-ceos
oc create namespace $namespace
# Fixes https://github.com/open-traffic-generator/ixia-c-operator/issues/18
# Fixes https://github.com/aristanetworks/arista-ceoslab-operator/issues/5
oc apply -f manifests/rbac/privileged-patch.yaml -n $namespace
tmp_dir=$(mktemp -d)
cp -r topologies/ $tmp_dir
echo "name: \"$namespace\"" >> $tmp_dir/topologies/3-node-ceos.pb.txt
kne create $tmp_dir/topologies/3-node-ceos.pb.txt --kubecfg $KUBECONFIG
```

Where:

* `$KUBECONFIG` - List of paths to configuration files used to configure access
  to a cluster

> Do not interrupt the `kne` command. It can take minutes until it returns. Just
> be patient and wait.

Wait a few minuted and then verify that the virtual instances are working
properly:

```bash
oc exec -it -n $namespace r1 -- Cli -c "show bgp statistics"
oc exec -it -n $namespace r2 -- Cli -c "show bgp statistics"
oc exec -it -n $namespace r3 -- Cli -c "show bgp statistics"
```

> If you check the output you will realize that BGP does not seem to work
> properly, as each virtual instances shows that `1 neighbor is in Idle(NoIf)
> state`. This is due to the fact, that in this topology the traffic generator
> is not attached yet and can be ignored for the moment.

You can also access each virtual instance by using their external or cluster IP
address. Additionally you could use the Kubernetes DNS service to address each
service via its DNS A record (`<svc>.<namespace>.svc.cluster.local`).

```bash
oc get services -n $namespace

NAME                           TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                     AGE
service-r1                     LoadBalancer   172.30.203.182   <REDACTED>    443:31420/TCP,22:32358/TCP,6030:31958/TCP   29s
service-r2                     LoadBalancer   172.30.23.251    <REDACTED>    22:32103/TCP,6030:32362/TCP,443:32467/TCP   28s
service-r3                     LoadBalancer   172.30.61.134    <REDACTED>    443:31661/TCP,22:32477/TCP,6030:32122/TCP   28s
```

Delete the topology:

```bash
oc delete namespace $namespace
```

## Traffic generation

The [OTG project][otg] offers an operator that through a CRD allows the
deployment of a modern, powerful and API-driven open source traffic generator
with limited functionality. A commercially supported version is available
through the [Keysight Elastic Network
Generator][keysight-elastic-network-generator] which offers advanced
functionality such as the emulation of key data center OSI layer 2 and OSI layer
3 control plane protocols. While simple traffic generation works with the open
source version as well, the topologies deployed in this guide rely on BGP, a
control plane protocol. Hence, in order to run a meaningful test the traffic
generator instances need to be configured using the protocol engine - a
functionality that is only available in the commercially supported version.

> Reach out to the [Keysight Support][keysight-support] in order to gain access
> to the container images for the commercially supported version.

Install the Ixia-C traffic generator:

```bash
oc apply -f https://github.com/open-traffic-generator/ixia-c-operator/releases/download/v0.2.6/ixiatg-operator.yaml
```

You need to decide if you want to use publicly available container images for
the open-source version or container images hosted on a private registry for the
commercially supported version.

If you want to use the open-source version apply the following configuration:

```bash
oc apply -f manifests/ixiatg/config-open-source.yaml
```

If you want to use the commercially supported version apply the following
configuration and [update the global cluster pull secret][global-pull-secret] by
appending a new pull secret for Keysights private container registry:

```bash
oc apply -f manifests/ixiatg/config-licensed.yaml
```

> For the deployment of the Ixia-C traffic generator with custom images the
> version for nodes of type `IXIA_TG` specified in
> [3-node-ceos-with-traffic.pb.txt](topologies/3-node-ceos-with-traffic.pb.txt)
> needs to match the release value at `.spec.data.versions` in
> [config-open-source.yaml](manifests/ixiatg/config-open-source.yaml) or
> [config-licensed.yaml](manifests/ixiatg/config-licensed.yaml). If you want to
> use a different version, make sure to adjust all files accordingly before
> applying them to the cluster. The latest upstream version of this
> configuration is published on the [ixia-c-operator
> releases](https://github.com/open-traffic-generator/ixia-c/releases/) page.

[3-node-ceos-with-traffic.pb.txt](topologies/3-node-ceos-with-traffic.pb.txt)
uses the same topology with a broken BGP configuration as
[3-node-ceos.pb.txt](topologies/3-node-ceos.pb.txt) but added services for
traffic generation.

```txt
                             ┌───────┐
                             │       │
                             │       │
                            ┌┴───┬───┴┐
                            │eth4│eth5│
                         ┌──┴────┴────┴─┐
                         │      otg     │
                         ├────┬────┬────┤
          ┌──────────────┤eth1│eth2│eth3├──────────────┐
          │              └────┴─┬──┴────┘              │
          │                     │                      │
    10.10.10.1/24         20.20.20.1/24          30.30.30.1/24
          │                     │                      │
          │                     │                      │
    10.10.10.2/24         20.20.20.2/24          30.30.30.2/24
          │                     │                      │
      ┌───┴┐                  ┌─┴──┐                  ┌┴───┐
      │eth3│                  │eth4│                  │eth5│
    ┌─┴────┴──┐             ┌─┴────┴──┐             ┌─┴────┴──┐
    │   r1    │             │   r2    │             │   r3    │
    ├────┬────┤             ├────┬────┤             ├────┬────┤
    │eth1│eth2│             │eth1│eth2│             │eth1│eth2│
    └┬───┴──┬─┘             └─┬──┴───┬┘             └─┬──┴───┬┘
     │      │                 │      │                │      │
     │ 1.2.0.1/30      1.2.0.2/30  1.2.0.9/30    1.2.0.10/30 │
     │      │                 │      │                │      │
     │      └─────────────────┘      └────────────────┘      │
1.2.0.5/30                                              1.2.0.6/30
     │                                                       │
     └───────────────────────────────────────────────────────┘

r1: AS1, 2.2.2.1/32
r2: AS1, 2.2.2.2/32
r3: AS1, 2.2.2.3/32
otg/eth1: AS1111, 2.2.2.4/32
otg/eth2: AS2222, 2.2.2.5/32
otg/eth3: AS3333, 2.2.2.6/32
```

Deploy this topology into a new namespace:

> Make sure the correct `image` is set for `ARISTA` nodes in
> [3-node-ceos-with-traffic.pb.txt](topologies/3-node-ceos-with-traffic.pb.txt)
> if you use a different version of cEOS.

```bash
namespace=3-node-ceos-with-traffic
oc create namespace $namespace
# Fixes https://github.com/open-traffic-generator/ixia-c-operator/issues/18
# Fixes https://github.com/aristanetworks/arista-ceoslab-operator/issues/5
oc apply -f manifests/rbac/privileged-patch.yaml -n $namespace
tmp_dir=$(mktemp -d)
cp -r topologies/ $tmp_dir
echo "name: \"$namespace\"" >> $tmp_dir/topologies/3-node-ceos-with-traffic.pb.txt
kne create $tmp_dir/topologies/3-node-ceos-with-traffic.pb.txt --kubecfg $KUBECONFIG
```

Where:

* `$KUBECONFIG` - List of paths to configuration files used to configure access
  to a cluster

> Do not interrupt the `kne` command. It can take minutes until it returns. Just
> be patient and wait.

Once the command returns successfully, a new topology consisting of several
switches and a reference implementation of the [Open Traffic Generator
API](https://github.com/open-traffic-generator/models) has been deployed with
the [Ixia-C traffic
generator](https://github.com/open-traffic-generator/ixia-c).

```bash
oc get services -n $namespace

NAME                           TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                     AGE
service-gnmi-otg-controller    LoadBalancer   172.30.36.53     <REDACTED>    50051:31039/TCP                             43s
service-grpc-otg-controller    LoadBalancer   172.30.148.74    <REDACTED>    40051:31636/TCP                             43s
service-https-otg-controller   LoadBalancer   172.30.226.247   <REDACTED>    443:30120/TCP                               43s
service-otg-port-eth1          LoadBalancer   172.30.17.198    <REDACTED>    5555:30754/TCP,50071:30271/TCP              43s
service-otg-port-eth2          LoadBalancer   172.30.94.133    <REDACTED>    5555:31173/TCP,50071:31540/TCP              43s
service-otg-port-eth3          LoadBalancer   172.30.94.236    <REDACTED>    5555:30410/TCP,50071:31177/TCP              43s
service-otg-port-eth4          LoadBalancer   172.30.88.101    <REDACTED>    5555:32371/TCP,50071:31531/TCP              43s
service-otg-port-eth5          LoadBalancer   172.30.209.82    <REDACTED>    5555:31945/TCP,50071:31893/TCP              43s
service-r1                     LoadBalancer   172.30.99.70     <REDACTED>    443:30896/TCP,22:30113/TCP,6030:30300/TCP   43s
service-r2                     LoadBalancer   172.30.60.68     <REDACTED>    443:31079/TCP,22:30859/TCP,6030:30132/TCP   43s
service-r3                     LoadBalancer   172.30.236.73    <REDACTED>    6030:32008/TCP,443:30317/TCP,22:32145/TCP   43s
```

In order to generate traffic scripts using
[snappi](https://github.com/open-traffic-generator/snappi) can be executed
against the newly deployed services (`*-otg-controller` and `*-otg-port-eth*`
services as shown above). Another, more simplistic way of running tests is by
using the [Open Traffic Generator CLI
tool](https://github.com/open-traffic-generator/otgen) which will be used in
this guide.

In order to validate the functionality of the traffic generator deploy a job
that triggers a back-to-back test using a direct link between two ports (`eth4`
and `eth5`) on the traffic generator:

```bash
oc create -f flows/job-flow-otg-otg.yaml -n $namespace
```

By inspecting the logs, for each flow `eth4>eth5` and `eth5>eth4`, the number of
frames received is equal to the number of frames sent. This indicates that this
particular connection works fine for the test parameters and the traffic
generator is operational.

```bash
oc get job -l flow=otg-otg -o name -n $namespace | xargs oc logs -n $namespace -f

+-----------+-----------+-----------+
|   NAME    | FRAMES TX | FRAMES RX |
+-----------+-----------+-----------+
| eth4>eth5 |      1000 |      1000 |
| eth5>eth4 |      1000 |      1000 |
+-----------+-----------+-----------+
```

> The following steps require access to the commercially supported version of
> Ixia-C traffic generator. Using the open-source version does not work as it
> does not provide a protocol-engine container image. Reach out to the [Keysight
> Support][keysight-support] in order to gain access to their private container
> registries.

Lets try something more complex and see if our initial assumption, that the
topology of the virtual switch instances is broken, remains true. This can be
done by running a flow that tries to send traffic from ports `eth1` (connected
to virtual instance `r1`), `eth2` (connected to virtual instance `r2`) and
`eth3` (connected to virtual instance `r3`) to all other ports. In theory, if
everything is configured properly, this should create a similar output as in the
previous flow but as a cautious reader you probably already know that something
is broken.

```bash
oc create -f flows/job-flow-r1-r2-r3.yaml -n $namespace
```

When inspecting the logs they confirm our assumption because no frames are
received on `eth1` at all, indicating that the connection with `r2` (`eth2`) and
`r3` (`eth3`) is misconfigured.

```bash
oc get job -l flow=r1-r2-r3 -o name -n $namespace | xargs oc logs -n $namespace -f

+-----------+-----------+-----------+
|   NAME    | FRAMES RX | FRAMES TX |
+-----------+-----------+-----------+
| eth3>eth2 |      1000 |      1000 |
| eth1>eth2 |      1000 |      1000 |
| eth1>eth3 |      1000 |      1000 |
| eth2>eth1 |         0 |      1000 |
| eth2>eth3 |      1000 |      1000 |
| eth3>eth1 |         0 |      1000 |
+-----------+-----------+-----------+
```

By inspecting [flow-r1-r2-r3.yaml](flows/flow-r1-r2-r3.yaml) you will see that
the traffic generator advertises a route for `198.51.100.0/24` but as shown in
[r1-config](topologies/ceos/r1-config) the switch `r1` drops all inbound traffic
for the neighbor `10.10.10.1` as part of the `PREFIX` route map.

Lets fix this by first removing the broken topology:

```bash
oc delete namespace $namespace
```

Inspect both configurations for [`r1`](topologies/ceos/r1-config) and
[`r2`](topologies/ceos/r2-config)

Then create a topology with a valid configuration
[r1-config-fixed](topologies/ceos/r1-config-fixed):

```bash
namespace=3-node-ceos-with-traffic-fixed
oc create namespace $namespace
# Fixes https://github.com/open-traffic-generator/ixia-c-operator/issues/18
# Fixes https://github.com/aristanetworks/arista-ceoslab-operator/issues/5
oc apply -f manifests/rbac/privileged-patch.yaml -n $namespace
tmp_dir=$(mktemp -d)
cp -r topologies/ $tmp_dir
echo "name: \"$namespace\"" >> $tmp_dir/topologies/3-node-ceos-with-traffic-fixed.pb.txt
kne create $tmp_dir/topologies/3-node-ceos-with-traffic-fixed.pb.txt --kubecfg $KUBECONFIG
```

Where:

* `$KUBECONFIG` - List of paths to configuration files used to configure access
  to a cluster

> Do not interrupt the `kne` command. It can take minutes until it returns. Just
> be patient and wait.

This time valid test results should be shown where the number of transmitted
frames is equal to the number of received frames:

```bash
oc create -f flows/job-flow-r1-r2-r3.yaml -n $namespace
oc get job -l flow=r1-r2-r3 -o name -n $namespace | xargs oc logs -n $namespace -f

+-----------+-----------+-----------+
|   NAME    | FRAMES TX | FRAMES RX |
+-----------+-----------+-----------+
| eth2>eth3 |      1000 |      1000 |
| eth3>eth1 |      1000 |      1000 |
| eth3>eth2 |      1000 |      1000 |
| eth1>eth2 |      1000 |      1000 |
| eth1>eth3 |      1000 |      1000 |
| eth2>eth1 |      1000 |      1000 |
+-----------+-----------+-----------+
```

If you are finished with testing delete the topology:

```bash
oc delete namespace $namespace
```

## Limitations

KNE itself is still under development and lacks some convenience features such
as deploying a topology into a specific namespace or taking into account the
current context set in the kubeconfig files. Additionally KNE deploys pods only,
which means the entire process described is vulnerable against disruptions such
as node failures even though there might be good reasons to not restart by using
more sophisticated approaches such as deployments since a failed instance could
indicate that a critical error caused the virtual instance to crash and in a
testing environment recovering automatically might be a bad idea.

The operators required to use different node types supplied by vendors such as
Arista or Keysight do not seem to work on OpenShift out of the box as resources
managed trough them still need additional privileges. Due to this limitation a
RBAC patch has to be applied between namespace and topology creation. Keep in
mind, that this was done to proof the point that it is possible to run KNE on
OCP. Before considering going into production reaching out to the vendors in
order to build a supported solution is highly recommended.

[kne]: https://github.com/openconfig/kne
[kne-docs]: https://github.com/openconfig/kne/blob/main/docs/setup.md
[okd-docs]: https://docs.okd.io/
[openshift-docs]: https://docs.openshift.com/
[okd-the-hard-way]: https://github.com/raballew/okd-the-hard-way
[metallb-docs]: https://metallb.universe.tf/installation/
[metallb-notes]:
    https://metallb.universe.tf/installation/clouds/#metallb-on-openshift-ocp
[keysight-elastic-network-generator]:
    https://www.keysight.com/de/de/products/network-test/protocol-load-test/keysight-elastic-network-generator.html
[keysight-support]: https://www.keysight.com/us/en/support.html
[arista-software-download]: https://www.arista.com/en/support/software-download
[arista-ceos]: https://www.arista.com/assets/data/pdf/cEOS_Solution_Brief.pdf
[ixia-c-otg]: https://github.com/open-traffic-generator/ixia-c
[otg]: https://github.com/open-traffic-generator
[global-pull-secret]:
    https://docs.openshift.com/container-platform/4.11/openshift_images/managing_images/using-image-pull-secrets.html#images-update-global-pull-secret_using-image-pull-secrets
