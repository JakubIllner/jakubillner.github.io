---
title: Querying Object Details from Autonomous Database
description: How to query details of objects stored in Object Storage from within Autonomous Database
tags:
- Oracle Autonomous Data Warehouse
- OCI Object Storage
- OCI PLSQL SDK
- Pipelined Table Functions
---


![Intro Picture](/images/2023-12-05-list-objects/limcovka.jpg)

# __Problem__

I was asked recently how to count objects in an OCI Object Storage bucket by the Storage
Tier, from within Oracle Autonomous Database (ADB).

Now this should be straightforward - Autonomous Database contains package `DBMS_CLOUD`, which
allows ADB to access and manage objects in the Object Storage. One of the modules in the
package is `LIST_OBJECTS` function, which lists all objects in a bucket.

Unfortunately, there is a catch - `LIST_OBJECTS` function does not return Storage Tier.

```
SQL> select *
  2  from dbms_cloud.list_objects('OCI$RESOURCE_PRINCIPAL','https://objectstorage.uk-london-1.oraclecloud.com/n/<namespace>/b/<bucket name>/o/')
  3* where rownum <= 10;

OBJECT_NAME                                                       BYTES CHECKSUM                      CREATED                                LAST_MODIFIED
__________________________________________________________ ____________ _____________________________ ______________________________________ ______________________________________
20231101T132937Z_20231101T142749Z.0.log                         3785999 M/kOPiNJnFKKkzg4wx4sHw==      28-NOV-23 03.00.47.715000000 PM GMT    28-NOV-23 03.00.47.715000000 PM GMT
bike-trips/2013-07-Citi-Bike-trip-data.csv                    164438561 drs4NeEQUN4kpodxKoB+uA==-3    14-NOV-23 12.53.23.193000000 PM GMT    16-NOV-23 08.11.43.538000000 AM GMT
bike-trips/2013-08-Citi-Bike-trip-data.csv                    195523200 +rp1qXjsNcj6wnqO9YBzCg==-3    14-NOV-23 12.53.28.660000000 PM GMT    16-NOV-23 08.12.03.630000000 AM GMT
bike-trips/2013-09-Citi-Bike-trip-data.csv                    201965642 iWdoPPeP/vAh4eZVN1JaeA==-4    14-NOV-23 12.39.00.468000000 PM GMT    16-NOV-23 08.12.19.380000000 AM GMT
bike-trips/2013-10-Citi-Bike-trip-data.csv                    202728202 gewHE2LoMYJkXXiCvvKeSA==-4    14-NOV-23 12.56.16.590000000 PM GMT    16-NOV-23 08.12.36.561000000 AM GMT
bike-trips/2013-11-Citi-Bike-trip-data.csv                    131891356 YXUIOFESJ3REzZ6DUo6xYA==-2    14-NOV-23 12.56.22.564000000 PM GMT    16-NOV-23 08.12.51.456000000 AM GMT
bike-trips/2013-12-Citi-Bike-trip-data.csv                     86622375 trkBH1X6tMaJh+wy2Emb3g==-2    14-NOV-23 12.54.20.143000000 PM GMT    16-NOV-23 08.19.59.071000000 AM GMT
gl/trans/gl_trans_20230323_100005_990544_000000.parquet         4080742 vKMTV6Qz68q5xBLBEnYMmQ==      23-MAR-23 10.00.10.524000000 AM GMT    23-MAR-23 10.00.10.524000000 AM GMT
gl/trans/gl_trans_20230323_103005_805517_000000.parquet         4036019 i4j9C63XPLqpGcEEbPzqnw==      23-MAR-23 10.30.10.242000000 AM GMT    23-MAR-23 10.30.10.242000000 AM GMT
gl/trans/gl_trans_20230323_110006_443909_000000.parquet         4104642 g3iubo3JA8yZx0kAlzePuQ==      23-MAR-23 11.00.11.243000000 AM GMT    23-MAR-23 11.00.11.243000000 AM GMT

10 rows selected.
```

This post shows how you can easily build an alternative PLSQL table function that provides
additional object level attributes like Storage Tier or Archival State, and use it instead
of `DBMS_CLOUD.LIST_OBJECTS`. And while the function is quite simple, you can use the same
concept to query many other OCI resources.


# __Solution__

## Approach

The solution is to build a table function `LIST_OBJECTS_EXTENDED` that returns list of
objects in the specified bucket, including attributes not available via
`DBMS_CLOUD.LIST_OBJECTS`. The function will use the following features:

* __OCI PLSQL SDK__ to query and manage Oracle Infrastructure resources from PLSQL code in
the database. By using OCI PLSQL SDK, the function can retrieve all attributes of objects
not available via DBMS_CLOUD. Note that OCI PLSQL SDK is preinstalled on Oracle Autonomous
Database - Serverless.

* __Table Function__ to produce a collection of rows that can be queried like a physical
database table, i.e., by specifying the function in the FROM clause of a SELECT query.

Note I used OCI PLSQL SDK in a previous post [How to Get OCI Resource Utilization in
PLSQL](https://jakubillner.github.io/2022/10/07/How-to-Get-OCI-Utilization-in-PLSQL.html),
however I did not returned results as a table function.



## Code

The complete code of the table function `LIST_OBJECTS_EXTENDED` is below.

```
create or replace function list_objects_extended (
  p_namespace_name in varchar2,
  p_bucket_name in varchar2,
  p_credential_name in varchar2,
  p_region in varchar2
) return dbms_cloud_oci_object_storage_object_summary_tbl pipelined is
  v_response dbms_cloud_oci_obs_object_storage_list_objects_response_t;
  v_next_start_with varchar2(4000) := null;
  v_page_count number := 0;
  v_record_count number := 0;
begin
  -- Loop over pages
  loop
    -- Call ListObjects API via PLSQL SDK to get a single page
    v_page_count := v_page_count+1;
    v_response := dbms_cloud_oci_obs_object_storage.list_objects(
      namespace_name => p_namespace_name,
      bucket_name => p_bucket_name,
      region => p_region,
      credential_name => p_credential_name,
      l_start => v_next_start_with,
      fields => 'name,etag,size,timeCreated,md5,timeModified,storageTier,archivalState'
    );
    -- Raise an exception if the API call was not successful
    if (v_response.status_code <> 200) then
      raise_application_error(-20000,'Call to DBMS_CLOUD_OCI_OBS_OBJECT_STORAGE.LIST_OBJECTS returned status '||v_response.status_code);
    end if;
    -- Loop over objects in the page and pipe them out
    for v_object_index in v_response.response_body.objects.first .. v_response.response_body.objects.last loop
      v_record_count := v_record_count+1;
      pipe row (v_response.response_body.objects(v_object_index));
    end loop;
    -- Exit if there are no more pages
    v_next_start_with := v_response.response_body.next_start_with;
    exit when v_next_start_with is null;
  end loop;
  return;
end list_objects_extended;
/
```

* Parameters specify mandatory information such as Namespace, Bucket, Region, and
Credential, needed by by the PLSQL API. The credential must be either based on the API Key
of an OCI user, or it must correspond to resource principal - `OCI$RESOURCE_PRINCIPAL`.

* Return type must be of table type, so that the function can be used as pipelined function.
The function uses table type `DBMS_CLOUD_OCI_OBJECT_STORAGE_OBJECT_SUMMARY_TBL` defined in the
PLSQL SDK. The `PIPELINED` keyword defines the function as pipelined function.

* The function uses PLSQL SDK procedure `DBMS_CLOUD_OCI_OBS_OBJECT_STORAGE.LIST_OBJECTS`
to retrieve list of objects in a bucket.

* As most OCI API calls, the procedure uses pagination to handle large number of objects.
If there are multiple pages, the procedure returns `NEXT_START_WITH`, which has to be used
with `L_START` parameter of the next procedure invocation. Note that single call returns
maximum of 1000 objects.

* Output from the function is provided via `PIPE ROW` statement. A single `PIPE ROW` emits
one row to the output. `RETURN` statement must not return any value.


## Usage

The table function `LIST_OBJECTS_EXTENDED` is used in the FROM part of the SELECT statement,
similarly to `DBMS_CLOUD.LIST_OBJECTS`.

Example 1 - Retrieve list of objects

```
SQL> select name, l_size, storage_tier, time_created, time_modified
  2  from list_objects_extended(
  3    p_namespace_name => '<namespace>',
  4    p_bucket_name => '<bucket name>',
  5    p_credential_name => 'OCI$RESOURCE_PRINCIPAL',
  6    p_region => 'uk-london-1'
  7  )
  8* where rownum <= 10;

NAME                                                             L_SIZE STORAGE_TIER    TIME_CREATED                           TIME_MODIFIED
__________________________________________________________ ____________ _______________ ______________________________________ ______________________________________
20231101T132937Z_20231101T142749Z.0.log                         3785999 Standard        28-NOV-23 03.00.47.715000000 PM GMT    28-NOV-23 03.00.47.715000000 PM GMT
bike-trips/2013-07-Citi-Bike-trip-data.csv                    164438561 Standard        14-NOV-23 12.53.23.193000000 PM GMT    16-NOV-23 08.11.43.538000000 AM GMT
bike-trips/2013-08-Citi-Bike-trip-data.csv                    195523200 Standard        14-NOV-23 12.53.28.660000000 PM GMT    16-NOV-23 08.12.03.630000000 AM GMT
bike-trips/2013-09-Citi-Bike-trip-data.csv                    201965642 Standard        14-NOV-23 12.39.00.468000000 PM GMT    16-NOV-23 08.12.19.380000000 AM GMT
bike-trips/2013-10-Citi-Bike-trip-data.csv                    202728202 Standard        14-NOV-23 12.56.16.590000000 PM GMT    16-NOV-23 08.12.36.561000000 AM GMT
bike-trips/2013-11-Citi-Bike-trip-data.csv                    131891356 Standard        14-NOV-23 12.56.22.564000000 PM GMT    16-NOV-23 08.12.51.456000000 AM GMT
bike-trips/2013-12-Citi-Bike-trip-data.csv                     86622375 Standard        14-NOV-23 12.54.20.143000000 PM GMT    16-NOV-23 08.19.59.071000000 AM GMT
gl/trans/gl_trans_20230323_100005_990544_000000.parquet         4080742 Standard        23-MAR-23 10.00.10.524000000 AM GMT    23-MAR-23 10.00.10.524000000 AM GMT
gl/trans/gl_trans_20230323_103005_805517_000000.parquet         4036019 Standard        23-MAR-23 10.30.10.242000000 AM GMT    23-MAR-23 10.30.10.242000000 AM GMT
gl/trans/gl_trans_20230323_110006_443909_000000.parquet         4104642 Standard        23-MAR-23 11.00.11.243000000 AM GMT    23-MAR-23 11.00.11.243000000 AM GMT

10 rows selected.
```

Example 2 - Count objects by Storage Tier

```
SQL> select
  2    storage_tier,
  3    count(*) as objects_count,
  4    sum(l_size) as objects_size
  5  from list_objects_extended(
  6    p_namespace_name => '<namespace>',
  7    p_bucket_name => '<bucket name>',
  8    p_credential_name => 'OCI$RESOURCE_PRINCIPAL',
  9    p_region => 'uk-london-1'
 10  )
 11* group by storage_tier;

STORAGE_TIER       OBJECTS_COUNT    OBJECTS_SIZE
_______________ ________________ _______________
Standard                     756      4029379495
```


# __Summary__

This post shows how you can easily use OCI PLSQL SDK and table function to query objects
in the Object Storage and to overcome the limitations of the DBMS_CLOUD.LIST_OBJECTS
function.

More importantly, you can use the same concept to query and analyze any other OCI resource from
within Autonomous Database. For example, you could define functions for the following queries:

* Object Storage - list versions of objects
* Object Storage - show metadata of objects
* Object Storage - list buckets in a compartment
* Compute - list VM instances in a compartment
* Monitoring - retrieve metrics for a given resource
* Logging - search for logs and return results as a table


# __Resources__

* OCI PLSQL SDK functions and types for OCI Object Storage: [OCI PLSQL SDK for OCI Object Storage](https://docs.oracle.com/en-us/iaas/pl-sql-sdk/doc/object-storage-package.html).
* OCI API Reference Documentation for Object Storage ListObjects: [Object Storage - ListObjects](https://docs.oracle.com/en-us/iaas/api/#/en/objectstorage/20160918/Object/ListObjects).
* Oracle Database 19c Data Cartridge Developer's Guide - Pipelined Functions: [Using Pipelined and Parallel Table Functions](https://docs.oracle.com/en/database/oracle/oracle-database/19/addci/using-pipelined-and-parallel-table-functions.html).
* Oracle Database 19c PL/SQL Language Reference - Pipelined Functions: [Chaining Pipelined Table Functions for Multiple Transformations](https://docs.oracle.com/en/database/oracle/oracle-database/19/lnpls/plsql-optimization-and-tuning.html#GUID-ED557894-EC08-47E0-A629-0E4AEDDBB77B).

