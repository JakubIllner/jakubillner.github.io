---
title: Decentralized Data Lake on OCI Object Storage (Part 1)
description: How to design decentralized and distributed Data Lake on OCI Object Storage
tags:
- Data Lake
- OCI Object Storage
- Oracle Cloud Infrastructure
---


![Intro Picture](/images/2022-12-30-data-lake-on-oci-object-storage/icecream.jpg)

# __Introduction__

Data Lake is a well established concept for storing, managing, and accessing large volumes
of structured and unstructured data on scalable, cost-efficient storage. Many data lakes
use cloud based object storage such as OCI Object Storage, AWS S3 or Microsoft Azure Blob
Storage, because of the scalability, data durability, and wide adoption of standard APIs
(Swift, S3).

Main reason for popularity of data lakes is agility - it is easy to onboard new data
entity or change an existing one, using the schema-on-read approach. There is no need for
upfront data modeling as in a data warehouse. Data retains the original content, either in
the same file format as provided by the source, or using optimized data formats such as
Parquet.

Data lake practitioners recognized a long time ago that this agility comes with a price.
If anybody can load or access any data, and if the data structures can change anytime, the
data lake will quickly become unusable, unmanageable, and unsecured. In other words, a
data lake requires governance and security to ensure data can be discovered, accessed, and
used by only entitled people or applications.

This post proposes an approach of how to apply the governance to a Data Lake on Oracle
Cloud Infrastructure (OCI) that use OCI Object Storage as data store. It addresses the
organization of Compartments, Groups, Policies, Buckets, and Data Catalog assets in a
multi-domain Data Lake, with decentralized and possibly geographically distributed data
domains.

This is the first part of the post describing Data Lake on Oracle Cloud Infrastructure
(OCI) with OCI Object Storage as data store. The second part is available here:
[Decentralized Data Lake on OCI Object Storage (Part 2)](https://jakubillner.github.io/2023/02/19/data-lake-on-oci-object-storage-2.html).


# __Principles__

Data domains and decentralization, that sounds like terms taken from Data Mesh concept.
Indeed, a decentralized Data Lake should comply with the key Data Mesh principles
formulated by Zhamak Dehghani in the post [Data Mesh Principles and Logical
Architecture](https://martinfowler.com/articles/data-mesh-principles.html) and described
in detail in the great Data Mesh book that Zhamak Dehghani published in March 2022.

* __Domain Ownership__ - the decentralized Data Lake consists of several independent data
domains, with decentralized data organization, ownership, and access control. The domains
should be isolated, without any common components.

* __Data as a Product__ - the Data Lake may contain both data published to users of the
domain (data product) as well as data internal to the domain. For example, data in the
Bronze layer (raw data from sources) may be considered as internal, while data in the
Silver layer (curated data) are the Data Product consumable by domain users. 

* __Self-serve Data Platform__ - in our case, the Data Platform is Oracle Cloud
Infrastructure, with OCI Object Storage, OCI Data Catalog and many other data management,
integration, analytics, governance, security, and infrastructure services.

* __Federated Computational Governance__ - the governance of Data Lake is the main topic
of this post, with the primary focus on access control, data security, and data
organization, in the decentralized, federated world.


# __Use Case__

I will use a fictional Revenue Assurance Analysis to illustrate Data Lake structure.

The objective of the Revenue Assurance is to verify that a company correctly bills for
products and services according to the commercial agreement with the clients. Revenue
Assurance Analysis identifies revenue leakage, for example caused by misconfigured
applications (prices, durations, eligibility, business rules etc.) or by fraudulent
activity.


## Data Domains

The Data Lake consists of 3 source oriented data domains, 1 consumer oriented domain, and
shared data platform.

![Use Case](/images/2022-12-30-data-lake-on-oci-object-storage/data-lake-on-oci-object-storage-use-case.png)

* __Subscription__ - source oriented domain which contains information about customers,
their contracts, and services they have subscribed to or which they can use. For the sake
of simplicity let's assume this domain contains also the pricing information. Subscription
domain is owned by the Customer Service team.

* __Usage__ - source oriented domain which contains information about how customers use
products and services. This information is available on detail level - there is one record
for every utilization of a service by a customer. Usage domain is owned by the Network team.

* __Billing__ - source oriented domain which contains information about usage and
subscription fees paid by customers for services, either via monthly bills or in real-time
from their credit. This includes invoices, invoice lines, and also rated usage records.
Billing domain is owned by the Billing team.

* __Revenue Assurance__ - consumer oriented domain which compares Subscription and Usage
data with Billing information to verify the Billing charges are correctly calculated. The
output is a data set with differences between billed revenue and amount calculated from
Subscription and Usage. This domain is owned by the Revenue Assurance team.

* __Data Platform__ - shared data platform providing infrastructure, product, security,
and governance services to all data domains in the Data Lake. An example of the governance
service is Shared Data Catalog, providing description of data products to Data Lake users.


## Personas

Let's assume the Data Lake will be created, managed, and used by the following personas.

![Personas](/images/2022-12-30-data-lake-on-oci-object-storage/data-lake-on-oci-object-storage-personas.png)

Data domain personas needed for every domain are:

* __Domain Administrators__ - administrators responsible for management of resources
required by the domain. They are also responsible for providing access to the data product
to consumers from other domains.

* __Domain DataOps__ - developers and operators of all capabilities of the data domain.
They develop and operate ingestion and transformation processes, design data models,
manage data stores, develop and monitor data products, provide data domain functional
requirements.

* __Domain Agents__ - non-human, automated agents that run the production data
extraction, validation, processing, lifecycle management, publishing and other regular or
continuous processes.

* __Domain Users__ - human users of data and other business functions of the data
domain. Users have read-only access to the domain data.

Data platform personas:

* __Data Platform Administrators__ - administrators responsible for management of
resources required by the data platform. They also onboard new data domains.

* __Data Platform Governance__ - data governance engineers responsible for managing and
operating governance services such as shared data catalog.

* __Data Platform Agents__ - non-human, automated agents that run shared services such as
harvesting of metadata information from data assets into data catalog.


## Services

Our Data Lake uses several OCI services for processing, transforming, publishing, and
analyzing data.

![Design](/images/2022-12-30-data-lake-on-oci-object-storage/data-lake-on-oci-object-storage-solution.png)

* __Source Oriented Domains__ get source data from Landing bucket in OCI Object Storage.
Source data is validated, transformed, and aggregated via OCI Data Integration service.
The output is stored in the Prepared Data bucket in OCI Object Storage. Data in this bucket
is published to Data Lake users as Source Data Product. 

* __Revenue Assurance Domain__ uses Subscription, Usage, and Billing data from respective
Source Data Products. It employes Spark application deployed in OCI Data Flow to
calculate differences between billed and calculated amounts. The differences are published
to the Prepared Data bucket in OCI Object Storage.

* __Revenue Assurance Analytics__ is part of Revenue Assurance Domain. It delivers
analytical environment to business users. This application consists of Autonomous Data
Warehouse and Oracle Analytics Cloud instances for exploration and analysis by domain
users.

* __Data Catalog__ is a shared Data Platform component for data discovery and governance.
Domain engineers publish data product descriptions, schemas, addresses, and other type of
metadata to OCI Data Catalog. Users may search and browse this information.


# __Governance of Data Lake Resources__

This section describes how you can organize Compartments, Groups, and Policies to
guarantee isolation between Data Lake data domains.


## __Compartments__

Compartments are basic isolation mechanisms in OCI, used to manage access to groups of
related resources. Every resource in OCI (such as Object Storage bucket) belongs to a
compartment. A compartment defines scope for policies granting privileges to users.
Compartments are hierarchical, with the "root" compartment automatically created for every
tenancy.

![Compartments](/images/2022-12-30-data-lake-on-oci-object-storage/data-lake-on-oci-object-storage-compartments.png)


Every data domain manages its own resources. Resources in one domain must be isolated from
the other domains. To implement this Domain Ownership principle, I recommend creating
__one Compartment for every data domain and to put all resources belonging to a domain to
this Compartment__.

Shared, Data Platform resources, do not belong to any single data domain. They are managed
by cross-domain, Data Platform team. Therefore, we need __additional Compartment with all
the Data Platform components__.

And finally, all the Data Lake resources might be __encompassed into the Data Lake Master
Compartment__. This Compartment is not mandatory but it is useful for enforcing governance
standards that you want to apply to all data domains - such as Tagging or Quotas.


## __Groups__

OCI Identity and Access Management has two types of groups - Groups and Dynamic Groups.

Group is a collection of users who all need the same type of access to a particular set
of resources in a compartment. Users in a Group are usually human users that need to work
with OCI resources. Users have IAM credentials used for authentication.

Dynamic Group is a collection of non-human, compute instances that may call OCI API
operations. Principals in Dynamic Group are authenticated by their membership in Dynamic
Group. The membership is based on matching rules that specify which resources in a
compartment belong to a Dynamic Group

![Groups](/images/2022-12-30-data-lake-on-oci-object-storage/data-lake-on-oci-object-storage-groups.png)

The Data Lake groups correspond to personas we identified in the [Use Case](#use-case)
section. Administrators, DataOps, and User groups have OCI IAM Groups each. Agents are
members of OCI IAM Dynamic Groups.

Note that OCI Users, Groups and Dynamic Groups are created on the tenancy level.
Management of Users and Groups is separated from the management of resources in the Data
Lake. It does not follow the decentralization principle.


## __Dynamic Groups__

Dynamic Groups are defined by matching rules that specify which resources and in which
compartments are in the group.

In our Data Lake example, the domain Agents are OCI Data Integration workspaces, OCI Data
Flow Runs, and other resource types (such as OCI Functions) created in the domain
compartment. To define the dynamic group `dl-<domain>-agents-dg`, any of the following
matching rules must be true.

```
ALL{resource.type = 'disworkspace', resource.compartment.id = '<OCID of domain compartment>'}
ALL{resource.type = 'dataflowrun', resource.compartment.id = '<OCID of domain compartment>'}
ALL{resource.type = 'fnfunc', resource.compartment.id = '<OCID of domain compartment>'}
```

Similarly, we can define the matching rule for dynamic group `dl-platform-dcat-agents-dg`,
which contains data platform catalog Agents - instance(s) of OCI Data Catalog in the data
platform compartment.

```
ALL{resource.type = 'datacatalog', resource.compartment.id = '<OCID of data platform compartment>'}
```


## Policies

A Policy is a document that specifies who can access which resources and in what
compartment, and what operations they can perform. Access is granted at the group and
compartment level. Privileges granted in a compartment are automatically inherited by all
sub-compartments.

Because of Domain Ownership principle and decentralized management of data domains, domain
administrators can manage all resources in their domain Compartments. Note this privilege
includes also the right to define Policies in these Compartments.

![Policies](/images/2022-12-30-data-lake-on-oci-object-storage/data-lake-on-oci-object-storage-policies.png)


### A. Administration Policies

#### Master Policy `dl-master-admin-policy`

Tenancy administrator grants the Data Lake administrator privilege to manage all resources
in Data Lake.

```
allow group dl-master-admin-grp to manage all-resources in compartment dl-master-cmp
```


#### Domain Administration Policy `dl-domain-admin-policy`

Data Lake administrator grants the Data Domain administrators privilege to manage all
resources in the domains they own, including the rights to define Policies.

```
allow group dl-subscription-admin-grp to manage all-resources in compartment dl-subscription-cmp
allow group dl-usage-admin-grp to manage all-resources in compartment dl-usage-cmp
allow group dl-billing-admin-grp to manage all-resources in compartment dl-billing-cmp
allow group dl-revassurance-admin-grp to manage all-resources in compartment dl-revassurance-cmp
```


#### Data Platform Administration Policy `dl-platform-admin-policy`

Data Lake administrator grants Data Platform administrators privilege to manage all
resources in the Data Platform.

```
allow group dl-platform-admin-grp to manage all-resources in compartment dl-platform-cmp
```


### B. Domain Policies

The Policies in this section must be defined for every Domain.


#### Domain DataOps Policy `dl-<domain>-dataops-policy`

Data Domain administrator grants DataOps engineers privilege to manage selected resources
in the domain they own. This policy should be modified to include only the resource types
that DataOps engineers need in the particular domain.

```
allow group dl-<domain>-dataops-grp to manage object-family in compartment dl-<domain>-cmp
allow group dl-<domain>-dataops-grp to manage dis-family in compartment dl-<domain>-cmp
allow group dl-<domain>-dataops-grp to manage dataflow-family in compartment dl-<domain>-cmp
allow group dl-<domain>-dataops-grp to manage autonomous-database-family in compartment dl-<domain>-cmp
allow group dl-<domain>-dataops-grp to manage analytics-instances in compartment dl-<domain>-cmp
allow group dl-<domain>-dataops-grp to manage analytics-instance-work-requests in compartment dl-<domain>-cmp
```

This policy grants the following privileges:

* Manage OCI Object Storage buckets and objects in the compartment.
* Manage OCI Data Integration workspaces (including projects, tasks, applications, etc.) in the compartment.
* Manage OCI Data Flow applications, clusters and runs in the compartment.
* Manage Autonomous Database instances in the compartment.
* Manage Oracle Analytics Cloud instances in the compartment.


#### Domain Agent Policy `dl-<domain>-agent-policy`

Data Domain administrator grants resource principals (automated Agents) privilege to
execute the domain pipeline and perform automated tasks. This policy should be modified to
include only the resource types and operations that Agents need to access in the
particular domain.

```
allow dynamic-group dl-<domain>-agents-dg to manage objects in compartment dl-<domain>-cmp
allow dynamic-group dl-<domain>-agents-dg to read buckets in compartment dl-<domain>-cmp
allow dynamic-group dl-<domain>-agents-dg to use dis-workspaces in compartment dl-<domain>-cmp where all {request.permission = 'DIS_WORKSPACE_OBJECT_EXECUTE'}
allow dynamic-group dl-<domain>-agents-dg to read dataflow-application in compartment dl-<domain>-cmp
allow dynamic-group dl-<domain>-agents-dg to manage dataflow-run in compartment dl-<domain>-cmp
```

This policy grants the following privileges:

* Read, create, delete, or rename OCI Object Storage objects in the compartment.
* Execute OCI Data Integration tasks in the compartment.
* Create, update, or cancel OCI Data Flow runs in the compartment.


#### Domain Access Policy `dl-<domain>-access-policy`

Data Domain administrator grants users and resource principals (automated Agents)
privilege to read data from the domain Data Product. __This policy realizes the
cross-domain access to Data Product and it must be updated whenever some other domain or
group of users need to access the domain Data Product__.

```
allow group dl-<domain>-user-grp to read object-family in compartment dl-<domain>-cmp where target.bucket.name='dl-<domain>-data-bucket'
allow group dl-<otherdomain>-user-grp to read object-family in compartment dl-<domain>-cmp where target.bucket.name='dl-<domain>-data-bucket'
allow dynamic-group dl-<otherdomain>-agents-dg to read object-family in compartment dl-<domain>-cmp where target.bucket.name='dl-<domain>-data-bucket'
allow dynamic-group dl-platform-dcat-agents-dg to read object-family in compartment dl-<domain>-cmp where target.bucket.name='dl-<domain>-data-bucket'
```

This policy grants the following privileges:

* Read OCI Object Storage bucket details of the domain Data Product bucket.
* Read OCI Object Storage objects in the domain Data Product bucket.

The policy allows reading of __all__ objects in the Data Product bucket. If you want to
provide access only to subset of objects, you need to separate these objects into another
bucket and define the access rules on the bucket level. Note this restriction may be
removed once OCI Object Storage supports object level access privileges.


### C. Data Platform Policies

#### Data Catalog Administration Policy `dl-platform-cat-admin-policy`

Data Platform administrator grants Data Governance engineers privilege to manage OCI Data
Catalog instances in the Data Platform.

```
allow group dl-platform-governance-grp to manage data-catalog-family in compartment dl-platform-cmp
```


#### Data Catalog Asset Management Policy `dl-platform-cat-asset-policy`

Data Platform administrator grants domain DataOps engineers privilege to manage the Data
Asset corresponding to their domain Data Product in the Data Catalog. __This policy
realizes the read-write access to Shared Data Catalog and it must be updated whenever a
new domain domain is added__. Note that this policy requires the Data Asset is created by
Data Governance stewards before it can be managed by DataOps engineers.

```
allow group dl-subscription-dataops-grp to read data-catalogs in compartment dl-platform-cmp'
allow group dl-subscription-dataops-grp to use data-catalog-data-assets in compartment dl-platform-cmp where target.data-asset.key='<subscription data asset key>'
allow group dl-usage-dataops-grp to read data-catalogs in compartment dl-platform-cmp'
allow group dl-usage-dataops-grp to use data-catalog-data-assets in compartment dl-platform-cmp where target.data-asset.key='<usage data asset key>'
allow group dl-billing-dataops-grp to read data-catalogs in compartment dl-platform-cmp'
allow group dl-billing-dataops-grp to use data-catalog-data-assets in compartment dl-platform-cmp where target.data-asset.key='<billing data asset key>'
allow group dl-revassurance-dataops-grp to read data-catalogs in compartment dl-platform-cmp'
allow group dl-revassurance-dataops-grp to use data-catalog-data-assets in compartment dl-platform-cmp where target.data-asset.key='<revassurance data asset key>'
```

This policy grants the following privileges:

* Browse and search OCI Data Catalog data asset corresponding to the domain
* Update OCI Data Catalog data asset corresponding to the domain (connections, entities,
attributes, folders, patterns etc.)


#### Data Catalog User Policy `dl-platform-cat-user-policy`

Data Platform administrator grants Data Catalog users privilege to list, browse, or search
data assets, glossaries, and other metadata maintained in Data Catalog. __This policy
realizes the read-only access to Shared Data Catalog and it must be updated whenever some
other domain or group of users need to access the Data Catalog__.

```
allow group dl-subscription-user-grp to read data-catalog-family in compartment dl-platform-cmp
allow group dl-usage-user-grp to read data-catalog-family in compartment dl-platform-cmp
allow group dl-billing-user-grp to read data-catalog-family in compartment dl-platform-cmp
allow group dl-revassurance-user-grp to read data-catalog-family in compartment dl-platform-cmp
```

This policy grants the following privileges:

* Browse and search OCI Data Catalog data assets
* Browse and search OCI Data Catalog glossaries


## __Data Lake Components__


### __Service Instances__

Once the compartments, groups, and policies are in place, data domain DataOps engineers
can provision and start using the components required by data domains. This includes
provisioning of Object Storage buckets and instances of data processing services such as
OCI Data Integration workspaces and OCI Data Flow applications.


![Policies](/images/2022-12-30-data-lake-on-oci-object-storage/data-lake-on-oci-object-storage-resources.png)


### __Catalog Data Assets__

Data domain DataOps engineers can also describe the domain Data Products in the Shared
Data Catalog. For every Data Product there should be one data asset containing Data
Product entities and attributes. Note the domain internal data objects (e.g. objects in
the Landing bucket) are usually not exposed in the Data Catalog.

![Data Catalog](/images/2022-12-30-data-lake-on-oci-object-storage/data-lake-on-oci-object-storage-catalog.png)


## __Summary__

This post shows how to design the governance of an OCI based Data Lake that uses OCI
Object Storage as data store. The main design rules described in the post are:

* __Domain Ownership__ - Data Lake consists of multiple, isolated, and independent data domains.
* __Data as a Product__ - Data is exchanged via Data Products, implemented as entities stored in OCI Object Storage buckets.
* __Compartments__ - There is one Compartment for every data domain, to provide isolation of domains.
* __Groups__ - Separate Groups are used for administrators, DataOps engineers, and users of every data domain.
* __Dynamic Groups__ - Separate Dynamic Groups are used for non-human, data processing agents of every data domain.
* __Policies__ - Policies apply rules to allow independent development, management, and operations of data domains.
* __Shared Data Catalog__ - Shared instance of OCI Data Catalog is used to publish information about data products.
* __Buckets__ - Data Product entities are separated from internal domain objects by using different Object Storage buckets.
* __Data Processing__ - Data processing services are using whatever services or tools are suitable and preferred by the domain team.

The domains may be geographically separated in different OCI regions, or they may located
within single region only.

Note I have intentionally disregarded the topic of networking for the sake of simplicity.
All the services used in this post support public access, so theoretically the presented
solution could work without any networking components. But, in reality, most designs would
require placement of components into private Virtual Client Networks (VCNs) and
establishing communication between data products.


## __Resources__

* Management of OCI Compartments: [Managing Compartments](https://docs.oracle.com/en-us/iaas/Content/Identity/compartments/managingcompartments.htm).
* Management of OCI Groups: [Managing Groups](https://docs.oracle.com/en-us/iaas/Content/Identity/groups/managinggroups.htm).
* Management of OCI Dynamic Groups: [Managing Dynamic Groups](https://docs.oracle.com/en-us/iaas/Content/Identity/dynamicgroups/managingdynamicgroups.htm).
* Management of access to resources with OCI Policies: [Managing Access to Resources](https://docs.oracle.com/en-us/iaas/Content/Identity/access/manage-accessresources.htm).
* Overview of OCI Data Catalog: [Data Catalog Overview](https://docs.oracle.com/en-us/iaas/data-catalog/using/overview.htm).


