NSX CNI plugin
==============

The nsx.py script partially implements the Container Network Interface,
providing a plugin for its 'main' interface - i.e.: the one responsible
for setting up and tearing down networking for a plugin.

There is no planned integration with CNI IPAM plugins, as both IP and
MAC allocation information are provided by the NSX platform.

Plugin installation
-------------------

Regardless of how the plugin is installed (setup.py or packages), the
plugin will be available in `/opt/cni/bin` However if kubelet was
configured with a different CNI network, the plugin should be moved to
that directory.

The network configuration file should be stored in the following path:

`` ` /etc/cni/net.d ``\`

The NSX integration uses the network configuration file only as a mean
to locate the CNI plugin and identify the integration bridge where
interfaces will be attached.

Other information, such as network CIDR, IP allocation strategy, network
gateway and masquerade behaviour are ignored. More information on CNI
network configuration can be found in the [CNI github repo]
(https://github.com/containernetworking/cni).

Adding or removing a container
------------------------------

The plugin is very simple, as it relies on the NSX node agent (TODO:
link to NSX node agent doc). When creating a container it sends a
request to create an interface in a given namespace for a given pod, and
plug it into the node OVS bridge. The result is then returned to the
kubelet. When destroying a container it sends a request to the NSX node
agent to remove the interface (and the OVS port) for a given container.

### CNI loopback plugin \[*Only for k8s 1.4*\]

This step is required for using CNI plugin in Kubernetes 1.4 only.

    $ wget https://github.com/containernetworking/cni/releases/download/v0.4.0/cni-amd64-v0.4.0.tgz
    $ sudo tar -xvf cni-amd64-v0.4.0.tgz -C /tmp
    $ sudo cp /tmp/loopback /opt/cni/bin
