# splunk
splunk notes and command

### How to get list of available indexes
The most efficient way to get accurate results is probably:
 
```| eventcount summarize=false index=* | dedup index | fields index```

If you want to include internal indexes, you can use:

```| eventcount summarize=false index=* index=_* | dedup index | fields index```

### btool usage for troubleshooting and finding configuration [btool link](https://docs.splunk.com/Documentation/Splunk/6.6.3/Troubleshooting/Usebtooltotroubleshootconfigurations)
Syntax - conf_file_prefix can be ```server|transforms|distsearch|``` etc

```./splunk cmd btool <conf_file_prefix> list [--debug] ```

Example

```./splunk cmd btool server list```

Adding a debug option will show from which config file the options are generated.

```./splunk cmd btool server list --debug ```

To get a list of stanza only in server.conf 

```./splunk cmd btool server list --debug | grep '\['```

### Find list of forwarders forwarding to splunk cluster
```
index=_internal source=*metrics.log group=tcpin_connections 
| where isnotnull(fwdType)
| eval sourceHost=if(isnull(hostname), sourceHost,hostname) 
| dedup sourceHost
| eval connectionType =case(fwdType=="uf","Universal", fwdType=="lwf", "Lightweight",
fwdType=="full","Heavy")
| rename sourceIp as "Source IP", sourceHost as "Source Host",
connectionType as "Forwarder Type"
| table "Source IP" "Source Host" "Forwarder Type"
```

### Find information about indexes in splunk like rentention period, sizeof index , index location 
#### Find index and index size total 
```
| rest /services/data/indexes | where disabled = 0 | search NOT title = "_*" | stats sum(currentDBSizeMB) as size_MB by title |sort - size_MB
```
#### Find index and index size list per indexer
```
| rest /services/data/indexes | where disabled = 0 | search NOT title = "_*" | stats values(currentDBSizeMB) as size_MB by title | sort - size_MB
```
Note: you can remove ```| search NOT title = "_*" ``` if internal indexes size also needed.

#### Find index rentention period, sizeof index , index location etc
```
| rest /services/data/indexes | where disabled = 0 | search NOT title = "_*" | eval currentDBSizeGB = round( currentDBSizeMB / 1024) | where currentDBSizeGB > 0 | table splunk_server title summaryHomePath_expanded minTime maxTime currentDBSizeGB totalEventCount frozenTimePeriodInSecs coldToFrozenDir maxTotalDataSizeMB | rename minTime AS earliest maxTime AS latest summaryHomePath_expanded AS index_path currentDBSizeGB AS index_size totalEventCount AS event_cnt frozenTimePeriodInSecs AS index_retention coldToFrozenDir AS index_path_frozen maxTotalDataSizeMB AS index_size_max title AS index
```

### Splunk CLI searches [CLI search link](https://docs.splunk.com/Documentation/SplunkCloud/6.6.1/SearchReference/CLIsearchsyntax) 
Example 
```
./splunk search '<searchstring>' -preview true -auth admin:<passwd>

```
#### Check searchhead and indexer status
```
./splunk show cluster-bundle-status
```

```
./splunk list shcluster-member-info -uri http://<SH_server>:8089 -auth <id>:<passwd>
```

#### check certificate information and expire etc and Generate the certificate
```
openssl x509 -in server.pem -noout -text 
Generate the cert
bin/splunk createssl server-cert -d etc/auth/ -n server

```

#### Find the daily ingested 
##### From search head using indexers query
```index=_internal group=thruput name=index_thruput | timechart span=1d sum(kb) AS daily_KB```
##### From search head using licence data
```| rest splunk_server=<server_name> /services/licenser/pools | rename title AS Pool | search [rest splunk_server=<server_name> /services/licenser/groups | search is_active=1 | eval stack_id=stack_ids | fields stack_id] | eval quota=if(isnull(effective_quota),quota,effective_quota) | eval "Used"=round(used_bytes/1024/1024/1024, 3) | search Used="*" | eval "Quota"=round(quota/1024/1024/1024, 3) | fields Pool "Used" "Quota"```
The above query can be seen from splunk DMC - performance

### Upgrade or restart of splunk
#### Stop sequence
* Cluster Master node
* Deployment Manager
* Search head cluster
* Search Peer
* Licence master
#### - Upgrade or do file changes
#### Start sequence
* Licence Master
* Cluster Master node  and put it in maintenance mode
* Start search peer
* Remove maintenance mode on cluster master
* Start search head
* Start Deployment Manager




