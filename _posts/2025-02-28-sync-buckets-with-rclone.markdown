---
title: Bi-directional Replication between Object Storage Buckets
description: Using rclone to bi-directionally replicate data between Object Storage buckets
tags:
- Oracle Cloud Infrastructure
- Object Storage
- rclone
---


![Intro Picture](/images/2025-02-28-sync-buckets-with-rclone/winter.JPG)

# __Introduction__

Oracle Cloud Infrastructure (OCI) Object Storage supports replication from one bucket to another
out-of-the-box, using the policy driven, completely automatic replication feature
described in [Object Storage Replication](https://docs.oracle.com/en-us/iaas/Content/Object/Tasks/usingreplication.htm).
This feature can replicate objects between buckets in different regions. So, can it be
used in the scenario when two buckets must be kept in sync and the data arrives in any of
them?

The answer is no. The built-in replication is uni-directional. Once you enable the
replication policy, the target bucket becomes read-only. And there are other restrictions.
You can replicate to single bucket only. And the target bucket must be in the same tenancy
as the source bucket.

Therefore, if you need more complex replication topologies, such as bi-directional,
active-active replication, data distribution from single source bucket to multiple
targets, or data consolidation from multiple buckets to single target, you cannot use
built-in replication.

![Replication Topologies](/images/2025-02-28-sync-buckets-with-rclone/rclone-replication-topologies.png)

Fortunately, __rclone__ comes to rescue. With rclone, you can easily implement all the above
scenarios. This post describes the installation and configuration of rclone on OCI Compute
and how to use it for the bi-directional replication between OCI buckets. You can easily
modify this example to other scenarios.



# __Rclone__

Rclone is an open-source, feature rich, command-line program to manage files on cloud
storages and on local file systems, to copy and synchronize files between different type of
storages. There are many supported providers, including the major cloud storages such as
Amazon S3, Microsoft Azure Blob Storage, or Google Cloud Storage. For more information
please refer to [https://rclone.org/](https://rclone.org/).

One of the well supported providers is OCI Object Storage. Together with OCI CLI and SDK,
rclone can be used to load local files to Object Storage, download files from Object
Storage, copy or move files between Object Storage and other cloud stores, or synchronize
files between multiple buckets in the Object Storage. For more information on using rclone
with OCI Object Storage please refer to
[https://rclone.org/oracleobjectstorage/](https://rclone.org/oracleobjectstorage/).



# __Scenario__

The scenario described in this post is bi-directional replication between two Object
Storage buckets.

![Replication Topologies](/images/2025-02-28-sync-buckets-with-rclone/rclone-replication-concept.png)

* Data producers in the region A generate files and store them in the bucket A.
* Data producers in the region B generate files and store them in the bucket B.
* Rclone copies new files produced in the region A from bucket A to the bucket B.
* Rclone copies new files produced in the region B from bucket B to the bucket A.
* At the end, both buckets A and B contain the same files, from regions A and B.

For the simplicity, I assume that files generated in the region A are different (i.e.,
have different names) then files generated in the region B. In other words, it is not
necessary to resolve conflicts during the replication.


# __Deployment__

I tested the replication with rclone using the following deployment.

![Configuration](/images/2025-02-28-sync-buckets-with-rclone/rclone-replication-configuration.png)

* Regions - the test configuration is deployed in two regions, UK South (London) and
Switzerland North (Zurich). The resources are deployed into region-specific compartments,
`sandbox-london` and `sandbox-zurich` respectively.

* Object Storage - two buckets, `test-bucket-london` in London region and compartment
`sandbox-london`, and `test-bucket-zurich` in Zurich region and compartment `sandbox-
zurich`.

* Compute VM - VM.Standard.E5.Flex shape with 1 OCPUs and 16 GB of memory, and running
Oracle Linux 8. The VM is placed in a private subnet in the London region. I installed
only the rclone on top of the Oracle Linux 8 image.

* Connectivity - Compute VM accesses Object Storage buckets via Storage Gateway. The
inter-regional traffic is routed over the OCI Backbone, it does not traverse Internet.
There is also NAT Gateway for installation of the rclone.


# __Installation__

I installed the latest version of rclone using the recommended script installation.

```
$ sudo -v ; curl https://rclone.org/install.sh | sudo bash
...
rclone v1.69.1 has successfully installed.
Now run "rclone config" for setup. Check https://rclone.org/docs/ for more details.

$ rclone --version
rclone v1.69.1
- os/version: oracle 8.9 (64 bit)
- os/kernel: 5.15.0-205.149.5.1.el8uek.x86_64 (x86_64)
- os/type: linux
- os/arch: amd64
- go/version: go1.24.0
- go/linking: static
- go/tags: none
```

That's it - no more installation steps are needed. Rclone does not require installation of
OCI SDK's or other packages.


# __Configuration__

## OCI Configuration

To connect to OCI Object Storage APIs, I created two profiles in `~/.oci/config` file. The
first profile will be used to connect to the Object Storage in the London region, the second
one to the Object Storage in the Zurich region.

```
[LONDON]
region=uk-london-1
tenancy=<London Tenancy OCID>
user=<London User OCID>
fingerprint=<London User Signing Key Fingerprint>
key_file=<London User Signing Key>

[ZURICH]
region=eu-zurich-1
tenancy=<Zurich Tenancy OCID>
user=<Zurich User OCID>
fingerprint=<Zurich User Signing Key Fingerprint>
key_file=<Zurich User Signing Key>
```

* region - code of the region.
* tenancy - OCID of the tenancy.
* user - OCID of the user (for user principal authentication).
* fingerprint - fingerprint of the user signing key.
* key_file - location of the signing key in the PEM format.

Note I used authentication via Signing Key. With this approach, the buckets could be not
only in different regions, but also in different tenancies. Alternatively, I could use
Instance Principal authentication for the London based tenancy.


## Rclone Configuration

In the second step, I defined two remote destinations in the rclone configuration file
`~/.config/rclone/rclone.conf`. Again, one  destination is for the Object Storage in the
London region, the other for the Object Storage in the Zurich region.

```
[OOS-LONDON]
type = oracleobjectstorage 
namespace = <London Namespace>
compartment = <London Compartment OCID>
region = uk-london-1 
provider = user_principal_auth 
config_file = ~/.oci/config 
config_profile = LONDON

[OOS-ZURICH]
type = oracleobjectstorage 
namespace = <Zurich Namespace>
compartment = <Zurich Compartment OCID>
region = eu-zurich-1
provider = user_principal_auth 
config_file = ~/.oci/config 
config_profile = ZURICH
```

* type - must be `oracleobjectstorage` for Object Storage.
* namespace - Object Storage namespace (on the tenancy level).
* compartment - OCID of the compartment with bucket(s).
* region - code of the region.
* provider - type of authentication (`user_principal_auth` for user principal and signing key).
* config_file - location of the OCI configuration file.
* config_profile - profile in the OCI configuration file defined in the previous step.


# __Replication__

## Design

For bi-directional replication, we need two pipelines. One pipeline replicates files from
the bucket in the London region to the bucket in the Zurich region, the other replicates
files in the opposite direction.

![Replication](/images/2025-02-28-sync-buckets-with-rclone/rclone-replication-replication.png)

Pipelines use rclone `copy` command to transfer files from the source to the target
bucket. Files which are present at the destination bucket and which are identical are not
copied. Rclone uses size and checksum to determine if a file is identical or not. `copy`
command does not delete files in the destination bucket.


## Test

### Initial Replication

Initially, there are 3 files in the bucket `test-bucket-london` and 2 files in the bucket
`test-bucket-zurich`.

```
$ rclone lsl "OOS-LONDON:test-bucket-london"
 81339603 2025-03-02 18:02:15.000000000 source-a/custsales-2019-01.csv
 66716592 2025-03-02 18:02:02.000000000 source-a/custsales-2019-02.csv
 86484539 2025-03-02 18:02:15.000000000 source-a/custsales-2019-03.csv
 
$ rclone lsl "OOS-ZURICH:test-bucket-zurich"
 97008913 2025-03-02 18:03:45.000000000 source-b/custsales-2019-04.csv
108381879 2025-03-02 18:03:45.000000000 source-b/custsales-2019-05.csv
```

In the next step, I copy the London files to Zurich and vice versa.

```
$ rclone copy "OOS-LONDON:test-bucket-london" "OOS-ZURICH:test-bucket-zurich" --verbose
2025/03/02 18:09:23 INFO  : source-a/custsales-2019-01.csv: Copied (new)
2025/03/02 18:09:23 INFO  : source-a/custsales-2019-02.csv: Copied (new)
2025/03/02 18:09:24 INFO  : source-a/custsales-2019-03.csv: Copied (new)
2025/03/02 18:09:24 INFO  : 
Transferred:   	  223.675 MiB / 223.675 MiB, 100%, 102.878 MiB/s, ETA 0s
Transferred:            3 / 3, 100%
Elapsed time:         2.5s

$ rclone copy "OOS-ZURICH:test-bucket-zurich" "OOS-LONDON:test-bucket-london" --verbose
2025/03/02 18:10:04 INFO  : source-b/custsales-2019-04.csv: Copied (new)
2025/03/02 18:10:05 INFO  : source-b/custsales-2019-05.csv: Copied (new)
2025/03/02 18:10:05 INFO  : 
Transferred:   	  195.876 MiB / 195.876 MiB, 100%, 97.938 MiB/s, ETA 0s
Checks:                 3 / 3, 100%
Transferred:            2 / 2, 100%
Elapsed time:         2.4s
```

And we can see that both buckets contain the same files.

```
$ rclone lsl "OOS-LONDON:test-bucket-london"
 81339603 2025-03-02 18:02:15.000000000 source-a/custsales-2019-01.csv
 66716592 2025-03-02 18:02:02.000000000 source-a/custsales-2019-02.csv
 86484539 2025-03-02 18:02:15.000000000 source-a/custsales-2019-03.csv
 97008913 2025-03-02 18:03:45.000000000 source-b/custsales-2019-04.csv
108381879 2025-03-02 18:03:45.000000000 source-b/custsales-2019-05.csv

$ rclone lsl "OOS-ZURICH:test-bucket-zurich"
 81339603 2025-03-02 18:02:15.000000000 source-a/custsales-2019-01.csv
 66716592 2025-03-02 18:02:02.000000000 source-a/custsales-2019-02.csv
 86484539 2025-03-02 18:02:15.000000000 source-a/custsales-2019-03.csv
 97008913 2025-03-02 18:03:45.000000000 source-b/custsales-2019-04.csv
108381879 2025-03-02 18:03:45.000000000 source-b/custsales-2019-05.csv
```

### Incremental Replication

To test the incremental replication, I added 1 new file to the bucket `test-bucket-london`
and 3 files to the bucket `test-bucket-zurich`.

```
$ rclone lsl "OOS-LONDON:test-bucket-london"
 81339603 2025-03-02 18:02:15.000000000 source-a/custsales-2019-01.csv
 66716592 2025-03-02 18:02:02.000000000 source-a/custsales-2019-02.csv
 86484539 2025-03-02 18:02:15.000000000 source-a/custsales-2019-03.csv
 87174708 2025-03-02 18:15:46.000000000 source-a/custsales-2019-06.csv
 97008913 2025-03-02 18:03:45.000000000 source-b/custsales-2019-04.csv
108381879 2025-03-02 18:03:45.000000000 source-b/custsales-2019-05.csv

$ rclone lsl "OOS-ZURICH:test-bucket-zurich"
 81339603 2025-03-02 18:02:15.000000000 source-a/custsales-2019-01.csv
 66716592 2025-03-02 18:02:02.000000000 source-a/custsales-2019-02.csv
 86484539 2025-03-02 18:02:15.000000000 source-a/custsales-2019-03.csv
 97008913 2025-03-02 18:03:45.000000000 source-b/custsales-2019-04.csv
108381879 2025-03-02 18:03:45.000000000 source-b/custsales-2019-05.csv
111103164 2025-03-02 18:16:33.000000000 source-b/custsales-2019-07.csv
 82869431 2025-03-02 18:16:36.000000000 source-b/custsales-2019-08.csv
 69759240 2025-03-02 18:16:36.000000000 source-b/custsales-2019-09.csv
 ```

The `copy` command works incrementally, identifying and copying only the new files.

```
$ rclone copy "OOS-LONDON:test-bucket-london" "OOS-ZURICH:test-bucket-zurich" --verbose
2025/03/02 18:19:48 INFO  : source-a/custsales-2019-06.csv: Copied (new)
2025/03/02 18:19:48 INFO  : 
Transferred:   	   83.136 MiB / 83.136 MiB, 100%, 83.136 MiB/s, ETA 0s
Checks:                 5 / 5, 100%
Transferred:            1 / 1, 100%
Elapsed time:         1.3s

$ rclone copy "OOS-ZURICH:test-bucket-zurich" "OOS-LONDON:test-bucket-london" --verbose
2025/03/02 18:19:58 INFO  : source-b/custsales-2019-09.csv: Copied (new)
2025/03/02 18:19:59 INFO  : source-b/custsales-2019-08.csv: Copied (new)
2025/03/02 18:19:59 INFO  : source-b/custsales-2019-07.csv: Copied (new)
2025/03/02 18:19:59 INFO  : 
Transferred:   	  251.514 MiB / 251.514 MiB, 100%, 115.760 MiB/s, ETA 0s
Checks:                 6 / 6, 100%
Transferred:            3 / 3, 100%
Elapsed time:         2.5s
```

At the end, both buckets contain identical files.

```
$ rclone lsl "OOS-LONDON:test-bucket-london"
 81339603 2025-03-02 18:02:15.000000000 source-a/custsales-2019-01.csv
 66716592 2025-03-02 18:02:02.000000000 source-a/custsales-2019-02.csv
 86484539 2025-03-02 18:02:15.000000000 source-a/custsales-2019-03.csv
 87174708 2025-03-02 18:15:46.000000000 source-a/custsales-2019-06.csv
 97008913 2025-03-02 18:03:45.000000000 source-b/custsales-2019-04.csv
108381879 2025-03-02 18:03:45.000000000 source-b/custsales-2019-05.csv
111103164 2025-03-02 18:16:33.000000000 source-b/custsales-2019-07.csv
 82869431 2025-03-02 18:16:36.000000000 source-b/custsales-2019-08.csv
 69759240 2025-03-02 18:16:36.000000000 source-b/custsales-2019-09.csv
 
$ rclone lsl "OOS-ZURICH:test-bucket-zurich"
 81339603 2025-03-02 18:02:15.000000000 source-a/custsales-2019-01.csv
 66716592 2025-03-02 18:02:02.000000000 source-a/custsales-2019-02.csv
 86484539 2025-03-02 18:02:15.000000000 source-a/custsales-2019-03.csv
 87174708 2025-03-02 18:15:46.000000000 source-a/custsales-2019-06.csv
 97008913 2025-03-02 18:03:45.000000000 source-b/custsales-2019-04.csv
108381879 2025-03-02 18:03:45.000000000 source-b/custsales-2019-05.csv
111103164 2025-03-02 18:16:33.000000000 source-b/custsales-2019-07.csv
 82869431 2025-03-02 18:16:36.000000000 source-b/custsales-2019-08.csv
 69759240 2025-03-02 18:16:36.000000000 source-b/custsales-2019-09.csv
```


### Zero Replication

And finally, if there are no new files to transfer, `copy` command does nothing.

```
$ rclone copy "OOS-LONDON:test-bucket-london" "OOS-ZURICH:test-bucket-zurich" --verbose
2025/03/02 18:24:48 INFO  : There was nothing to transfer
2025/03/02 18:24:48 INFO  : 
Transferred:   	          0 B / 0 B, -, 0 B/s, ETA -
Checks:                 9 / 9, 100%
Elapsed time:         0.2s

$ rclone copy "OOS-ZURICH:test-bucket-zurich" "OOS-LONDON:test-bucket-london" --verbose
2025/03/02 18:24:55 INFO  : There was nothing to transfer
2025/03/02 18:24:55 INFO  : 
Transferred:   	          0 B / 0 B, -, 0 B/s, ETA -
Checks:                 9 / 9, 100%
Elapsed time:         0.2s
```


## Deletes

Rclone `copy` command transfers new files between buckets, but it does not handle deleted
files. If you need to apply also deletes, you can use `sync` command instead of `copy`.

__However, using `sync` command on the whole bucket in the bi-directional replication will
lead to data loss. To avoid that, you need to separate data originated in different
locations into folders, and apply the `sync` command to folders instead of buckets.__

For example, suppose that data producers in the London region put files into the folder
`source-a`, while data producers in the Zurich region put files into the folder `source-b`.
In other words, the London region is the master for `source-a`, while the Zurich region is
the master for `source-b`.

In this case we can synchronize both folders as follows:

```
$ rclone sync "OOS-LONDON:test-bucket-london/source-a" "OOS-ZURICH:test-bucket-zurich/source-a"
$ rclone sync "OOS-ZURICH:test-bucket-zurich/source-b" "OOS-LONDON:test-bucket-london/source-b"
```

Let's test it by deleting one file from every bucket.

```
$ rclone deletefile "OOS-LONDON:test-bucket-london/source-a/custsales-2019-01.csv" --verbose
2025/03/02 19:33:07 INFO  : custsales-2019-01.csv: Deleted

$ rclone deletefile "OOS-ZURICH:test-bucket-zurich/source-b/custsales-2019-04.csv" --verbose
2025/03/02 19:33:48 INFO  : custsales-2019-04.csv: Deleted
```

And running the `sync` command to apply the deleted to other buckets.

```
[opc@test-rclone-compute ~]$ rclone sync "OOS-LONDON:test-bucket-london/source-a" "OOS-ZURICH:test-bucket-zurich/source-a" --verbose
2025/03/02 19:35:48 INFO  : custsales-2019-01.csv: Deleted
2025/03/02 19:35:48 INFO  : There was nothing to transfer
2025/03/02 19:35:48 INFO  : 
Transferred:   	          0 B / 0 B, -, 0 B/s, ETA -
Checks:                 5 / 5, 100%
Deleted:                1 (files), 0 (dirs), 78.306 MiB (freed)
Elapsed time:         0.1s

[opc@test-rclone-compute ~]$ rclone sync "OOS-ZURICH:test-bucket-zurich/source-b" "OOS-LONDON:test-bucket-london/source-b" --verbose
2025/03/02 19:36:27 INFO  : custsales-2019-04.csv: Deleted
2025/03/02 19:36:27 INFO  : There was nothing to transfer
2025/03/02 19:36:27 INFO  : 
Transferred:   	          0 B / 0 B, -, 0 B/s, ETA -
Checks:                 5 / 5, 100%
Deleted:                1 (files), 0 (dirs), 92.515 MiB (freed)
Elapsed time:         0.1s
```

As you can see both files are deleted from both buckets.

```
$ rclone lsl "OOS-LONDON:test-bucket-london"
 66716592 2025-03-02 18:02:02.000000000 source-a/custsales-2019-02.csv
 86484539 2025-03-02 18:02:15.000000000 source-a/custsales-2019-03.csv
 87174708 2025-03-02 18:15:46.000000000 source-a/custsales-2019-06.csv
108381879 2025-03-02 18:03:45.000000000 source-b/custsales-2019-05.csv
111103164 2025-03-02 18:16:33.000000000 source-b/custsales-2019-07.csv
 82869431 2025-03-02 18:16:36.000000000 source-b/custsales-2019-08.csv
 69759240 2025-03-02 18:16:36.000000000 source-b/custsales-2019-09.csv
 
$ rclone lsl "OOS-ZURICH:test-bucket-zurich"
 66716592 2025-03-02 18:02:02.000000000 source-a/custsales-2019-02.csv
 86484539 2025-03-02 18:02:15.000000000 source-a/custsales-2019-03.csv
 87174708 2025-03-02 18:15:46.000000000 source-a/custsales-2019-06.csv
108381879 2025-03-02 18:03:45.000000000 source-b/custsales-2019-05.csv
111103164 2025-03-02 18:16:33.000000000 source-b/custsales-2019-07.csv
 82869431 2025-03-02 18:16:36.000000000 source-b/custsales-2019-08.csv
 69759240 2025-03-02 18:16:36.000000000 source-b/custsales-2019-09.csv
```


# __Automation__

In the production environment, you need to run the replication rclone pipelines
automatically. The easiest way is to use cron on the Compute VM and to periodically
execute the rclone commands from the cron.

You can also automate the rclone execution with Events, Functions, and Container
Instances. Configure buckets to emit an Event when a new file arrives or when a file is
deleted. Define Event Rule to execute a Function. The Function will run Container Instance
with rclone. With this approach you can synchronize buckets with minimal delay.


# __Resources__

* Main rclone page: [Rclone syncs your files to cloud storage](https://rclone.org/).
* Documentation and configuration of rclone with Object Storage: [Oracle Object Storage](https://rclone.org/oracleobjectstorage/).
* Rclone support announcement: [Announcing native OCI Object Storage provider backend support in rclone](https://blogs.oracle.com/cloud-infrastructure/post/native-oci-object-storage-support-rclone).
* Oracle Architecture Center: [Move data to object storage in the cloud by using Rclone](https://docs.oracle.com/en/solutions/move-data-to-cloud-storage-using-rclone/index.html).
* Mount Object Storage as file system: [Use OCI Object Storage as a Mounted Filesystem with Rclone](https://www.ateam-oracle.com/post/use-oci-object-storage-as-a-mounted-filesystem).