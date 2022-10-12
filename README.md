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
workings of KNE I recommend using the projects source code as reference since
the official documentation is a bit lacking.

This document will guide you through setting up KNE on [OpenShift
(OCP)][openshift-docs] (or on [OKD][okd-docs] respectively) in the first step.
Then you will use the KNE cluster to create and interact with your own topology
based on Arista cEOS. In the last step you will set up a minimal workflow using
a simple job and virtual instances to pre-validate hardware tests. This should
give you a good understanding of what virtual testing is all about and allow you
to adopt this concept to more complex workflow leveraging tools such as
[Tekton][tekton] or GitHub actions.

If you are struggling to setup an OKD cluster, you might want to follow [this
guide][okd-the-hard-way] to learn it the hard way.

> With a few minor changes all steps mentioned in this guide are also usable on
> other Kubernetes distributions.

## Prerequisites

The following system specifications are required for the cluster if you plan to
follow the guide and run some additional workload:

* x86_64 cluster system architecture
* Dynamic storage provisioning configured
* At least 3 worker nodes
* At least 8 CPU cores per worker node
* At least 16 GB of RAM per worker node
* At least 1 GBit/s network interfaces on all nodes
* Cluster administrator permissions
* Internet access

> If the cluster worker nodes are hosted on a hypervisor, make sure to
> pass-trough the CPU information as features such as SSSE3 are required to run
> traffic generation successfully.

You also will be able to use a smaller setup but it could significantly impact
performance.

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
> access to the virtual environment is required, this step can be skipped.

Follow the instructions in the [official MetalLB documentation][metallb-docs]
and its [notes for OCP and OKD][metallb-notes].

## Getting cEOS image

Arista requires its users to register with
[arista.com][arista-software-download] before downloading any images.

> Make sure to register as with the user role Partner or Customer (do not use
> Guest role) because otherwise you might not be able to download the required
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
nodes. For cEOS Arista provides a controller.

Deploy the Arista controller:

```bash
oc apply -f https://raw.githubusercontent.com/aristanetworks/arista-ceoslab-operator/v1.0.2/config/kustomized/manifest.yaml
```

> Make sure the correct `image` is set for `ARISTA_CEOS` nodes in
> [3-node-ceos.pb.txt](/topologies/3-node-ceos.pb.txt) if you use a different
> version of cEOS.

Deploy the topology into a new namespace:

```bash
namespace=3-node-ceos
oc create namespace $namespace
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

Verify that the virtual instances are working:

```bash
oc exec -it -n $namespace r1 -- Cli -c "show bgp statistics"
oc exec -it -n $namespace r2 -- Cli -c "show bgp statistics"
oc exec -it -n $namespace r3 -- Cli -c "show bgp statistics"
```

> If you check the output you will realize that BGP does not seem to work
> properly, as each virtual instance shows that `2 neighbors are in Connect
> state`. This was done on purpose and will be fixed in one of the next steps.

You can also access each virtual instance by using their external or cluster IP
address. Additionally you could use the Kubernetes DNS service to access each
service via its DNS A record (`<svc>.<namespace>.svc.cluster.local`).:

```bash
oc get services -n $namespace
service-r1                     LoadBalancer   172.30.203.182   <REDACTED>    443:31420/TCP,22:32358/TCP,6030:31958/TCP   29s
service-r2                     LoadBalancer   172.30.23.251    <REDACTED>    22:32103/TCP,6030:32362/TCP,443:32467/TCP   28s
service-r3                     LoadBalancer   172.30.61.134    <REDACTED>     443:31661/TCP,22:32477/TCP,6030:32122/TCP   28s
```

Delete the topology:

```bash
oc delete namespace $namespace
```

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
> the cluster. The latest upstream version of this configuration is published on
> the [ixia-c-operator
> releases](https://github.com/open-traffic-generator/ixia-c/releases/) page.

Configure the usage of publicly available container images:

```bash
oc apply -f manifests/ixiatg/config.yaml
```

[3-node-ceos-with-traffic.pb.txt](/topologies/3-node-ceos-with-traffic.pb.txt)
uses the same misconfigured topology as
[3-node-ceos.pb.txt](/topologies/3-node-ceos.pb.txt) but enhanced it with
services that allow traffic generation. Deploy this topology into a new
namespace:

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

> Do not interrupt the `kne` command. It can take minutes until it returns. Just
> be patient and wait.

Now a topology consisting of several switches and a reference implementation of
the [Open Traffic Generator
API](https://github.com/open-traffic-generator/models) has been deployed with
the [IXIA-C traffic
generator](https://github.com/open-traffic-generator/ixia-c). In order to
generate traffic usually test scripts are written in
[snappi](https://github.com/open-traffic-generator/snappi) and executed against
the IXIA-C service (`*-otg-controller` services as shown above). Another, more
simplistic way of running tests is by using the [Open Traffic Generator CLI
Tool](https://github.com/open-traffic-generator/otgen) which will be used for
this guide.

In order to validate the functionality of the traffic generator deploy a job
that triggers a workflow where traffic flows trough a direct link between two
ports (`eth4` and `eth5`) on the traffic generator:

```bash
oc create -f flows/job-flow-otg-otg.yaml -n $namespace
```

By inspecting the logs, for each flow `eth4>eth5` and `eth5>eth4` the number of
frames received is equal to the number of frames sent. This indicates that this
particular connection works fine and the traffic generator is operational.

```bash
oc get job -l flow=otg-otg -o name | xargs oc logs -f
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   805  100   805    0     0   3120      0 --:--:-- --:--:-- --:--:--  3120
time="2022-10-11T19:57:53Z" level=info msg="Applying OTG config..."
time="2022-10-11T19:57:53Z" level=info msg=ready.
time="2022-10-11T19:57:53Z" level=info msg="Starting traffic..."
time="2022-10-11T19:57:53Z" level=info msg=started...
time="2022-10-11T19:57:53Z" level=info msg="Total packets to transmit: 1000, ETA is: 1s\n"
+-----------+-----------+-----------+
+-----------+-----------+-----------+
|   NAME    | FRAMES TX | FRAMES RX |
+-----------+-----------+-----------+
| eth4>eth5 |       500 |       500 |
| eth5>eth4 |       500 |       500 |
+-----------+-----------+-----------+

time="2022-10-11T19:57:56Z" level=info msg=stopped.
```

Lets try something more complex and see if our initial assumption, that the
topology of the virtual switch instances is broken. This can be done by running
a flow that tries to send traffic from ports `eth1` (connected to virtual
instance `r1`), `eth2` (connected to virtual instance `r2`) and `eth3`
(connected to virtual instance `r3`) to all other ports. In theory, if
everything is configured properly, this should create a similar output as in the
previous flow but we already know that something is broken.

```bash
oc create -f flows/job-flow-r1-r2-r3.yaml -n $namespace
```

When inspecting the logs confirms our assumption because no frames are seen on
receiving ends.

```bash
oc get job -l flow=r1-r2-r3 -o name | xargs oc logs -f
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  2344  100  2344    0     0  66971      0 --:--:-- --:--:-- --:--:-- 68941
time="2022-10-11T20:22:12Z" level=info msg="Applying OTG config..."
time="2022-10-11T20:22:53Z" level=info msg=ready.
time="2022-10-11T20:22:53Z" level=info msg="Starting traffic..."
time="2022-10-11T20:22:53Z" level=info msg=started...
time="2022-10-11T20:22:53Z" level=info msg="Total packets to transmit: 3000, ETA is: 1s\n"
+-----------+-----------+-----------+
+-----------+-----------+-----------+
|   NAME    | FRAMES TX | FRAMES RX |
+-----------+-----------+-----------+
| eth1>eth2 |       500 |         0 |
| eth1>eth3 |       500 |         0 |
| eth2>eth1 |       500 |         0 |
| eth2>eth3 |       500 |         0 |
| eth3>eth1 |       500 |         0 |
| eth3>eth2 |       500 |         0 |
+-----------+-----------+-----------+

time="2022-10-11T20:22:59Z" level=info msg=stopped.
```

Lets fix this by pushing a valid configuration to each virtual instance:

```bash
kne topology push $tmp_dir/topologies/3-node-ceos-with-traffic.pb.txt r1 $tmp_dir/topologies/ceos/r1-config-fixed --kubecfg $KUBECONFIG
kne topology push $tmp_dir/topologies/3-node-ceos-with-traffic.pb.txt r2 $tmp_dir/topologies/ceos/r2-config-fixed --kubecfg $KUBECONFIG
kne topology push $tmp_dir/topologies/3-node-ceos-with-traffic.pb.txt r3 $tmp_dir/topologies/ceos/r3-config-fixed --kubecfg $KUBECONFIG
```

Where:

* `$KUBECONFIG` - List of paths to configuration files used to configure access
  to a cluster

> Do not interrupt the `kne` commands. It can take minutes until they return.
> Just be patient and wait.

Then wait for a few minutes as BGP requires some time to configure itself,
clean-up the previous jobs and rerun the test. This time valid results should be
shown where the number of transmitted frames is equal to the number of received
frames:

```bash
sleep 2m
oc delete jobs --all -n $namespace
oc create -f flows/job-flow-r1-r2-r3.yaml -n $namespace
oc get job -l flow=r1-r2-r3 -o name | xargs oc logs -f
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  2342  100  2342    0     0   9840      0 --:--:-- --:--:-- --:--:--  9881
time="2022-10-11T20:45:56Z" level=info msg="Applying OTG config..."
time="2022-10-11T20:46:01Z" level=info msg=ready.
time="2022-10-11T20:46:01Z" level=info msg="Starting traffic..."
time="2022-10-11T20:46:01Z" level=info msg=started...
time="2022-10-11T20:46:01Z" level=info msg="Total packets to transmit: 3000, ETA is: 0s\n"
+-----------+-----------+-----------+
+-----------+-----------+-----------+
|   NAME    | FRAMES TX | FRAMES RX |
+-----------+-----------+-----------+
| eth1>eth3 |       500 |       500 |
| eth2>eth1 |       500 |       500 |
| eth2>eth3 |       500 |       500 |
| eth3>eth1 |       500 |       500 |
| eth3>eth2 |       500 |       500 |
| eth1>eth2 |       500 |       500 |
+-----------+-----------+-----------+

time="2022-10-11T20:46:04Z" level=info msg=stopped.
```

If you are finished with testing, as a last step delete the topology:

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

IXIA traffic generation as described in the [KNE examples][kne-examples] does
not seem to work on OpenShift out of the box as IXIA requires additional
privileges and seem to have problems handling arbitrary UID which are enforced
by OpenShift. Due to this, containers images have been rebuilt and a bunch of
patches have been applied to overcome these shortcomings. Keep in mind, that
this was only done to proof the point that it is possible to run KNE on OCP and
before considering going into production reaching out to the vendors in order to
build a supported solution should be the first thing to do.

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
