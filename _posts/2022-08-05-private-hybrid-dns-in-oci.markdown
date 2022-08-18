
![Intro Picture](/images/2022-08-05-private-hybrid-dns-in-oci/oder.jpg)

# __Why We Need Private Hybrid DNS?__

Name resolution is often overlooked topic in many data management projects. It is not a
problem when all the components are in a single Virtual Cloud Network (VCN), but it becomes
an issue when the solution consists of multiple interconnected VCNs. Or when part of the
solution, such as data sources or users, are outside of OCI.

Typically, integration, data warehousing and analytical component are deployed in OCI,
while the data sources are on-premises or in other VCNs in OCI. And users often access
analytical environments from on-premises networks, over private link such as VPN Connect
or FastConnect.

In order for these hybrid data management solutions to work, we need to resolve on-premises
hostnames in VCNs, and, vice versa, OCI hostnames in on-premises network. In other words,
we need __private hybrid DNS__.

Here are some scenarios in which OCI names must be resolvable in on-premises network:

* Analytics Cloud (OAC) users on-premises need to access OAC instance with Private Endpoint.
* Autonomous Database (ADB) administrators on-premises need to access ADB instance with Private Endpoint.
* ETL tool on-premises needs to insert or update data in Autonomous Data Warehouse (ADW) database.

And here are some scenarios in which on-premises names must be resolvable in OCI VCNs:

* OAC needs to access Oracle Database on-premises via Private Access Channel (PAC).
* OCI Data Integration needs to connect to SQL Server database on-premises.
* OCI Data Science notebook needs to access Hadoop cluster on-premises.


# __Alternatives to Private Hybrid DNS__

Before looking at private hybrid DNS, let's briefly mention the alternatives.


## Modify /etc/hosts

You can modify `/etc/hosts` file on your notebook and enter hostnames and private IP
addresses of OCI resources there.

This approach is possible for simple testing of the access, but is not feasible for larger
user base or for production systems. Also, many organizations prevent users from tinkering
with configuration files like `/etc/hosts` because of security and manageability.


## On-premises DNS Resolver

You can also decide to use on-premises private DNS to resolve OCI hostnames. In this case,
you would modify VCN's __DHCP Options__ and specify on-premises nameserver(s), so that OCI
instances will use on-premises nameserver instead of default OCI resolver available on
`169.254.169.254`. And, of course, you need to manage mapping of OCI hostnames and private
IP addresses in on-premises DNS registry.

While this option is certainly viable, it brings significant management overhead that can
be eliminated by private hybrid DNS.


## IP Addresses instead of Hostnames

You might also ask, why not to use private IP addresses instead of hostnames?

The first reason is __flexibility__ - private IP address may change when you reprovision an
instance or when you move the instance to different location. Also, some services use
dynamic IP addresses to provide scalability, HA and DR capabilities.

The second, also important reason, is __security__ - SSL enabled OCI services require FQDN
(fully qualified domain name), as SSL certificates are based on domain names. You cannot
use these services with IP addresses.


# __How VCN Resolver Works__

## DNS Resolver

When you provision a new Virtual Cloud Network (VCN), OCI automatically creates a
__dedicated DNS Resolver__ for the VCN, with the same name as VCN. All instances in the
VCN may use the resolver by sending DNS queries to `169.254.169.254`. The VCN resolver
automatically resolves hostnames in VCN's private views. Every VCN contains a single
private view with all the hostnames in VCN; hence hostnames within VCN are resolved
automatically. You may also add additional private views.

Note the address `169.254.169.254` is the same address as used by instance metadata
service (IMDS) in OCI. IMDS is provided to every instance via primary VNIC - you do not
have to configure any routing and security list in the VCN for the IMDS (and the default
VCN resolver) to work. More information about IMDS is available here:
https://docs.oracle.com/en-us/iaas/Content/Compute/Tasks/gettingmetadata.htm


## Private Views

The DNS Resolver automatically resolves only the hostnames in the VCN. If you need to
resolve hostnames from other VCNs in the same region, you may __associate private views__,
corresponding to these other VCNs, with the DNS Resolver. This is a simple operation that
can be done in OCI Console, or through CLI, API, or SDK. Note the association works in the
same region; you cannot associate private view from another region.


## Forwarding Endpoint

The DNS Resolver can also forward DNS queries for selected domains to other nameservers.
This feature is useful for resolving on-premises hostnames in OCI; or for resolving
hostnames from other OCI regions. To forward the DNS queries, you have to create a
__Forwarding Endpoint__ in the DNS Resolver, from which the resolver will send the
queries. Furthermore, you have to add a __Forwarding Rule__ to forward queries to
specified DNS domains and the IP address of the external nameserver that will resolve
these queries.

It is worth noting that currently it is possible to specify only one nameserver for the
given domain. If you define another rule with the same domain and another nameserver, the
rule will be ignored even if the first nameserver fails.


## Listening Endpoint

And finally, the DNS Resolver can answer DNS queries from other nameservers. This
feature is useful for resolving OCI hostnames in the on-premises network; or for resolving
hostnames from other OCI regions. To answer DNS queries, you have to create a __Listening
Endpoint__ in the VCN. Furthermore, you have to configure forwarding rules in the source
nameservers, that will redirect DNS queries to OCI domains to the Listening Endpoint.


# __Private Hybrid DNS Design__

## Concept

Using the VCN DNS Resolver capabilities described in the above section, we can design
hybrid DNS with on-premises hostnames managed in the on-premises DNS, and OCI hostnames
managed by the OCI DNS. The diagram shows the design for single OCI region and single
on-premises data center; however it can be easily extended to multiple regions and multiple
data centers.

The design is based on hub-and-spoke topology, with DNS Resolver in Management VCN being
the hub, and DNS Resolvers in workload VCNs and DNS Server on premises being the spokes.

![Hybrid DNS Design](/images/2022-08-05-private-hybrid-dns-in-oci/private-hybrid-dns-in-oci.png)


## DNS Resolver in Management VCN

The DNS Resolver in the Management VCN plays the role of DNS hub for other workload VCNs
and on-premises DNS.

It contains associated private views of workload VCNs `wk1-fra-vcn` and `wk2-fra-vcn`; and
therefore it can resolve hostnames not only in the Management VCN, but also hostnames of
instances in these workload VCNs.

Furthermore, it contains Forwarding Endpoint `forwarder.dns.mgmtfra.oraclevcn.com` and a
rule, which forwards DNS queries to on-premises domains like `abc.com` to the on-premises
DNS Server. In other words, it is also able to resolve hostnames from the on-premises
domain. Note the on-premises DNS server must be spcified by IP address; not by its hostname.

And finally, it contains Listening Endpoint `listener.dns.mgmtfra.oraclevcn.com`, which
answers DNS queries coming from on- premises, from the workload VCNs, and possibly also
from VCNs in other regions.


## DNS Resolver in Workload VCNs

The DNS Resolvers in workload VCNs contain Forwarding Endpoints such as
`forwarder.dns.wk1fra.oraclevcn.com` and a rule, which forwards DNS queries to on-premises
domains like `abc.com` to the hub listener `listener.dns.mgmtfra.oraclevcn.com`. Note the
hub listener must be specified by IP address; not by its hostname.

The DNS Resolvers in workload VCNs may also contain associated views of other VCNs such as
`mgmt-fra-vcn`, so that they can resolve hostnames from other VCNs in the same region.

Alternatively, the local DNS Resolver may be configured to forward DNS queries to other
VCNs to hub listener. However, I believe that associating private views is more simple
and easier to manage, as private view may contain multiple zones with different domains,
such as `oraclevcn.com`, `oraclecloud.com`, or `oraclecloudapps.com`, particularly if
you use platform services with private endpoints, such as Autonomous Database or OAC.


## On-premises DNS Server

The on-premises DNS Server must be configured to forward DNS queries for OCI domains to
the hub listener `listener.dns.mgmtfra.oraclevcn.com`. In our example, this includes
domains `mgmtfra.oraclevcn.com`, `wk1fra.oraclevcn.com`, and `wk2fra.oraclevcn.com`. Note
the hub listener must be specified by IP address; not by its hostname.


## Forwarding Rules and VCN DNS Labels

When you create VCNs in OCI, you have to specify DNS Label of VCNs in a way that supports
forwarding rules. I strongly recommend including abbreviation of OCI region in the DNS
Label - e.g., `mgmtfra` for DNS Label of Management VCN in Frankfurt region, or `mgmtlhr`
for DNS Label of Management VCN in London region.

With this naming convention, on-premises DNS Server may forward DNS queries to domain
`*fra.oraclevcn.com` to hub listener in Frankfurt Management VCN; and DNS queries to
domain `*lhr.oraclevcn.com` to hub listener in London Management VCN.


## Security Lists and Firewalls

The network configuration in both OCI and on-premises networks must allow DNS traffic
between the on-premises DNS Server, DNS Resolver Listening Endpoint, and DNS Resolver
Forwarding Endpoint. The DNS traffic consists of ingress (to Listening Endpoint and to on-
premises DNS Server) and egress (from Forwarding Endpoint and from on-premises DNS Server)
for `UDP` and `TCP` on protocol `53`. Both `UDP` and `TCP` must be enabled.

On the OCI side, the preferred way of enabling the DNS traffic is to define Network
Security Group (NSG) in each VCN that will contain Listening and Forwarding Endpoints and
allow `UDP/53` and `TCP/53` traffic.

As already mentioned, DNS queries from instances in OCI VCNs to DNS resolver on
`169.254.169.254` do not require any configuration as the name service is automatically
available on every OCI instance.

