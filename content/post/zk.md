+++
title = "zookeeper introduction with expiration explanation"
date = "2016-11-24T14:02:41+08:00"
tags = ["zookeeper"]
categories = ["programming"]
menu = ""
banner = "banners/zk.jpg"
+++
Introduction of Apache zookeeper,and some pitfalls from the session expiration expained here.
<!--more-->


<!--more-->
# Zookeeper

## Start

```
zkServer.sh start

```

## Config
- config location, `conf/zoo.cf` by default.contents of the config file.
- ** tickTime **, It is used to do heartbeats and the minimum session timeout will be twice the tickTime.
- ** dataDir **, the location to store the in-memory database snapshots and, unless specified otherwise, the transaction log of updates to the database.
- ** clientPort **, the port to listen for client connections.

## Connect

- by `zkCli.sh -server 127.0.0.1:2181`,or because the `127.0.0.1:2181` is the default server,we can do it by just `zkCli.sh` by default. the `zkCli.sh` is located in the `ZK_PATH/bin/` directory.

## Commands

- `ls` , `ls /` or `ls /xxx/yyy` command to see what the directory looks like.
- `create`, `create /zk_test my_data`,this creates a new znode(ie. /zk_test) and associates the string "my_data" with the node.
- `get` , `get /zk_test`,this returns the data which was associated with the znode .
- `set` , `set /zk_test new_data` ,this will modify the data associated with zk_test from my_data to new_data.
- `delete`, `delete /zk_test` ,this will delete the node zk_test.

## Cluster

- a minimum of three servers are required for clustering.
- it's strongly recommended that you have an odd number of servers.
- cluster config file example
	
	```bash
	tickTime=2000
	dataDir=/var/lib/zookeeper
	clientPort=2181
	initLimit=5
	syncLimit=2
	server.1=zoo1:2888:3888
	server.2=zoo2:2888:3888
	server.3=zoo3:2888:3888
	```
- the `initLimit` is timeouts zookeeper uses to limit the length of time the zookeeper servers in quorum have to connect to a leader.
- the `syncLimit` limits timeout interval for messaging between leader and followers,it is n times of the tickTime. for the above setting,the syncLimit is 2*2000=4000ms.

- both `initLimit` and `syncLimit` use the unit of time based on `tickTime`, `initLimit` is `5*2000ms` and `syncLimit` is `2*2000ms`.

- `server.x`list the servers  that make up the zookeeper service. When the server starts up, it knows which server it is by looking for the file `myid` in the data directory. That file has the contains the server number, in ASCII.
- `server.x=zoo1:2888:3888`,the server ports of 2888 and 3888 meanings: peers use the former port to connect to other peers.Such a connection is necessary so that peers can communicate. the second port is for leader election.

## Ephemeral Node
- the ephemeral znode will be automatically deleted when a session expires or when the client disconnects.
- it can be created with the `create` command with the `-e` flag.
- *No any children nodes can be added into `Ephemeral node`*.
- example:
	
	```bash
	create -e /hello "hellonode"
	//when the client disconnects or the session expires,the znode will be removed.
	```
- **the node is not removed instantly with the client disconnection,it is detected by the session expiration**.

## Four Letter Word Command

- ZooKeeper responds to a small set of commands. Each command is composed of four letters. You issue the commands to ZooKeeper via telnet or nc, at the client port.

- Command list:
	- `conf`, Print details about serving configuration.
	- `cons`, List full connection/session details for all clients connected to this server. Includes information on numbers of packets received/sent, session id, operation latencies, last operation performed, etc.
	- `crst`,client-reset, Reset connection/session statistics for all connections.
	- `srst`,server-reset, Reset server statistics.
	- `dump`, Lists the outstanding sessions and ephemeral nodes. **This only works on the leader**.
	- `envi`, Print details about serving environment.
	- `ruok`, Tests if server is running in a non-error state. The server will respond with imok if it is running. Otherwise it will not respond at all.
	- `srvr`, Lists full details for the server.
	- `stat`,Lists brief details for the server and connected clients.
	- `wchs`, Lists brief information on watches for the server.
	- `wchc`, Lists detailed information on watches for the server, by session. This outputs a list of sessions(connections) with associated watches (paths). Note, depending on the number of watches this operation may be expensive (ie impact server performance), use it carefully.`
	- `dirs` (version>=3.5.1),Shows the total size of snapshot and log files in bytes.
	- `wchp`, Lists detailed information on watches for the server, by path. This outputs a list of paths (znodes) with associated sessions. Note, depending on the number of watches this operation may be expensive (ie impact server performance), use it carefully.
	- `mntr`, Outputs a list of variables that could be used for monitoring the health of the cluster.
	- `isro`,  Tests if server is running in read-only mode. The server will respond with "ro" if in read-only mode or "rw" if not in read-only mode.
	- `gtmk`,Gets the current trace mask as a 64-bit signed long value in decimal format. See stmk for an explanation of the possible values.
	- `stmk`, Sets the current trace mask. The trace mask is 64 bits, where each bit enables or disables a specific category of trace logging on the server. Log4J must be configured to enable TRACE level first in order to see trace logging messages. The bits of the trace mask correspond to the following trace logging categories.
	
	```
		0b0000000000	Unused, reserved for future use.
		0b0000000010	Logs client requests, excluding ping requests.
		0b0000000100	Unused, reserved for future use.
		0b0000001000	Logs client ping requests.
		0b0000010000	Logs packets received from the quorum peer that is the current leader, excluding ping requests.
		0b0000100000	Logs addition, removal and validation of client sessions.
		0b0001000000	Logs delivery of watch events to client sessions.
		0b0010000000	Logs ping packets received from the quorum peer that is the current leader.
		0b0100000000	Unused, reserved for future use.
		0b1000000000	Unused, reserved for future use.
		
	```
 - All remaining bits in the 64-bit value are unused and reserved for future use. Multiple trace logging categories are specified by calculating the bitwise OR of the documented values. The default trace mask is 0b0100110010. Thus, by default, trace logging includes client requests, packets received from the leader and sessions.To set a different trace mask, send a request containing the stmk four-letter word followed by the trace mask represented as a 64-bit signed long value. This example uses the Perl pack function to construct a trace mask that enables all trace logging categories described above and convert it to a 64-bit signed long value with big-endian byte order. The result is appended to stmk and sent to the server using netcat. The server responds with the new trace mask in decimal format.

	```bash
			$ perl -e "print 'stmk', pack('q>', 0b0011111010)" | nc localhost 2181
			250
	```


- New in 3.5.0: The AdminServer is an embedded Jetty server that provides an HTTP interface to the four letter word commands. By default, the server is started on port 8080, and commands are issued by going to the URL "/commands/[command name]", e.g., http://localhost:8080/commands/stat. The command response is returned as JSON. Unlike the original protocol, commands are not restricted to four-letter names, and commands can have multiple names; for instance, "stmk" can also be referred to as "set_trace_mask". To view a list of all available commands, point a browser to the URL /commands (e.g., http://localhost:8080/commands). See the AdminServer configuration options for how to change the port and URLs.
 
 The AdminServer is enabled by default, but can be disabled by either:

 Setting the zookeeper.admin.enableServer system property to false.
Removing Jetty from the classpath. (This option is useful if you would like to override ZooKeeper's jetty dependency.)
Note that the TCP four letter word interface is still available if the AdminServer is disabled.


## Expiration With Pitfalls
- the client is not removed instantly ,that's why the `tickTime`, `initLimit` and `syncLimit` exists.
- about the expiration and client state transitions: described here:

	> http://zookeeper.apache.org/doc/r3.4.8/zookeeperProgrammers.html#ch_zkSessions
	
- the session timeout is negotiated by the client and the server.the server provides a session timeout range.the minimum and the maxmum.the minimum is `2*tickTime` and the maximum is `20*syncLimit*tickTime`.if the client specify a value,the server will respond according to the client value:

	```bash
	clientTimeout<minimum ,return the minimum 
	clientTimeout>=minimum && clientTimeout<=maximum,return clientTimeout
	clientTimeout> maximum,return maximum
	```
- **the session timeout negotiated by the server and the client will finally work on some senarios,like the ephemeral znode removal.If the session timeout val is too small,the ephemeral znode might be removed due to the network unstability.If the timeout val is too big,after the connection was lost,the znode will still be visible for other clients for a relatively long time.so the time should be set carefully**.
- the `zkCli.sh` can specify the client session timeout value to negotiate with the server with the flag `-timeout`.
 example:
 
 	```bash
 		$ ./zkCli.sh -timeout 1000
 		...
 		2016-10-23 23:10:32,046 [myid:] - INFO  [main:ZooKeeper@438] - Initiating client connection, connectString=localhost:2181 sessionTimeout=1000 watcher=org.apache.zookeeper.ZooKeeperMain$MyWatcher@3d555cf5
Welcome to ZooKeeper!
2016-10-23 23:10:32,066 [myid:] - INFO  [main-SendThread(localhost:2181):ClientCnxn$SendThread@1032] - Opening socket connection to server localhost/127.0.0.1:2181. Will not attempt to authenticate using SASL (unknown error)
JLine support is enabled
2016-10-23 23:10:32,073 [myid:] - INFO  [main-SendThread(localhost:2181):ClientCnxn$SendThread@876] - Socket connection established to localhost/127.0.0.1:2181, initiating session
2016-10-23 23:10:32,083 [myid:] - INFO  [main-SendThread(localhost:2181):ClientCnxn$SendThread@1299] - Session establishment complete on server localhost/127.0.0.1:2181, sessionid = 0x157f21695650000, negotiated timeout = 4000
	```

---
# EOF
