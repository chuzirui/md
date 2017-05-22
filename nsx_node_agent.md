NSX Node Agent
==============

NSX Node Agent is used to configure container network namespace and
interface. HyperBus (the daemon running in NSX hypervisor) uses nsx-rpc
to send container network information to the agent [^1] . The agent also
has a local socket for NSX CNI plugin to get container network
information.

Configuration
-------------

NSX Node Agent reads the configuration file with the following setting:

    [nsx_node_agent]

proc\_mount\_path\_prefix = /host nsxrpc\_cip = 169.254.1.2/16
nsxrpc\_port = 2345 vlan = 4094 enable\_hyperbus = False

"proc\_mount\_path\_prefix" is the prefix of path for host /proc to
mount in agent container. Set to '/' if agent runs as a daemon.
"enable\_hyperbus" is the option to use HyperBus to get pod
configuration. If set it False, agent uses kubernetes annotation to get
information. "nsxrpc\_cip":"nsxrpc\_port" is the endpoint for HyperBus
to connect with nsx\_node\_agent. The agent creates an internal port on
OVS bridge with "nsxrpc\_cip" and "vlan" tag. Note: "nsxrpc\_cip",
"nsxrpc\_port" and "vlan" should be the same setting with HyperBus
configuration file [^2] .

Installation
------------

a\) Run nsx\_node\_agent as a daemon in container host VM. Install nsxrpc
python library first. \$ cd installation/nsx-rpc \$ tar xvf
vmware-nsxrpc-client-1.0.tar.gz \$ cd vmware-nsxrpc-client-1.0 \$ pip
install .

Then install nsx\_node\_agent. \$ cd nsx-ujo \$ pip install .

b\) Run nsx\_node\_agent in container as a DaemonSet. Check
installation/kubernetes/nsx-node-agent-ds.yml.in for example.

Workflow
--------

When NSX Node Agent starts, there are two threads to listen connections.
It starts local socket to accept CNI connection, and get container 'ADD'
or 'DEL' operations from CNI. Then do container network configuration or
deconfiguration. The agent also creates a OVS port to get HyperBus
connection. Then it will establish a session and receive container
configuration from HyperBus.

[^1]: <http://url.eng.vmware.com/3w2c>

[^2]: <https://opengrok.eng.vmware.com/source/xref/nsx.git/controller/lcp/hyperbus/src/configuration/hyperbus.xml>
