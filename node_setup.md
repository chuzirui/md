Kubernetes Node Configuration
=============================

This guide assumes you are either using the ovf which comes as a
deliverable of the *k8snode* product
(<https://buildweb.eng.vmware.com/ob/?product=k8snode>) or the the OVF
available at
<http://pa-dbc1116.eng.vmware.com/shenj/ujo/ubuntu-16_04-k8s.ovf>. The
former is an Ubuntu 16.10 desktop instance, whereas the latter is an
Ubuntu 16.04 desktop instance.

VM settings
-----------

A second network interface must be added to the VM, and attached to the
node logical switch. More info in the
NSX Config guide &lt;nsx-config&gt;.

It is also recommended to use 2 vCPUs and at least 2048MB RAM. Please
make sure you are running OVS &gt;= 2.6.0 and Linux Kernel &gt;= 4.6.0
in order to ensure CT-NAT support is enabled as that is required by
`nsx_kube_proxy`.

Network Configuration
---------------------

1.  Ensure OVS is running:

        # service openvswitch-switch status

2.  Create br-int instance (if not already created):

        # ovs-vsctl add-br br-int

3.  Add the network interface attached to the node logical
    switch (node-if) to br-int:

        # ovs-vsctl add-port br-int <node-if> -- set Interface <node-if> ofport=1

4.  Ensure br-int and node-if link status is up:

        # ip link set br-int up
        # ip link set <node-if> up

5.  Configure ip addressing on br-int (never on node-if). You can either
    use the ip command or edit `/etc/network/interfaces`.
    `The IP address should be in the same subnet of the address used for configuring the router logical port for the node logical router`:

        # ip addr add <node-ip>/<prefix> dev br-int

6.  Configure a route for pod ips via br-int, otherwise traffic will go
    through the management interface which has the default route. The
    pod CIDR is the one specified when creating the IP block in NSX:

        # ip route add <pod-cidr> via <node-ip>

7.  If using a test external network for features such as Ingress or
    SNAT, also add a route for this network. For instance, its CIDR on
    Jarvis instances is 172.20.1.0/24:

        # ip route add <ext-cidr> via <node-ip>

8.  (Optional) Configure the host name for the node as below, and add it
    into /etc/hosts with the newly set host name.:

        # hostnamectl set-hostname <hostname>
        # echo <hostname> > /etc/hostname

9.  Install libffi-dev and libssl-dev. They are required for building
    pycripto, needed by vmware-nsxlib. This has to be done on all nodes,
    and not just on masters, because of the way we package nsx-ujo
    components for pip install. Depending on the OVF template you are
    using, these packages may have already been installed:

        # apt install -y libffi-dev libssl-dev

10. Clone nsx-ujo repository and install it in *editable mode* using pip
    (also look at pip installation gotchas below):

        $ git clone https://gitreview.eng.vmware.com/nsx-ujo
        $ cd nsx-ujo
        $ sudo pip install -r requirements.txt
        $ sudo pip install -e .

11. Make sure the CNI plugin has been install in /opt/cni/bin
12. Make sure the CNI network configuration has been configured in
    `/etc/cni/net.d/10-net.conf` and looks like the following:

        { "name": "net", "type": "nsx" }

Installation gotchas
--------------------

There could be version mismatches when installing with pip because the
same libraries may be required by different projects with conflicting
versions.

During our experience we incurred a few issues:

-   vmware-nsxlib requiring version of oslo libraries more updated than
    nsx-ujo. This could be solved in two ways: upgrade the libraries
    that do not stastify vmware-nsxlib requirements, eg.:
    `pip install --upgrade oslo_log`, or install vmware-nsxlib
    separately, cloning from git, and then remove the line for nsxlib
    from nsx-ujo's `requirements.txt` file
-   There could be an obscure conflict with pbr versions required
    by sqlalchemy-migrate. This could be solved by upgrading
    sqlalchemy-migrate. If you wonder why sqlalchemy-migrate is
    required, that is an indirect dependency coming from vmware-nsxlib.
    vmware-nsxlib -&gt; neutron\_lib -&gt; oslo\_db -&gt;
    sqlalchemy-migrate If you think there's no dark side to python
    think again.
-   installing in *editable mode* or with the *develop* command won'
    copy data files. This means you have to setup the CNI plugin and the
    CLI files manually. For the CNI plugin it is recommendable to use a
    symbolic link:
    `ln -s /<path_to_nsx_ujo>/nsx_ujo/bin/nsx /opt/cni/bin/`. Otherwise,
    you can avoid installing in editable mode, but have to remember to
    do `pip install .` before testing your changes.

Only if running Kubernetes 1.4, make sure the CNI loopback plugin is
installed:

    $ wget https://github.com/containernetworking/cni/releases/download/v0.4.0/cni-amd64-v0.4.0.tgz
    $ sudo tar -xvf cni-amd64-v0.4.0.tgz -C /tmp
    $ sudo cp /tmp/loopback /opt/cni/bin
