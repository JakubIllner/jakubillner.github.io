
![Intro Picture](/images/2022-10-07-how-to-get-oci-utilization-in-plsql/rybnik.jpg)

# __Introduction__

A nice thing in Oracle Cloud Infrastructure (OCI) and most other public clouds is that
many useful services are available automatically as part of the cloud platform. One of
these services is OCI Monitoring, which collects metrics data from other OCI services,
aggregates them and notifies users when resource specific alarms are triggered. Metrics
data from OCI Monitoring may be consumed via OCI Console, OCI CLI, or OCI API/SDK.

During a recent testing of JSON documents in Autonomous Data Warehouse (ADW) I wanted to
correlate observed loading and query throughputs with the CPU utilization of OCI resources
used in the testing - ADW and Compute instance. CPU utilization for both ADW and Compute
is automatically tracked by OCI Monitoring so I could easily get the utilization data from
there.

Furthermore, as I used ADW also for aggregation and evaluation of testing results, I
needed to consume the utilization data from within ADW, using SQL/PLSQL language.
Fortunately, one of the OCI SDK's is PLSQL SDK, which is automatically available on all
ADW instances.

__This post shows how easy and convenient it is to configure and use OCI PLSQL SDK for
retrieving resource utilization (and other) metrics from OCI Monitoring.__


# __OCI Console__

As I never used OCI Monitoring programmatically before, I decided first to explore how the
metric data look in OCI Console. I opened required resource (ADW or Compute Instance in my
case) in OCI Console, clicked on _Metrics_ on the left side of the screen, selected the
chart with required metric - _CPU Utilization_ - and from _Options_ selected action _View
Query in Metrics Explorer_.

The Metrics Explorer screen contains most of the information that is needed for querying
OCI Monitoring programmatically:

![Metrics Explorer](/images/2022-10-07-how-to-get-oci-utilization-in-plsql/metrics-explorer.jpg)

* Start time - date and time from which the metrics data are provided.
* End time - date and time to which the metrics data are provided.
* Compartment - compartment with the resource we are interested in.
* Metric namespace - namespace corresponding to the type of resource (`oci_autonomous_database` for ADB, `oci_computeagent` for Compute Instances).
* Metric name - name of the metric we want to retrieve (`CpuUtilization` in our case).
* Interval - frequency of measurements (`1m` is 1 minute, `5m` is 5 minutes, `1h` is 1 hour etc.).
* Statistic - statistical function that is applied to measurements (`mean()` for average, `P50()` for 50th percentile etc.).
* Resource Id - OCID of the resource we are interested in.
* Query - text of query used by OCI Monitoring to retrieve metrics data. It consists of Metric Name, Interval, Resource Id, and Statistical function.


# __OCI CLI__

With the above information I tested the metrics retrieval via OCI CLI. I used OCI Cloud
Shell because it contains OCI CLI preinstalled, including credentials and other parameters
needed for connecting to OCI API.

```
 oci monitoring metric-data summarize-metrics-data \
> --compartment-id <Compartment OCID> \
> --namespace oci_autonomous_database \
> --query-text 'CpuUtilization[1m]{resourceId = "<Resource OCID>"}.mean()' \
> --start-time "2022-10-01T09:46:13.000Z" \
> --end-time "2022-10-01T09:53:07.000Z"
```

This command returns a JSON document with array of measurements corresponding to the
required frequency (`1m` - 1 minute). The metrics are the same as ones presented in OCI
Console Metrics Explorer.

```
{
  "data": [
    {
      "aggregated-datapoints": [
        {
          "timestamp": "2022-10-01T09:47:00+00:00",
          "value": 96.443680391495
        },
        {
          "timestamp": "2022-10-01T09:48:00+00:00",
          "value": 97.2647348798675
        },
        {
          "timestamp": "2022-10-01T09:49:00+00:00",
          "value": 96.5618526785715
        },
        {
          "timestamp": "2022-10-01T09:50:00+00:00",
          "value": 94.7174844249875
        },
        {
          "timestamp": "2022-10-01T09:51:00+00:00",
          "value": 95.631255798542
        },
        {
          "timestamp": "2022-10-01T09:52:00+00:00",
          "value": 94.455463763298
        },
        {
          "timestamp": "2022-10-01T09:53:00+00:00",
          "value": 93.673341208235
        }
      ],
      "compartment-id": "<Compartment OCID>",
      "dimensions": {
        "AutonomousDBType": "ADW",
        "deploymentType": "Shared",
        "displayName": "CpuUtilization",
        "region": "uk-london-1",
        "resourceId": "<Resource OCID>",
        "resourceName": "<Resource Name>"
      },
      "metadata": {},
      "name": "CpuUtilization",
      "namespace": "oci_autonomous_database",
      "resolution": null,
      "resource-group": null
    }
  ]
}
```


# __OCI PLSQL SDK__

Now I understand the shape of metrics data returned by OCI Monitoring, I can use the same
query in ADW database with OCI PLSQL SDK. However, I need to get an average across the
specified interval, not a list of averages with the defined frequency. To do that, I wrote
a simple Function that retrieves metrics datapoints with 1 minute frequency and it
calculates the average from the datapoints.

The function parameters contain all the required parameters for the OCI Monitoring API.

```
create or replace package my_oci_metrics is

function get_average_cpu_utilization (
  p_credential_name  in varchar2,
  p_region           in varchar2,
  p_compartment_id   in varchar2,
  p_namespace_name   in varchar2,
  p_metric_name      in varchar2,
  p_resource_ocid    in varchar2,
  p_start_time       in timestamp with time zone,
  p_end_time         in timestamp with time zone
) return number;

end my_oci_metrics;
/
```

The body of the function uses `dbms_cloud_oci_mn_monitoring.summarize_metrics_data()` to
retrieve the metrics data. After that, it calculates average utiliation from the
collection of datapoints.


```
create or replace package body my_oci_metrics is

function get_average_cpu_utilization (
  p_credential_name  in varchar2,
  p_region           in varchar2,
  p_compartment_id   in varchar2,
  p_namespace_name   in varchar2,
  p_metric_name      in varchar2,
  p_resource_ocid    in varchar2,
  p_start_time       in timestamp with time zone,
  p_end_time         in timestamp with time zone
) return number is
  /* additional parameters */
  c_resource_group   varchar2(200) := null;
  c_frequency        varchar2(200) := '1m';
  c_resolution       varchar2(200) := '1m';
  c_query_text       varchar2(400) := p_metric_name||'['||c_frequency||']{resourceId = "'||p_resource_ocid||'"}.mean()';
  /* variables */
  v_metrics_input         dbms_cloud_oci_monitoring_summarize_metrics_data_details_t;
  v_metrics_data          dbms_cloud_oci_mn_monitoring_summarize_metrics_data_response_t;
  v_aggregated_datapoints dbms_cloud_oci_monitoring_aggregated_datapoint_tbl;
  v_value_average number;
begin
  /* populate input record */
  v_metrics_input := dbms_cloud_oci_monitoring_summarize_metrics_data_details_t(
    namespace      => p_namespace_name,
    query          => c_query_text,
    start_time     => p_start_time,
    end_time       => p_end_time,
    resource_group => c_resource_group,
    resolution     => c_resolution
  );
  /* get utilization data */
  v_metrics_data := dbms_cloud_oci_mn_monitoring.summarize_metrics_data (
    compartment_id                 => p_compartment_id,
    summarize_metrics_data_details => v_metrics_input,
    credential_name                => p_credential_name,
    region                         => p_region
  );
  /* calculate averages */
  if (v_metrics_data.status_code = 200) then
     if (v_metrics_data.response_body.count > 0) then
       v_aggregated_datapoints := v_metrics_data.response_body(1).aggregated_datapoints;
       if (v_aggregated_datapoints.count > 0) then
         select avg(value)
         into v_value_average
         from table(v_aggregated_datapoints);
         return v_value_average;
       end if;
     end if;
  else
     raise_application_error(-20100, 'DBMS_CLOUD_OCI_MN_MONITORING.SUMMARIZE_METRICS_DATA returned status: '||v_metrics_data.status_code);
  end if;
  return null;
end get_average_cpu_utilization;

end my_oci_metrics;
/
```


Note it might be possible to get the same information using the _Resolution_ parameter of
the query, however I was unable to fully understand how _Resolution_ and _Interval_
parameters work together so I decided to use the more manual but (to me) safer method.


# __Calling the Function in SQL__

Finally I can use the new function to retrieve average CPU utilization for the intervals
when my tests were running.


```
select
  scenario_short
, trunc(sum(target_records)/max(elapsed_sec_load),0) as records_per_second
, to_char(min(start_datetime),'YYYY/MM/DD HH24:MI:SS') as start_datetime
, to_char(max(end_datetime),'YYYY/MM/DD HH24:MI:SS') as end_datetime
, trunc(my_oci_metrics.get_average_cpu_utilization (
    p_credential_name  => '<Credential Name>'
  , p_region           => 'uk-london-1'
  , p_compartment_id   => '<Compartment OCID>'
  , p_namespace_name   => 'oci_autonomous_database'
  , p_metric_name      => 'CpuUtilization'
  , p_resource_ocid    => '<Resource OCID>'
  , p_start_time       => min(start_datetime)
  , p_end_time         => max(end_datetime)
),2) as avg_cpu_util
from test_results
group by scenario_short
order by scenario_short
/
```

and the results are here:

```
SCENARIO_SHORT                              RECORDS_PER_SECOND       START_DATETIME         END_DATETIME  AVG_CPU_UTIL
___________________________________________ __________________ ____________________ ____________________ _____________
1. JSON Column                                            1307 2022/10/01 09:33:23  2022/10/01 09:46:09          65.33
2. JSON Column format OSON                                1530 2022/10/01 09:46:13  2022/10/01 09:57:07          84.96
3. JSON Column format OSON compress medium                1320 2022/10/01 09:57:11  2022/10/01 10:09:48          95.28

```


# __OCI PLSQL SDK Prerequisites__

* OCI PLSQL SDK uses a credential defined via `dbms_cloud.create_credential` procedure.
The credential must use either the private key of the OCI user, or it must be based on the
Resource Principal of the ADB database. Credential using Authentication Token does not
work.

* ADB user calling OCI PLSQL SDK must be granted `execute` privilege on all the packages
and types used by the code. In our example it means the following objects:

  * `dbms_cloud_oci_monitoring_summarize_metrics_data_details_t` (type)
  * `dbms_cloud_oci_mn_monitoring_summarize_metrics_data_response_t` (type)
  * `dbms_cloud_oci_monitoring_aggregated_datapoint_tbl` (type)
  * `dbms_cloud_oci_mn_monitoring` (package)



