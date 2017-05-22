NCP Project Topologies
======================

Each NCP Project is implemented with a NSX topology that can be
summarized as follows:

> 1.  One or more logical switches, each one associated with a distinct
>     subnet, on which logical ports for containers are created.
> 2.  One Tier-1 logical router uplinked to a cluster-wide Tier-0
>     router. Every logical switch in the project is uplinked to this
>     Tier-1 router.
> 3.  A masquerade SNAT rule for each subnet in the project, applied to
>     the project's Tier-1 router. In addition No-NAT rules are created
>     from traffic originated from within the cluster and directed to
>     project subnets, and for traffic originated from cluster nodes.

Theory of operations
--------------------

### Project creation

When a project is created, NCP creates the Tier-1 router. Then it
creates an IP address pool, and a logical switch. The logical switch is
then uplinked to the Tier-1 router. The Tier-1 router is finally
uplinked to the cluster's Tier-0 router, specified in `ncp.ini`. Route
advertisement for connected and NAT routes is enabled on the Tier-1
router.

At the same time NAT rules for the first project subnet are created in
the Tier-1 router. These rules include:

-   A masquerade SNAT rule for outbound traffic;
-   One or more no-NAT rules for ingress traffic from other projects;
-   One or more no-NAT rules for ingress traffic from container hosts;

If SNAT must be configured for the project, NCP allocates a SNAT IP for
the project. This IP address is stored in the `ncp/snat-ip` tag applied
to the project's Tier-1 logical router.

NCP will by default configure SNAT for every project, unless explictly
told not to do so. When using the kubernetes adaptor, this is achieving
by adding a no-nat annotation to kubernetes namespaces. No NAT
configuration will be performed for namespaces annotated with:

    ncp-nsx/no-snat: True

### Container (Pod) operations

Every time a container is created, NCP finds the first of the project
logical switches with available port capacity, and creates a logical
port on that logical switch. It then annotates the kubernetes node with
the port data so that the CNI plugin can configure the container network
interface.

*NOTE: The annotation mechanism will be eventually replaced by Hyperbus.
Therefore at some point NCP will stop annotating Kubernetes nodes.*

Eventually IP pools associated with the switches will become exhausted.
When this happen, NCP will create an additional logical switch for the
project, and attach it to the tier-1 router, and configure the relevant
NAT rules.

When a container is deleted, NCP will destroy the container's logical
port. So far there is not plan for recycling unused logical switches
when usage goes down.

### IP Pool allocation

Project IP pools are managed by NCP. Users are not supposed to modify,
or delete these pools. IP Pools are allocated from one or more IP
blocks. In order for NCP to consider these blocks they have to be
appropriately tagged in NSX. For pods and containers this tag should be
the cluster name.

### External IP pool

This pool is used by a cluster to define ingress IP addresses and SNAT
IP addresses. Each project in the cluster gets a distinct SNAT IP from
an external pool.

If no external IP pool is available, NCP will look for an external IP
block, from which will create an external IP pool that will be reused.

*NOTE: This leads in theory to racy behavior. However, since events are
processed serially this does not constitute a threat for the correctness
of NCP operations. In other words, the current code does not allow for
conditions where several pool are concurrently created.*

Usage
-----

### NAT Topologies

In order to use NAT Topologies, it is necessary to specify an IP block
or IP pool to be used for external addresses. To this aim either an IP
block or an IP pool should be tagged with `ncp/external`, with value
`true`.

Priority is given to tags on IP pools. IP pools are however cluster
specific and therefore must also have a `ncp/cluster` tag whose value
matches the name of the cluster managed by the NCP instance.

When no pool is found, NCP will generate an IP subnet (and pool) from
the external IP block and set the `ncp/external` and `ncp/cluster` tags
on the newly created IP pool.

### IP Blocks

NCP allows for specifying one or more IP blocks to be used for
allocating IP pools for project's subnets. NCP will allocate subnets
from IP blocks tagged with the name of the cluster managed by the NCP
instance. Subnets can be allocated from any block with available space.
