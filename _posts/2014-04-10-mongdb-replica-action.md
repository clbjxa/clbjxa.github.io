---
layout: post
title: Mongodb Replica Set HA
description:  Replication provides redundancy and increases data availability. With multiple copies of data on different database servers, replication protects a database from the loss of a single server. Replication also allows you to recover from hardware failure and service interruptions. With additional copies of the data, you can dedicate one to disaster recovery, reporting, or backup.
category: operation
---

##Overview
For Mongodb usage and document, please refer to:[Mongdb][Mongdb]	
In this topic, we just simply generalize the below aspects:

* Built the Replica Set solution in lab.
* Test the failover in server and client.
* Test the write and read divide.
* Add the auth function.

## Built the Replica Set solution in lab
Relica Sets uses n mongod to build auto-failover, auto-recovery. Theoretically, we need three mongodb instances, in this case, I will use 2 mongodb+1 arbiter.

(0)Basis Environment

	#node1
	mongod1 : deploy 
	ip : 10.12.7.107 
	port : 27031 
	$ mkdir -p /data/db/0 
	./mongod  --dbpath /data/db/0 --port 27031 --replSet myset

	#node2
	mongod2 : doc 
	ip : 10.12.7.108 
	port : 27032 
	$ mkdir -p /data/db/1 
	./mongod  --dbpath /data/db/1 --port 27032 --replSet myset
	
	#node3
	mongod3 : deploy 
	ip : 10.12.7.107 
	port : 27033 
	$ mkdir -p /data/db/2 
	./mongod  --dbpath /data/db/2 --port 27033 --replSet myset

(1)Create Replica Sets

Select any on mongodb node, using mongo shell to login and execute the below actions:

    >config = {_id: 'myset', members: [ 
        {_id: 0, host: '10.12.7.107:27031'}, 
        {_id: 1, host: '10.12.7.108:27032'}, 
        {_id: 2, host: '10.12.7.107:27033', arbiterOnly: true}]} 
	> rs.initiate(config) 
	> rs.conf() #check the config info  > rs.staus()
	
Now, The Replica Sets setup over and the relative info will be saved in local db.

(2)Introduce the process

At the same time，there has nobly one primary to accept writting actions。Then async them to other database. Once the current primary killed，the solution will automatically vote for a new primary.


##Test the failover in server and client
(1)At the server, insert test data into primary(27031):

	> use test 
		switched to db test 
	> db.users.insert({name:"terrylc"})
	
You will see the below messages:

	#27031  Tue Oct 18 08:44:07 [FileAllocator] allocating new datafile /data/db/0/test.ns, filling with zeroes... 
	Tue Oct 18 08:44:07 [FileAllocator] done allocating datafile /data/db/0/test.ns, size: 16MB,  took 0.054 secs 
	Tue Oct 18 08:44:07 [FileAllocator] allocating new datafile /data/db/0/test.0, filling with zeroes... 
	Tue Oct 18 08:44:07 [FileAllocator] done allocating datafile /data/db/0/test.0, size: 64MB,  took 0.153 secs 
	Tue Oct 18 08:44:07 [FileAllocator] allocating new datafile /data/db/0/test.1, filling with zeroes... 
	Tue Oct 18 08:44:07 [conn3] building new index on { _id: 1 } for test.users 
	Tue Oct 18 08:44:07 [conn3] done for 0 records 0secs 
	Tue Oct 18 08:44:07 [conn3] insert test.users 213ms 
	Tue Oct 18 08:44:09 [FileAllocator] done allocating datafile /data/db/0/test.1, size: 128MB,  took 1.527 secs #27032  Tue Oct 18 23:43:02 [FileAllocator] allocating new datafile /data/db/1/test.ns, filling with zeroes... 
	Tue Oct 18 23:43:02 [FileAllocator] done allocating datafile /data/db/1/test.ns, size: 16MB,  took 0.054 secs 
	Tue Oct 18 23:43:02 [FileAllocator] allocating new datafile /data/db/1/test.0, filling with zeroes... 
	Tue Oct 18 23:43:03 [FileAllocator] done allocating datafile /data/db/1/test.0, size: 64MB,  took 0.556 secs 
	Tue Oct 18 23:43:03 [FileAllocator] allocating new datafile /data/db/1/test.1, filling with zeroes... 
	Tue Oct 18 23:43:03 [replica set sync] building new index on { _id: 1 } for test.users 
	Tue Oct 18 23:43:03 [replica set sync] done for 0 records 0secs 
	Tue Oct 18 23:43:03 [FileAllocator] done allocating datafile /data/db/1/test.1, size: 128MB,  took 0.166 secs #27033  Tue Oct 18 08:42:22 [ReplSetHealthPollTask] replSet info 10.12.7.107:27031 is up 
	Tue Oct 18 08:42:22 [ReplSetHealthPollTask] replSet member 10.12.7.107:27031 PRIMARY 
	Tue Oct 18 08:42:22 [ReplSetHealthPollTask] replSet info 10.12.7.108:27032 is up 
	Tue Oct 18 08:42:22 [ReplSetHealthPollTask] replSet member 10.12.7.108:27032 RECOVERING 
	Tue Oct 18 08:42:34 [ReplSetHealthPollTask] replSet member 10.12.7.108:27032 SECONDARY

Stop 27031 service, To observe the phenomenon:

    #27032  Tue Oct 18 23:47:47 [conn2] end connection 10.12.7.107:58587 
	Tue Oct 18 23:47:47 [replica set sync] replSet syncThread: 10278 dbclient error communicating with server: 10.12.7.107:27031 
	Tue Oct 18 23:47:48 [ReplSetHealthPollTask] DBClientCursor::init call() failed 
	Tue Oct 18 23:47:48 [ReplSetHealthPollTask] replSet info 10.12.7.107:27031 is down (or slow to respond): DBClientBase::findOne: transport error: 10.12.7.107:27031 query: { replSetHeartbeat: "myset", v: 1, pv: 1, checkEmpty: false, from: "10.12.7.108:27032" } 
	Tue Oct 18 23:47:48 [rs Manager] replSet info electSelf 1 
	Tue Oct 18 23:47:48 [rs Manager] replSet couldn't elect self, only received -9999 votes  	Tue Oct 18 23:47:54 [rs Manager] replSet info electSelf 1 
	Tue Oct 18 23:47:54 [rs Manager] replSet PRIMARY #27033  Tue Oct 18 08:48:52 [conn2] end 	connection 10.12.7.107:43768 
	Tue Oct 18 08:48:53 [conn3] 10.12.7.108:27032 is trying to elect itself but 10.12.7.107:27031 is already primary and more up-to-date 
	Tue Oct 18 08:48:54 [ReplSetHealthPollTask] DBClientCursor::init call() failed 
	Tue Oct 18 08:48:54 [ReplSetHealthPollTask] replSet info 10.12.7.107:27031 is down (or slow to respond): DBClientBase::findOne: transport error: 10.12.7.107:27031 query: 	{ replSetHeartbeat: "myset", v: 1, pv: 1, checkEmpty: false, from: "10.12.7.107:27033" } 
	Tue Oct 18 08:48:59 [conn3] replSet info voting yea for 1 
	Tue Oct 18 08:49:00 [ReplSetHealthPollTask] replSet member 10.12.7.108:27032 PRIMARY 

Mongod 27032 was elected as new primary, in the corresponding terminal，do the below mongo shell:
	
	$ ./mongo localhost:27032 
	> rs.status() 
	{ "set" : "myset", "date" : ISODate("2011-10-18T15:53:01Z"), "myState" : 1, "members" : [ 
        { "_id" : 0, "name" : "10.12.7.107:27031", "health" : 0, "state" : 1, "stateStr" : "(not reachable/healthy)", "uptime" : 0, "optime" : { "t" : 1318952647000, "i" : 1 
            }, "optimeDate" : ISODate("2011-10-18T15:44:07Z"), "lastHeartbeat" : ISODate("2011-10-18T15:47:46Z"), "errmsg" : "socket exception" }, 
        { "_id" : 1, "name" : "10.12.7.108:27032", "health" : 1, "state" : 1, "stateStr" : "PRIMARY", "optime" : { "t" : 1318952647000, "i" : 1 
            }, "optimeDate" : ISODate("2011-10-18T15:44:07Z"), "self" : true 
        }, 
        { "_id" : 2, "name" : "10.12.7.107:27033", "health" : 1, "state" : 7, "stateStr" : "ARBITER", "uptime" : 705, "optime" : { "t" : 0, "i" : 0 
            }, "optimeDate" : ISODate("1970-01-01T00:00:00Z"), "lastHeartbeat" : ISODate("2011-10-18T15:53:00Z") 
        } 
    ], "ok" : 1 
	} 
 
Review the previous inserting recode:

    myset:PRIMARY> db.users.findOne() 
	{ "_id" : ObjectId("4e9d9ec76df8f2f72bd9c45a"), "name" : "terrylc" }
	
If we restart the 27031，the node will be set as secondary

	Tue Oct 18 23:56:46 [ReplSetHealthPollTask] replSet info 10.12.7.107:27031 is up 
	Tue Oct 18 23:56:46 [ReplSetHealthPollTask] replSet member 10.12.7.107:27031 SECONDARY 
	Tue Oct 18 23:56:47 [conn6] I am already primary, 10.12.7.107:27031 can try again once 	I've stepped down  Tue Oct 18 23:56:48 [initandlisten] connection accepted from 	10.12.7.107:44777 #7  Tue Oct 18 23:56:49 [slaveTracking] building new index on { _id: 1 } for local.slaves 
	Tue Oct 18 23:56:49 [slaveTracking] done for 0 records 0secs 
	Tue Oct 18 23:57:03 [conn5] end connection 127.0.0.1:58555

(2)Client Test

[Mongdb]:	http://docs.mongodb.org/manual