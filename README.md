# sqlstream-diagnostics

* [Diagnostic Tools for SQLstream](#diagnostic-tools-for-sqlstream)
  * [Trace File](#trace-file)
  * [Syslog](#syslog)
  * [Backtrace](#backtrace)
  * [jstack thread dumps](#jstack-thread-dumps)
* [SQLstream Telemetry](#sqlstream-telemetry)
  * [Visualizing the execution graph](#visualizing-the-execution-graph)
  * [Identifying a deadlock](#identifying-a-deadlock)
  * [Identifying top resource consumers](#identifying-top-resource-consumers)
  * [Notes](#notes)
* [Monitoring s-Server](#monitoring-s-server)
  

# Diagnostic Tools for SQLstream

## Trace file

Normally found in /var/log/sqlstream (if you have installed as root) or in `$SQLSTREAM_HOME/trace`, the Trace.log file is rotated; the latest should be Trace.log.0.

You can configure the level of tracing, size and number of trace files, etc in the Trace.properties file, normally in the same directory.

See also https://docs.sqlstream.com/building-applications/using-trace-files-to-troubleshoot/. 

## Syslog
This may be located at `/var/log/syslog` or `/var/log/messages` depending on your platform O/S. Some messages from s-Server are reported to the syslog rather than to Trace.log. 

One common cause of s-Server failure is memory exhaustion. If your server runs close to zero free memory, the O/S will pick a candidate process to kill. That candidate will normally be the biggest memory consumer – which is usually s-Server. In that case the syslog will include a message referencing the OOM (out of memory) killer.

This article https://www.cyberciti.biz/faq/linux-log-files-location-and-how-do-i-view-logs-files/ describes how to find and read syslog.

## Backtrace
In some circumstances s-Server scheduler will report a "backtrace" – a dump of the state of the scheduler and the execution plans it is managing. The backtrace is sent to the Trace.log file.

You can request a backtrace by sending the USR1 signal to the s-Server process:
```
PID=$(cat /var/run/sqlstream/jvm.pid)
kill -usr1 $PID
```

The format of the backtrace is hard to read – but the SQLstream support team can assist.

## jstack thread dumps

The Java utility jstack can give a thread dump:
```
jstack <options> $PID <filename>
```
Use these options:
*	-l for a long listing
*	-m for a mixed stack trace (including native stack frames)
*	-F to force a response. 
See [this Oracle document](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr016.html) for more information and see also `man jstack`. If you have trouble finding jstack, try:
```
. /etc/sqlstream/environment
$JAVA_HOME/bin/jstack <options>
```
More tools to add: kill -3 (thread dump); hprof; etc
-XX:ErrorFile=./hs_err_pid<$pid>.log

# Telemetry
SQLstream supports a telemetry API – see https://docs.sqlstream.com/administration-guide/telemetry/. 

You can use the Telemetry API functions to:
*	View execution plans
*	Identify bottlenecks and deadlocks
*	Identify which parts of the execution plan are consuming the most cpu or memory

The telemetry scripts mentioned here are available at https://github.com/NigelThomas/telemetry/.

## Visualizing the execution graph
In versions since 7.0.4 we provide a graphical snapshot of the execution graph (see https://docs.sqlstream.com/integrating-sqlstream/webagent/#telemetry-request). 

For earlier versions, a similar graph can be generated using a `telemetry.sql` script and post-processing the resulting file using the `dot` program (part of the graphviz package) – either on the s-Server host, or elsewhere. The graphviz package is included on recent SQLstream appliances (docker images, VMs, Amazon / Azure machine instance) or can be installed using `yum` or `apt`.

## Identifying a deadlock
In the execution plan, we say a deadlock is present when an upstream operator P is ready to send data to the next operator Q, but operator Q is not taking that data.

In the execution graph, this shows as P and its upstream nodes being in OVF (overflow) status (highlighted in red), while Q is in UND (underflow, green). The deadlock is then likely to be at Q.

Using the `telemetry.sql` visualization makes these blockages much easier to locate.

You can also find them using a SQL query (`telemetry-blockers.sql`):
```
SELECT ovf.node_id as blocked_node_id, ovf.sched_state as blocked_state
     , und.node_id as blocking_node_id, und.sched_state as blocking_state
     , und.source_sql blocking_sql
FROM (
    SELECT * from TABLE(getStreamOperatorInfo(0,1)) 
    WHERE last_exec_result = 'UND'
) und
JOIN (
    SELECT * FROM TABLE(getStreamOperatorInfo(0,1))
    WHERE last_exec_result = 'OVF'
    ) ovf
ON ','||ovf.output_nodes||',' LIKE '%,'||cast(und.node_id AS VARCHAR(5))||',*' 
;
```

## Identifying top resource consumers
We can use SQL to identify the top consumers of memory and cpu. 

### Resources by stream graph
See https://docs.sqlstream.com/administration-guide/telemetry/#stream-graph-virtual-table-column-types-and-definitions for column definitions.

We start by analyzing at stream graph level.  Each graph corresponds to a running statement; in a production system that generally means a pump, although there may also be client sessions (a remote client inserting or reading data); there will be a session for reading the telemetry api, and one or more internal sessions (one is populating the `ALL_TRACE` stream, for example).

For cpu time (approximately) use `total_execution_time`:
```
SELECT graph_id, total_execution_time, net_input_rows, source_sql
FROM TABLE(getStreamGraphInfo(0,1))
ORDER BY total_execution_time DESC
```
For memory, replace `total_execution_time` with `net_memory_bytes` (or `max_memory_bytes`).

The `source_sql` can be spread over multiple lines of the output so you may wish to leave it out from the list – you can use `statement_id` instead, and look up the SQL test from `sys_boot.mgmt.statements_view`.

### Resources by stream operator
See https://docs.sqlstream.com/administration-guide/telemetry/#stream-nodes-virtual-table-column-types-and-definitions-getstreamoperatorinfo for column information.
```
SELECT node_id, net_execution_time, net_input_rows
FROM TABLE(getStreamOperatorInfo(0,1))
ORDER BY net_execution_time DESC
```
In versions up to and including 7.0.3, the net_input_rows and net_input_bytes columns are not populated on every stream operator; but they are collected for native streams.

You may find that the most resource-consuming stream operator is not a part of the most resource-consuming stream graph.

## Notes
As always, if you are tuning you should concentrate on:
•	The biggest consumers of cpu, memory, i/o
•	Where you can make the most difference
Change one thing, then re-test. 

# Monitoring s-Server
Some ideas for monitoring s-Server are presented here: https://docs.sqlstream.com/administration-guide/monitoring-overview/

It is helpful to have one or more health checks that tell you as soon as there is a problem before your customers notice.
