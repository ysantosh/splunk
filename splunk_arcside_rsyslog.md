#### Requirement
Collect logs and send it to arcsite. Need to use heavy forwarder as there may be many system and hence need to use heavy forwarder as aggregator.
#### Suggestion1:
Add an output from a Splunk indexer in outputs.conf in a tcpout- stanza while setting the sendCookedData = false to send raw data events not processed by Splunk. This is a setting designed for use in sending to third-party systems, like ArcSight or other SIM tools.

### Streaming Data from Splunk to ArcSight
Splunk can stream real-time data to ArcSight in two ways: by forking off raw events at index-time or by forwarding transformed events at search-time. Index-time output can be used only for data that is natively supported by ArcSight, and events must be routed to the appropriate ArcSight Connector. The index-time method is less flexible but simple to setup and maintain. Search-time output offers the maximum flexibility – the full Splunk search language can be used to select the subset of events to be forwarded – with the added ability to configure how an event should be re-formatting before it’s released. The two methods can also be combined; for example, Splunk can collect multiple data streams from a Linux server, such as CPU metrics, changes to the passwd file, and updates to the audit log, then route only syslog events (e.g. the audit log) at index-time, while using search-time routing for a subset of the other events (e.g. processes that are consuming more than 50% of CPU), sending those as CEF.

#### Index-time output

To configure index-time output, an extra output processor can be inserted into the index-time pipeline (the pipeline handles events from when Splunk receives them to when they are written to disk). Two output processors are available: syslog and tcpout. The tcpout processor sends completely raw events over tcp; it is the same processor that a lightweight forwarder uses when sending data to a Splunk indexer. The syslog processor adds a RFC 5424 compliant header to events before it sends them over TCP or UDP.
The following example configures a syslog and tcpout processor, and routes all events to both. To insert a processor, the user must modify outputs.conf in the correct location, as described here: http://www.splunk.com/base/Documentation/latest/Admin/Aboutconfigurationfiles. For other examples, including an example of conditionally routing subsets of data to different places, see: http://www.splunk.com/base/Documentation/latest/Admin/Forwarddatatothird-partysystems. Note that comments must be removed to constitute a valid config:
```
[syslog] defaultGroup=syslog_receiver indexAndForward=true type=udp
[tcpout]
defaultGroup=raw_tcp_receiver
indexAndForward=true
sendCookedData=false receiver
[syslog:syslog_receiver] server=my.ip.or.name:port
[tcpout:raw_tcp_receiver] server=my.ip.or.name:port2
#initiates the syslog output processor
#specifies the target group name where all events will be sent #forks events so a copy is indexed as well as forwarded #specifies whether to send syslog over TCP or UDP
#initiates the tcp output processor.
#a copy of events will be sent here as well as to the syslog processor above #forks events so a copy is indexed as well as forwarded
#leaves the data untouched, otherwise will be optimized for a Splunk
#defines the syslog target group referenced above #specifying a comma separated list here will load-balance
#defines the tcp target group referenced above #specifying a comma separated list here will load balance
```
#### Search-time output
For non-standard formats such as application logs Splunk can convert data to CEF in real time and export it to an ArcSight Connector. This allows Splunk to help the user avoid building his own FlexConnector. The CEF-output framework functions as follows:
• First, the real-time search API is used to inspect events just before they are indexed. Because the full Splunk search language works here, it is easy to intelligently filter what gets forwarded.
• Once an event occurs, the framework reformats it into CEF (or other defined format). To structure the output, information about the event must be made available by using Splunk functionality such as field extraction and lookups. Fields extracted to the Splunk Common Information Model3 are automatically translated to standard CEF fields. Lookups enrich events with the CEF meta-information, like ‘Device Vendor’ and ‘Product’. For help formatting events as CEF, please see the concluding section of this document.
• Finally, the event is sent to an ArcSight Connector. The default is to send the message as syslog to a Syslog Connector, but Splunk can also write the event to a local file for a software-based CEF Connector to read.
As an example, this is the CEF definition for DHCP events:
CEF:0|Microsoft|DHCP Server||{EventID}|{EventName}|Unknown|cn1={leases expired} cn2={leases deleted} cs4={MAC Vendor Prefix} cs5={Ethernet Vendor} rt={Date, Time} src={Address} shost={HostName} smac={sourceMAC}
The CEF output framework is invoked like this:
splunk cmd ./rtoutput.py -t CEF -S ‘CEF.Connector.ip’ –P ‘port’ -I "dhcpd request | fields *"
The flag ‘–t’ is set to ‘CEF’, which means the command will output in CEF (‘KV’ is also supported, which will output in key-value pairs.) ‘-S’ and ‘-P’ specify the CEF Connector destination (‘-f’ is supported for output to a file). ‘-l’ allows for interactive Splunk password prompts. The real-time search ‘dhcpd request | fields *’ looks at matching events and request for extraction on all known fields, which must include all necessary fields in the “Extension” portion of the CEF, such as src and smac. A Splunk lookup table must be configured to enrich events with the ‘EventId’ and ‘EventName’ fields.
An example result is: Jul 14 11:42:14 splunkindexer CEF:0|Linux|Dhcp Server|1.0|15|Lease requested|5|smac=001B63CDA156 rt=1279132934 src=192.168.0.1 cs5=apple computer inc cs4=001B63
A word on scalability of the integration: The speed at which Splunk can stream data can overwhelm a single ArcSight Connector. Furthermore, Splunk’s real-time search works in distributed mode, so running the search-time output framework from one place will pull events from all Splunk indexers in the distributed environment. To scale the output for ArcSight, either restrict the subset of events Splunk will forward, or set up a cluster of Connectors that Splunk can load-balance across.
3 See http://www.splunk.com/base/Documentation/latest/Knowledge/UnderstandandusetheCommonInformationModel for additional information.
