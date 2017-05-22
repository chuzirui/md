NCP: Network Policies
=====================

Network policies are used to control the communication between pods. It
defines whitelist rules which allow traffic from selected ports and
protocols to a group of pods identified by their labels.

Network policies are supported in kubernetes as beta extensions in
v1.4/1.5. Network policies in kubernetes v1.3 were supported as alpha
extensions and thereby not supported for k8s v1.3.

Network policies are processed only for those namespaces whose isolation
value is set to `DefaultDeny`. Setting namespace isolation drops all
ingress traffic on pods unless they are whitelisted by a network policy.
Once namespace isolation is turned OFF, all ingress traffic is
whitelisted and network policies are ignored.

Configuration
-------------

Network policy extension must be enabled as a runtime config while
starting kubernetes apiserver:

    --runtime-config=extensions/v1beta1/networkpolicies=true

Set `watch_k8s_network_policies=True` in `ncp.ini` to start network
policy watcher. This value is True by default.

Isolation for namespaces must be turned ON for network policy to be
processed. Isolation can be turned as follows:

    kubectl annotate ns <namespace> "net.beta.kubernetes.io/network-policy={\"ingress\": {\"isolation\": \"DefaultDeny\"}}"

One thing to note here is that if the isolation property for namespace
is set to any value other than DefaultDeny, network policies will not be
processed and instead ignored.

Workflows
---------

### Namespace isolation ON/OFF events

Namespace isolation can be turned ON or OFF for a namespace with help of
annotations. When the namespace is isolated, network policies for that
namespace will be processed, otherwise they will be ignored. Namespace
isolation is enforced via firewall sections in the backend. Once
namespace is isolated for the first time, a corresponding firewall
section will be created in the backend with two rules. The first rule
will allow EGRESS traffic for all pods belonging to that namespace. The
second rule will DENY all ingress traffic for all pods belonging to this
namespace. Once the isolation property of namespace is turned OFF, the
ingress and egress rule for that isolation firewall section will be
deleted. However the isolation firewall section will not be deleted.
Resetting the isolation property of the namespace to ON will now lead to
creation of ingress and egress rules only.

Once a namespace is no longer isolated, the network policies applied to
that namespace should have no effect. This is taken care of by disabling
all the firewall rules belonging to all network policies under that
namespace. On the other hand, once the namespace isolation property to
ON, all previously disabled rules are enabled within that namespace.

### Network policy creation

A network policy consists of a set of whitelist rules applied to pods
matching a specific label selector. Creation of a network policy, leads
to creation of a Firewall Section in the backend. For each rule, a
corresponding Firewall Rule is created under that section with the
action set to ALLOW. Each rule may consist of a set of protocols and
ports which should be whitelisted. Currently, kubernetes supports
TCP(default) and UDP protocols. For each of these protocol and port
pair, NCP creates a NSService of type L4PortSetNSService. A NSGroup is
created to group the source pods/projects and another NSGroup is created
to group the destination pods/projects. These NSGroups are created with
dynamic membership criterias based on the label selector specified in
the policy. The `applied_tos` and `destinations` field of the rule is
set to the destination NSGroup. The `sources` field of the rule is set
to source NSGroup. The `direction` field of the rule is always set to
IN.

### Network policy update

Network policy updates are not supported by kubernetes.

### Network policy delete

When a network policy is deleted, the firewall section associated with
this policy is deleted. This automatically deletes all the firewall
rules associated with this section. NCP also deletes the NSGroups
created for this network policy.

### Effect of pod events

Whenever a pod is added/modified with label which matches an existing
network policy within its namespace, it is automatically added to the
corresponding NSGroup due to the presence of dynamic membership
criterias. There is no additional event handling in this case.

### Effect of namespace events

Whenever a namespace is added/modified with label which matches an
existing network policy, it is automatically added to the corresponding
NSGroup due to the presence of dynamic membership criteria.

Deletion of namespace leads to deletion of all network policies
associated with this namespace and deletion of the network policy
watcher for that namespace.

*NOTE: Kubernetes does not expose a global API to retrieve and watch
network policies for all namespaces. Therefore NCP starts a distinct
watcher per namespace, to watch network policy events.*
