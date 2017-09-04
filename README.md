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
