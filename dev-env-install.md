Creating a nsx-ujo development environment
==========================================

This guide provides a step by step description for creating a
development environment for nsx-ujo components (NCP, nsx\_kube\_proxy,
nsx\_node\_agent, and nsx CNI plugin). Such components will be deployed
straight from source code and will not be executed in containers.

If you are interested in setting up an environment where NCP components
run in containers please refer to the
official build install guide &lt;official-build-install&gt;.

Deploying NSX infrastructure
----------------------------

The preferred way for deploying development environments is by using
[Jarvis](https://vdnet.eng.vmware.com/jarvis) to deploy an instance of
the *NSX-Transformers-OSHybridEdge* topology. This topology, albeit
created for developing OpenStack integration works nicely for Container
Orchestrator integration as well.

In the editable parameters box select version 6.5 for vc and ESXi
(container networking is not supported in earlier versions). As for NSX
builds, use recommended builds unless you explicitly need to use
different build numbers.

**NOTE:** *Beware of instance leases!*

If you rather deploy a development environment in your own
infrastructure, consider using [Yves' ansible
scripts](https://github.com/yfauser/nsxt-ova-deployer). Please note that
this is a private repository so you may have to be granted access before
using it.

Preparing NSX management plane
------------------------------

The NSX configuration guide &lt;nsx-config&gt; provides plenty of
information for preparing the NSX management plane for us with container
orchestrators. In particular:

-   The *Resource Configuration* section will guide through the process
    of tagging NSX resources which will be consumed by NCP to configure
    logical network topologies and container networking.
-   The *Logical Networking* section provides a step-by-step guide to
    the initial network configuration required for kubernetes nodes.

Setting up the master node
--------------------------

First, do initial host preparation as specified in the
Node setup guide &lt;node\_setup&gt;. Use the host name *k8s-master*, or
something similar.

*NOTE*: It is advisable to leverage utilities such as screen or tmux to
manage the various nsx-ujo executables, or alternatively configure
systemd units for them.

At this stage the CNI plugin is installed and the CNI network is
configured. The kubernetes cluster can therefore be initialized with
kubeadm. If using kubeadm &gt;= 1.6.1 run the following command:

    kubeadm init --apiserver-advertise-address <node_ip> --skip-preflight-checks

Otherwise execute the following command:

    kubeadm init --api-advertise-addresses <node_ip> --skip-preflight-checks

The *&lt;node\_ip&gt;* above should be the IP address of the interface
on the NSX network. The process should last less than 60 seconds and
result in a basic kubernetes installation. Depending on which version of
kubeadm you are using, the resulting manifest might or might not enable
the insecure endpoint. Likewise the RBAC admission controller might or
might not be enabled. For more details setting up the cluster with
kubeadm, please refer to the
kubeadm setup guide &lt;k8s-kubeadm-install&gt;.

At the end of the process you should get a token to use when joining
other nodes. The master node should appear as `ready` and kubernetes
services such as the API server and the scheduler should be running as
pods in the kube-system namespaces.

While most of these service run in `HostNetwork` mode and do not invoke
the CNI plugin, the DNS pod will be stuck in `ContainerCreating` status
as the CNI plugin fails since neither NCP or the NSX node agent are
running. This is expected. These containers should be wired as soon as
the nsx-ujo entities are started.

*NOTE*: If kubeadm creates the kube-discovery deployment, it is
recommended to delete it as soon as possible. Its pods won't be
scheduled on master, and the scheduler will try to create another pod.
It not unlikely to end up in a situation with over 10,000 failed pods.

Configuring and launching NCP
-----------------------------

The first step is populating the configuration file. A sample is
available at `installation/kubernetes/ncp.ini.in`

1.  Copy ncp.ini from the sample file:

        mkdir -p /etc/nsx-ujo
        cp installation/kubernetes/ncp.ini.in /etc/nsx-ujo/ncp.ini

2.  Retrieve the token for default service account in the default
    namespace. In this guide we will use the default service account for
    simplicity, but any service account can be used:

        secret=`kubectl get serviceaccount default -o yaml | grep -A1 secrets | \
          tail -n1 | awk {'print $3'}`
        kubectl get secret $secret -o yaml | grep "token:" | awk {'print $2'} | \
          base64 -d > /etc/nsx-ujo/default_token

3.  Remove marker line `>>> Start of NCP only options` and 2 subsequent
    lines
4.  Set the following items in ncp.ini:

        [coe]
        cluster = cluster_name

        [k8s]
        api_server_host_ip = <k8s_api_server_ip>
        api_server_host_port = 6443 # or 8080 if using insecure endpoint
        client_token_file = /etc/nsx-ujo/default_token
        insecure = True

        [nsx_v3]
        nsx_api_managers = <nsx_manager_ip>
        nsx_api_cert_file = <path_to_nsx_secret>
        insecure = True

Please note that *k8s\_api\_server\_ip* should be 127.0.0.1 if using the
insecure endpoint, otherwise the address where the API server is
listening on https (by default 6443). The NSX certificate file can be
created using the utility in `sample/create_nsx_cert.py`. More
information for using this utility is available in the
NSX config guide &lt;nsx-config&gt;. It is however possible, even if not
recommended, to use admin credentials for NSX, populating the
`nsx_api_user` and `nsx_api_password` configuration parameters.

At this stage it is possible to start NCP:

    ncp --config-file /etc/nsx-ujo/ncp.ini

### Configuring and launching nsx\_kube\_proxy

This one is fairly easy! ncp.ini already contains all the information
the NSX kube proxy needs. Just launch it:

    nsx_kube_proxy --config-file /etc/nsx-ujo/ncp.ini

Optionally we can specify the OVS uplink OpenFlow port where to apply
the NAT rules to as follows:

    [nsx_kube_proxy]
    ovs_uplink_port = eth1

If not specified, the port that gets assigned ofport=1 is used.

### Configuring and launching nsx\_node\_agent

Edit ncp.ini as follows:

    [nsx_node_agent]
    proc_mount_path_prefix = /
    enable_hyperbus = False
    nsxrpc_cip = 169.254.1.2/16
    nsxrpc_port = 2345
    nsxrpc_vlan = 4094

We need to specify this as the default setting is meant for when the
process is executed in a container.

Then create a directory for the socket handle:

    mkdir -p /var/run/nsx-ujo

Install nsx-rpc library:

    cd installation/nsx-rpc
    tar xvf vmware-nsxrpc-client-1.0.tar.gz
    cd vmware-nsxrpc-client-1.0
    pip install .

And finally start the agent:

    nsx_node_agent --config-file /etc/nsx-ujo/ncp.ini

Setting up minion nodes
-----------------------

First, do initial host preparation as specified in the
Node setup guide &lt;node\_setup&gt;. Use the host name
*k8s-minion-&lt;N&gt;*, or something similar.

### Configuring and launching nsx\_kube\_proxy

Before launching the nsx\_kube\_proxy we need to make sure that
`ncp.ini` has been configured to access the Kubernetes API server.

*NOTE*: we must not use the API server cluster API because of [issue
17764673](https://bugzilla.eng.vmware.com/show_bug.cgi?id=1776473)

1.  Copy ncp.ini from the sample file:

        mkdir -p /etc/nsx-ujo
        cp installation/kubernetes/ncp.ini.in /etc/nsx-ujo/ncp.ini

2.  Copy the token file used on the master node into `/etc/nsx-ujo`
3.  Remove marker line `>>> Start of NCP only options` and 2 subsequent
    lines
4.  Set the following items in ncp.ini:

        [k8s]
        api_server_host_ip = <k8s_api_server_ip>
        api_server_host_port = 6443
        client_token_file = /etc/nsx-ujo/default_token
        insecure = True

Finally launch the NSX Kube Proxy:

    nsx_kube_proxy --config-file /etc/nsx-ujo/ncp.ini

### Configuring and launching nsx\_node\_agent

The steps are the same as for the master node.

### Joining the Kubernetes Cluster

In case the master token is lost, it can be easily recovered. If running
kubeadm 1.6.1 or above, on the master node simply type:

    kubeadm token list

otherwise:

    kubectl -n kube-system get secret clusterinfo -o yaml | grep token-map | \
    awk {'print $2'} | base64 -d | sed "s|{||g;s|}||g;s|:|.|g;s/\"//g;"

Setting up NCP CLI on the master node
-------------------------------------

NCP CLI will be auto-installed when NCP is deployed as a pod. When
running NCP as a process, NCP CLI needs to be installed manually.
Assuming nsx-ujo source tree is cloned under /home/nicira/ on the node.

1.  Install NCP if not done yet. For example,

    cd /home/nicira/ python setup.py build sudo python setup.py install

2.  Run setup script to install NCP CLI.

    /home/nicira/nsx-ujo/doc/source/devref/samples/ncp-cli-setup.sh

3.  Because the installed CLI framework has only pyo files, we need to
    run NCP with "python -O" in order to import those pyo libraries. For
    example, edit NCP startup file:

    vi /usr/local/bin/ncp and replace the first line

    :   \#!/usr/bin/python

    with

    :   \#!/usr/bin/python -O

    Note that this step is not required if the nsx-ujo package was built
    before installation as described in Step 1.

4.  Now run NCP process as root.

    sudo ncp

5.  On the client side, CLI also needs to run as root.

    sudo nsxcli


