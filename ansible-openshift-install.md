OpenShift NSX Environment Install Automation
============================================

1. Create the OpenShift base VMs
--------------------------------

To deploy a OpenShift Test Cluster, and test the OpenShift NSX
integration, the first thing you need is a set of VMs running either
CentOS 7.2 (or latest), Fedora (latest) or RHEL 7.2. Usually those will
be 3 VMs, one OpenShift Master, and two OpenShift Nodes.

If you need a base CentOS 7.2 OVA for testing, please contact
<yfauser@vmware.com>.

2. Edit you hosts file with your details
----------------------------------------

Edit the hosts file in /openshift-ansible-nsx/hosts

In the \[all-nodes\], \[masters\], and \[nodes\] sections add the right
host names and ip addresses of your VMs you want to provision

    [all-nodes]
    openshift-master      ansible_ssh_host=10.114.209.74
    openshift-node1   ansible_ssh_host=10.114.209.75
    openshift-node2   ansible_ssh_host=10.114.209.76

    [masters]
    openshift-master      ansible_ssh_host=10.114.209.74

    [nodes]
    openshift-node1   ansible_ssh_host=10.114.209.75
    openshift-node2   ansible_ssh_host=10.114.209.76

------------------------------------------------------------------------

Next edit the \[all-nodes:vars\], \[masters:vars\], \[nodes:vars\] and
\[OSEv3:vars\] sections with your username/password details:

    [all-nodes:vars]
    ansible_ssh_user=localadmin
    ansible_ssh_pass=VMware1!
    ansible_sudo_pass=VMware1!
    ansible_ssh_user=localadmin
    ansible_become=true

    ... SNIP ...

    [nodes:vars]
    ansible_ssh_user=localadmin
    ansible_ssh_pass=VMware1!
    ansible_sudo_pass=VMware1!
    ansible_ssh_user=localadmin

    ... SNIP ...

    [masters:vars]
    ansible_ssh_user=localadmin
    ansible_ssh_pass=VMware1!
    ansible_sudo_pass=VMware1!
    ansible_ssh_user=localadmin

    ... SNIP ...

    [OSEv3:vars]
    # SSH user, this user should allow ssh based auth without requiring a password
    ansible_ssh_user=localadmin
    ansible_ssh_pass=VMware1!

Alternative to the above you could also configure ssh key based access
to your test VMs

------------------------------------------------------------------------

In the \[all-nodes:vars\] section you will need to fill in the details
of your NSX deployment. These will get populated into the ncp.ini file
of the K8s NSX integration

    [all-nodes:vars]
    ... SNIP ...

    # All variables used for NSX are bellow
    git_user='yfauser'
    nsx_manager='10.114.209.212'
    nsx_user='admin'
    nsx_password='VMware1!'
    transport_zone='12befb2b-239f-4c67-9959-ee1cdd216e46'
    tier_0_router='1fe9e60f-1286-4706-8637-8267f42777af'
    kube_master='10.114.209.74'
    kube_master_port='8443'
    ovs_path='/Users/yfauser/Downloads/ISOs'
    ovs_rpm='openvswitch-2.6.0.4469839-1.x86_64.rpm'
    ovs_kmod_rpm1='openvswitch-kmod-2.6.0.4469839-1.el7.x86_64.rpm'
    ovs_kmod_rpm2='kmod-openvswitch-2.6.0.4467838-1.el7.x86_64.rpm'

git\_user

:   your username for gitreview.eng.vmware.com

nsx\_manager

:   The IP Address of your NSX Manager. This will be entered into the
    ncp.ini file of nsx-ujo

nsx\_user

:   The username on NSX Manager

nsx\_password

:   The password of the user specified to access NSX Manager

transport\_zone

:   The UUID for the Transport Zone in use on NSX

tier\_0\_router

:   The UUID for the default Tier-0 router to uplink the dynamically
    created T1 Routers to

kube\_master

:   The IP Address of the OpenShift Master

kube\_master\_port

:   The tcp port on which the k8s API is listening to on the OpenShift
    Master

ovs\_path

:   This is the path on your local machine (where the Ansible
    plabook runs) that has the OVS RPMs

ovs\_rpm

:   This is the RPM file for the OVS, e.g.
    openvswitch-2.6.0.4469839-1.x86\_64.rpm

ovs\_kmod\_rpm1

:   This is the first of two RPMs needed for the OVS Kernel Module, e.g.
    openvswitch-kmod-2.6.0.4469839-1.el7.x86\_64.rpm

ovs\_kmod\_rpm2

:   This is the second of two RPMs needed for the OVS Kernel
    Module, e.g. kmod-openvswitch-2.6.0.4467838-1.el7.x86\_64.rpm

**NOTE**: The latest RPMs for OVS can be found at
<https://buildweb.eng.vmware.com/ob/?&product=nsx-ovs-build&branch=master>.

------------------------------------------------------------------------

There is one NSX specific configuration in the \[nodes:vars\] section
that you will need to make

    [nodes:vars]
    ... SNIP ...

    # All Node variables used for NSX are bellow
    uplink_port='eno33557248'

uplink\_port

:   The local name of the uplink port vnic on the Node VMs, this will
    usually be 'eno33557248' for RedHat based systems

------------------------------------------------------------------------

The \[OSEv3:vars\] holds more details for the OpenShift deployment that
you need to fill out. You can find all the supported variables in the
OpenShift Origin Documentation for the Advanced Installation at
<https://docs.openshift.org/latest/install_config/install/advanced_install.html>.

    # Set the default route fqdn
    openshift_master_default_subdomain=apps.yves.local

    os_sdn_network_plugin_name=cni
    openshift_use_openshift_sdn=false
    openshift_node_sdn_mtu=1500

    # If ansible_ssh_user is not root, ansible_become must be set to true
    ansible_become=true

    deployment_type=origin

    # uncomment the following to enable htpasswd authentication; defaults to DenyAllPasswordIdentityProvider
    openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true', 'kind': 'HTPasswdPasswordIdentityProvider', 'filename': '/etc/origin/master/htpasswd'}]
    openshift_master_htpasswd_file=<enter_full_path_here>/openshift-ansible-nsx/files/htpasswd

openshift\_master\_default\_subdomain

:   This is the default subdomain used in the OpenShift routes for
    External LB

os\_sdn\_network\_plugin\_name

:   Needs to be set to 'cni' for the NSX Integration

openshift\_use\_openshift\_sdn

:   This disables the built-in OpenShift SDN solution

deployment\_type

:   This is set to origin for the open source version of OpenShift

openshift\_master\_htpasswd\_file

:   This file is holding the htpasswd password file. You will need to
    fix the path to it for the deployment to work, so exchange
    &lt;enter\_full\_path\_here&gt; with your 'real path'

3. Run the preparation for all hosts and for the Nodes
------------------------------------------------------

First check that you have connectivity to all hosts and that python is
installed and at the right version:

    ansible all-nodes -i hosts -m ping

The result should look similar to this, if not, troubleshoot the
connectivity problems first:

    openshift-node1 | SUCCESS => {
        "changed": false,
        "ping": "pong"
    }
    openshift-master | SUCCESS => {
        "changed": false,
        "ping": "pong"
    }
    openshift-node2 | SUCCESS => {
        "changed": false,
        "ping": "pong"
    }

In the openshift-ansible-nsx directory now run the prepare-os-nodes.yml
Playbook:

    ansible-playbook -i hosts prepare-os-nodes.yml

This playbook will reboot the nodes at the end of the playbook to
activate the OVS 2.6 Kernel Modules.

After the Nodes are back up, you can run the nodes-cni-install.yml
playbook:

    ansible-playbook -i hosts nodes-cni-install.yml

4. Install OpenShift Origin
---------------------------

Now that all preparation steps are done, we can install OpenShift
Origin. For this, change to the OpenShift origin Ansible repo. You can
get it from:

    git clone https://github.com/openshift/openshift-ansible.git

Now run the playbooks/byo/config.yml Playbook, referencing to the hosts
file we created earlier

    ansible-playbook -i ../../gitreview/nsx-ujo/mock/openshift-ansible-nsx/hosts playbooks/byo/config.yml

This process will take quite a while to complete

5. Post OpenShift Deployment Steps
----------------------------------

Before continuing to install the NCP and NKP Service we will need to do
some pre-req steps.

-   Enable 'open access' to the k8s API on the master.

**NOTE** this step will not be needed at a later stage of the project
when NCP and the CNI Plugin are able to authenticate using a service
account

    oc login -u system:admin -n default
    oadm policy add-cluster-role-to-user cluster-admin system:anonymous

-   Find out the VIF Id for the nodes and annotate the Nodes with the
    VIF Id:

<!-- -->

    kubectl annotate node openshift-node1.yves.local vif='5005d6f1-d4e7-4b38-a914-5e5924bae8a0'
    kubectl annotate node openshift-node2.yves.local vif='3ac72b99-ffcb-4376-8547-e9a32ecb721c'

6. Install NCP on the Master
----------------------------

Right now NCP (aka NCM) can be installed on the Master as a systemd unit
File by running the Playbook master-ncp-install.yml

    ansible-playbook -i hosts master-ncp-install.yml

In later version we will mode to a Container / Pod type of deployment
for the NCP.

The above will install NCP as a systemd unit, so that you can use
systemctl and journalctl to work with it.

    systemctl status/restart/stop/start ncp

    journalctl -f -u ncp

7. Add the NSX Kube Proxy Service
---------------------------------

To install the NSX Kube Proxy Service as a systemd unit File you can
run:

    ansible-playbook -i hosts nodes-nsx-kube-proxy-install.yml

The above will install NSX Kube Proxy as a systemd unit, so that you can
use systemctl and journalctl to work with it.

    systemctl status/restart/stop/start nkp

    journalctl -f -u nkp

8. Fix Docker Networking & Registry
-----------------------------------

There's a missing IPtables firewall rule to allow DNS requests from the
Docker default bridge containers to the dnsmasq process on the Node. It
needs to be opened manually (other options might exist and will be
discussed with the RedHat OpenShift Team)

    vim /etc/sysconfig/iptables

add the following Rules at the bottom of the file before COMMIT

    ...
    -A OS_FIREWALL_ALLOW -p tcp -m state --state NEW -m tcp --dport 53 -j ACCEPT
    -A OS_FIREWALL_ALLOW -p udp -m state --state NEW -m udp --dport 53 -j ACCEPT
    COMMIT

and then restart iptables, docker and origin-node (restarts kube-proxy
and kubelet)

    systemctl restart iptables
    systemctl restart docker
    systemctl restart origin-node

Also, the internal docker registry of OpenShift needs to be allowed to
use non-TLS for OpenShift to work. Normaly this should be added
automatically by the OpenShift Ansible installer, but it seems that this
is currently not working

    vi /etc/sysconfig/docker

And add

    INSECURE_REGISTRY='--insecure-registry 172.30.0.0/16'

And restart docker

    systemctl restart docker

9. Add routes to the POD IP Subnets
-----------------------------------

The Nodes will need access to the PODs e.g. for Kubelet health-checks
and so that the PODs can access the K8s and OpenShift API as well as
DNS, etc. on the Master. Therefore we need to add static routes pointing
to the Tier-0 router (if the underlying network doesn't route the
Networks to the POD Address space):

    ip route add 10.1.0.0/24 via 10.114.209.89
    ip route add 10.2.0.0/24 via 10.114.209.89
    ip route add 10.3.0.0/24 via 10.114.209.89
    ... etc

10. Various Notes
-----------------

-   Allow root access in PODs/Containers (only for testing). The bellow
    will root access in all PODs of the oc project ayou are currently
    logged in to

<!-- -->

    oc new-project test-project
    oc project test-project
    oadm policy add-scc-to-user anyuid -z default

-   Configure (add) the OpenShift Registry

<!-- -->

    oc login -u system:admin -n default
    oadm registry --service-account=registry --config=/etc/origin/master/admin.kubeconfig

-   Delete the OpenShift Registry

<!-- -->

    oc login -u system:admin -n default
    oc delete svc/docker-registry dc/docker-registry

-   Configure (add) the OpenShift routers (HA-Proxy N/S LBs)

<!-- -->

    oc login -u system:admin -n default
    oadm router router --replicas=2 --service-account=router

-   Delete the create routers

<!-- -->

    oc login -u system:admin -n default
    oc delete svc/router dc/router

-   Create a sample Ruby based 2 tier apps

<!-- -->

    oc login -u system:admin -n default
    oc new-project nsx
    oc process -n openshift mysql-ephemeral -v DATABASE_SERVICE_NAME=database | oc create -f -
    oc new-app centos/ruby-22-centos7~https://github.com/openshift/ruby-hello-world.git
    oc expose service ruby-hello-world
    oc env dc database --list | oc env dc ruby-hello-world -e -
