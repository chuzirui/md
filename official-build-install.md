Installing nsx-ujo components for k8s using nsx-ujo official/sandbox build
==========================================================================

This document describes the steps to install NCP as a
ReplicationController and nsx-node-agent as a DaemonSet using the NCP
Docker image and ymal templates in the nsx-ujo builds, and install CNI
plugin using the nsx-cni deb or rpm package.

The Kubernetes Service VIP/port and ServiceAccount token and ca\_file
are recommended to used for NCP and nsx-node-agent to connect and
authenticate to the k8s API Server.

Prerequisite
------------

1\. NSX and hypervisor deployment One way is to deploy
NSX-Crosshairs-OSHybridEdge topology from Jarvis. Then you will get a
NSX topology of a NSX Manager, a NSX Controller, two ESX hosts (managed
by a vCenter), two KVM hosts, and a NSX Edge node.

Due to changes in the NSX MP API, code from the master branch will not
work with NSX Crosshairs builds. To use nsx-ujo components with CH
builds, please use the crosshairs branch.

2\. Deploy VMs for k8s nodes NSX CNI plugin and nsx\_ujo package require
OVS (&gt;= 2.6.0), and Linux Kernel &gt;= 4.6.0). Some python components
(see nsx-ujo.git/requirements.txt) als need to be installed in the VM.

It is recommended to use the OVF deliverable from the k8snode build.
(<https://buildweb.eng.vmware.com/ob/?product=k8snode&branch>=) - which
has already kubeadm, kubectl, and all the required dependencies
installed. Access credentials are: nicira/nicira and root/ca\$hc0w.

NOTE: If using ESXi, these VMs must be deployed on ESXi 6.5 or above.

3\. Kubernetes cluster setup See k8s-kubeadm-install for setting up k8s
cluster with kubeadm.

4\. Minion node setup You will need to set up OVS bridge and connect the
PARENT VIF VNIC to a NSX LS. See the node\_setup for details. *Note*:
This step will need to be performed for the master node as well, if you
wish to deploy pods on it too.

5\. NSX resources initialization Refer to nsx-config.

NCP deployment
--------------

1\. Prepare NCP Docker image NCP Docker image is published by nsx-ujo
builds via url:
<http://build-squid.eng.vmware.com/build/mts/release/bora->&lt;build\_no&gt;/publish/nsx-ncp-ob-&lt;build\_no&gt;.tar
It is recommended to pull the image from our internal Docker registry
with url:
nsx-ujo-docker-local.artifactory.eng.vmware.com/nsx-ncp:ob-&lt;build\_no&gt;.

Unfortunately, NCP image pushing to the registry has not been integrated
into the nsx-ujo build process yet, so you might need to manually push
the NCP image of a specific build to the registry. See appendix for
details.

Of course, you could also choose to use your local Docker registry.

2\. Update NCP ReplicationController yaml template The yaml is published
via url:
<http://build-squid.eng.vmware.com/build/mts/release/bora->&lt;build\_no&gt;/publish/ncp-rc.yml
It creates the ncp.ini ConfigMap and NCP ReplicationController.

By Default the Kubernetes Service VIP/port and ServiceAccount token and
ca\_file are used for k8s API access. No change is required here. But
you need to fill in some NSX API parameters of ncp.ini.

*NOTE*: kube-scheduler by default will not schedule pods on the master
node. If you wish to run the NCP pod on the master node, you should
either: 1. Updates nodes unsetting the 'dedicated' taint annotation:

    kubectl taint nodes --all dedicated-

2.  Change the nodespec so the taint will be tolerated:

        scheduler.alpha.kubernetes.io/tolerations: '[
            {"key":"dedicated","value":"master","effect":"NoSchedule"}]'

3.  Add `nodeName: <master-node-name>` to the pod template spec
    in ncp-rc.yml.

You also need to update the container image name adding the prefix of
the image repository you will use. As an example using the internal
registry this will be:

    image: nsx-ujo-docker-local.artifactory.eng.vmware.com/nsx-ncp:ob-<build_id>

3.  Create NCP ReplicationController:

        kubectl apply -f ncp-rc.yml

nsx-node-agent deployment
-------------------------

The NSX node agent is a daemon set where each pod runs 2 containers:

-   nsx-node-agent, whose main responsability is to manage container
    network interfaces. It interacts with the CNI plugin and the
    Kubernetes API server. The Kubernetes API server interface will soon
    be replaced by Hyperbus
-   nsx-kube-proxy, whose only responsability is to implement Kubernetes
    service abstraction by translating cluster IPs into pod IPs. It
    implements the same functionality as the upstream kube-proxy, but is
    not mutually exclusive with it.

1\. Prepare NCP Docker image nsx-node-agent DaemonSet shares the same
Docker image with NCP. For DaemonSets you will not need to taint the
node for running an instance on the master node.

2\. Update nsx-node-agent DaemonSet yaml template The yaml is published
via url:
<http://build-squid.eng.vmware.com/build/mts/release/bora->&lt;build\_no&gt;/publish/nsx-node-agent-ds.yml
It creates the ncp.ini ConfigMap and nsx-node-agent DaemonSet.

Again by default the Kubernetes Service VIP/port and ServiceAccount
token and ca\_file are used for k8s API access. You could customize
other k8s options if needed.

NCP image url in the ReplicationController spec should be updated.

3.  Create nsx-node-agent DaemonSet:

        kubectl apply -f nsx-node-agent-ds.yml

CNI plugin installation
-----------------------

1\. nsx-cni deb/rpm installation Just install the nsx-cni deb or rpm
package in the minion node.
<http://build-squid.eng.vmware.com/build/mts/release/bora->&lt;build\_no&gt;/publish/nsx-cni-1.0.0.0.0.&lt;build\_no&gt;-1.x86\_64.rpm
<http://build-squid.eng.vmware.com/build/mts/release/bora->&lt;build\_no&gt;/publish/nsx-cni\_1.0.0.0.0.&lt;build\_no&gt;.deb

See nsx-ujo.git/installation/kubernetes/cni/dir\_tree.txt for files
installed.

2\. Update ncp.ini The file is installed in /etc/nsx-ujo/ncp.ini. You
need to fill in k8s API parameters.

Note the file is installed as a configuration file, so uninstall/upgrade
will keep the file.

3.  Resolve external python dependencies

Now nsx-cni deb/rpm does not include the python dependencies. If the
node VM does not have all the required python packages, you could
install them by (the command will also install vmware-nsxlib and a few
its dependencies which are not needed by CNI plugin):

    pip install -r /opt/vmware/nsx_ujo/requirements.txt

Please also note that in order for vmware-nsxlib installation to succeed
you will need to install the `libssl-dev` and `libffi-dev` packages.

Appendix
--------

1\. Internal Docker registry A shared account:
nsx-ujo-deployer/nsx-ujo-deployer could be used to log in the registry.
When pushing an NCP image to the registry, please follow the naming
convention of:
nsx-ujo-docker-local.artifactory.eng.vmware.com/nsx-ncp:ob-&lt;build\_no&gt;,
or:
nsx-ujo-docker-local.artifactory.eng.vmware.com/nsx-ncp:sb-&lt;build\_no&gt;
for sandbox builds.

<http://pa-dbc1116.eng.vmware.com/shenj/ujo/docker-push.sh> is a script
that downloads and pushes the NCP image to the registry given an
official build number.

You could browse
<https://build-artifactory.eng.vmware.com/artifactory/webapp/#/artifacts/browse/tree/General/nsx-ujo-docker-local>
to look at or delete existing images.

2\. Sandbox build You could run the following command in your nsx-ujo
workspace to start a sandbox build with your local commits:

    ./gobuild.sh nsx-ujo --localcommits

Note, the url of the published deliverables of a sandbox builds is a
little different form that of an official build. It is like:
<http://build-squid.eng.vmware.com/build/mts/release/sb-build-no/publish/nsx-ncp-sb-build-no.tar>

<http://build-squid.eng.vmware.com/build/mts/release/sb-&lt;build\_no&gt;/publish/ncp-rc.yml>
<http://build-squid.eng.vmware.com/build/mts/release/sb-&lt;build\_no&gt;/publish/nsx-node-agent-ds.yml>
<http://build-squid.eng.vmware.com/build/mts/release/sb-&lt;build\_no&gt;/publish/nsx-cni-1.0.0.0.0.&lt;build\_no&gt;-1.x86\_64.rpm>
<http://build-squid.eng.vmware.com/build/mts/release/sb-&lt;build\_no&gt;/publish/nsx-cni\_1.0.0.0.0.&lt;build\_no&gt;.deb>
