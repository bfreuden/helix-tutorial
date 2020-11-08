

Per doc, Apache Helix is a generic cluster management framework used for the automatic management of partitioned, 
replicated and distributed resources hosted on a cluster of nodes. 
Helix automates reassignment of resources in the face of node failure and recovery, cluster expansion, and reconfiguration.

Official documentation: https://helix.apache.org/

- [Helix use cases](#helix-use-cases)
- [Installation](#installation)
  * [Installing Zookeeper](#installing-zookeeper)
  * [Checking Zookeeper installation](#checking-zookeeper-installation)
    + [Using zkCli](#using-zkcli)
    + [Using Zoonavigator](#using-zoonavigator)
- [Helix Cluster setup](#helix-cluster-setup)
  * [Creating a cluster](#creating-a-cluster)
  * [Other cluster-related operations](#other-cluster-related-operations)
  * [Adding nodes to the cluster](#adding-nodes-to-the-cluster)
  * [Declaring a resource in the cluster](#declaring-a-resource-in-the-cluster)
    + [Preamble](#preamble)
    + [Let's do it](#let-s-do-it)
  * [Ideal state & external view](#ideal-state---external-view)
  * [Assigning partitions to nodes](#assigning-partitions-to-nodes)
    + [Rebalancing](#rebalancing)
    + [Effect on the ideal state](#effect-on-the-ideal-state)
- [Controllers and participants](#controllers-and-participants)
  * [Controller](#controller)
    + [Starting a controller](#starting-a-controller)
    + [Effect on the external view](#effect-on-the-external-view)
  * [Participants](#participants)
    + [Starting a participant](#starting-a-participant)
    + [Effect on the external view](#effect-on-the-external-view)
    + [Starting all participants](#starting-all-participants)
- [Failover](#failover)
  * [Setup a watch on the external view](#setup-a-watch-on-the-external-view)
  * [Gently terminating a participant](#gently-terminating-a-participant)
  * [Brutally killing a participant](#brutally-killing-a-participant)
  * [Gently terminating the Helix controller and a participant](#gently-terminating-the-helix-controller-and-a-participant)
  * [No single point of failure](#no-single-point-of-failure)
- [Implementing a participant](#implementing-a-participant)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>


# Helix use cases

Quote from an article written by a LinkedIn engineer: *Helix is widely used at LinkedIn, not only for distributed storage systems but also for streaming processes, search infrastructure, and analytics platforms.*

Full article here: https://engineering.linkedin.com/blog/2017/02/building-venice-with-apache-helix

# Installation

It is important to understand that Helix is "only a jar" and a set of scripts. 
You might want to run dedicated Helix *controllers* (provided with Helix), but it is not a requirement.
You might as well choose to embed Helix in your own application.

However, Helix relies on Zookeeper so you will need to run a Zookeeper ensemble.

Download (you can choose your mirror here: https://helix.apache.org/0.9.8-docs/download.cgi) and unpack Helix:
```bash
wget https://apache.mediamirrors.org/helix/0.9.8/binaries/helix-core-0.9.8-pkg.tar
tar xvf helix-core-0.9.8-pkg.tar
cd helix-core-0.9.8/bin
```

## Installing Zookeeper

Helix delivers a standalone Zookeeper for test purpose:
```bash
./start-standalone-zookeeper.sh 2181
```

You can alternatively use docker to install a Zookeeper for test purpose (it is convenient if you want to use Zookeeper CLI):
```bash
docker pull zookeeper:3.4.14
docker run -d --network host --name zookeeper zookeeper:3.4.14
```

For running Zookeeper in production see:

https://zookeeper.apache.org/doc/r3.4.10/zookeeperAdmin.html

https://docs.confluent.io/current/zookeeper/deployment.html

## Checking Zookeeper installation

### Using zkCli

Let's open a Zookeeper CLI (zkCli) right now (we'll get back at it later on):
```bash
docker exec -it zookeeper zkCli.sh
```
And let's see Zookeeper root nodes (zkCli):
```
[zk: localhost:2181(CONNECTED) 0] ls /
[zookeeper]
```

### Using Zoonavigator

You can also install a web interface to browse zookeeper data:
```bash
docker pull elkozmon/zoonavigator
docker run -d --network host -e HTTP_PORT=9000  --name zoonavigator elkozmon/zoonavigator:latest
```
The server will be running on http://localhost:9000/

Just put ``localhost:2181`` in the connection string and click connect.

# Helix Cluster setup

If you want to use Helix, it probably means you want to build a distributed application, 
and instances of your distributed application will form a cluster.

It is important to understand that this paragraph is not about creating a cluster of Helix servers
(you will not connect to many machines in order to install Helix).

This paragraph is about defining the cluster of your own application: 
* how many instances of your application,
* the different kinds of resources (e.g. databases) it is dealing with, 
* and how they are distributed on different nodes.

In this paragraph you will use Helix concepts to do so, and you will only use Helix admin command-line  
to store data in Zookeeper.  

## Creating a cluster

Creating a Helix cluster must be understood like this: declaring a cluster in Zookeeper.

Creating the cluster using the command line:
```bash
./helix-admin.sh --zkSvr localhost:2181 --addCluster MYCLUSTER
```
This command will only create a ``MYCLUSTER`` root znode in Zookeeper with a Helix-specific layout of znodes.

Let's have a look at the result using zkCli:
```
[zk: localhost:2181(CONNECTED) 1] ls /
[MYCLUSTER, zookeeper]
[zk: localhost:2181(CONNECTED) 2] ls /MYCLUSTER 
[EXTERNALVIEW, LIVEINSTANCES, CONTROLLER, IDEALSTATES, STATEMODELDEFS, PROPERTYSTORE, INSTANCES, CONFIGS]
```

You can also create a cluster programmatically (remember: Helix is a jar):
```java
ZKHelixAdmin admin = new ZKHelixAdmin("localhost:2181");
admin.addCluster("MYCLUSTER");
admin.close();
```

See: https://helix.apache.org/0.9.8-docs/Tutorial.html

## Other cluster-related operations

You can list clusters managed by Helix:
```bash
./helix-admin.sh --zkSvr localhost:2181 --listClusters
```
```
Existing clusters:
MYCLUSTER
```

To drop the cluster (this will remove the ``MYCLUSTER`` root znode from Zookeeper, and that's the only consequence so far):
```bash
./helix-admin.sh --zkSvr localhost:2181 --dropCluster MYCLUSTER
```

For the moment the Helix cluster has no node (zkCli):
```
[zk: localhost:2181(CONNECTED) 3] ls /MYCLUSTER/INSTANCES 
[]
```
## Adding nodes to the cluster

Adding nodes to the cluster must be understood like this: declaring them in Zookeeper.

Adding nodes to the cluster using the command line:
```bash
./helix-admin.sh --zkSvr localhost:2181  --addNode MYCLUSTER localhost:12913
./helix-admin.sh --zkSvr localhost:2181  --addNode MYCLUSTER localhost:12914
./helix-admin.sh --zkSvr localhost:2181  --addNode MYCLUSTER localhost:12915
```

When you start reading Helix docs, this notion of ``hostname:port`` is somehow confusing.
You start wondering about a server you'll have to run on those ports in your Helix-powered application. 
Actually it is not required to have any server running on those ports.
So for the moment just consider that those are identifier. 

Instances have appeared in Zookeeper (zkCli):
```
[zk: localhost:2181(CONNECTED) 5] ls /MYCLUSTER/INSTANCES
[localhost_12913, localhost_12914, localhost_12915]
```

We can use helix admin to list nodes (called instances) in the cluster:
```bash
./helix-admin.sh --zkSvr localhost:2181  --listClusterInfo MYCLUSTER
```
```
Existing resources in cluster MYCLUSTER:
Instances in cluster MYCLUSTER:
localhost_12913
localhost_12914
localhost_12915
```
Note that ```localhost_12913``` is the **instance identifier** of the node.

## Declaring a resource in the cluster

### Preamble

It would be common for a distributed database to have shards and handling shards in a master/slave fashion.
Some nodes would be masters for a given shard, and slave for other shards.

Master/slave is one way of seeing things, but there might be others.
Helix comes with predefined **state models** (zkCli):
```
[zk: localhost:2181(CONNECTED) 6] ls /MYCLUSTER/STATEMODELDEFS 
[MasterSlave, SchedulerTaskQueue, Task, STORAGE_DEFAULT_SM_SCHEMATA, LeaderStandby, OnlineOffline]
```
We can see ``MasterSlave``, ``LeaderStandby`` and ``OnlineOffline`` for instance.

But you can define your own state model: https://helix.apache.org/0.9.8-docs/tutorial_state.html.

At this point you are probably guessing that Helix will decide the placement of the shards (and their states: master or slave).
You would be right: Helix can do that for you... but that behavior, called **rebalance mode**, is actually customizable.

For more information, see: 

https://helix.apache.org/0.9.8-docs/tutorial_rebalance.html

https://helix.apache.org/0.9.8-docs/tutorial_user_def_rebalancer.html

### Let's do it

Defining a resource must be understood like this: declaring it in Zookeeper.

Now let's define a resource (e.g. a database) with 6 partitions (e.g. with 6 shards) behaving in a master/slave fashion:
```bash
./helix-admin.sh --zkSvr localhost:2181 --addResource MYCLUSTER myDB 6 MasterSlave
```
This can be done programmatically:
```java
ZKHelixAdmin admin = new ZKHelixAdmin("localhost:2181");
admin.addResource("MYCLUSTER", "myDB", 6, "MasterSlave", "SEMI_AUTO");
admin.close();
```

## Ideal state & external view

Defining a resource will create an **ideal state** for it in Zookeeper (zkCli):
```
[zk: localhost:2181(CONNECTED) 19] get /MYCLUSTER/IDEALSTATES/myDB
{
  "id" : "myDB",
  "mapFields" : {
  },
  "listFields" : {
  },
  "simpleFields" : {
    "IDEAL_STATE_MODE" : "AUTO",
    "NUM_PARTITIONS" : "6",
    "REBALANCE_MODE" : "SEMI_AUTO",
    "REBALANCE_STRATEGY" : "DEFAULT",
    "REPLICAS" : "0",
    "STATE_MODEL_DEF_REF" : "MasterSlave",
    "STATE_MODEL_FACTORY_NAME" : "DEFAULT"
  }
}
```
As stated in Helix concepts (https://helix.apache.org/Concepts.html), 
Helix is based on the idea that a given task/resource has the following attributes:
* a location (e.g. it is available on Node N1)
* a state (e.g. it is running, stopped, etc...)

The IdealState is the mapping of resources/tasks to a location and a state.

For the moment the ideal state is quite empty because requires an explicit Helix action called **rebalancing** to be actually computed.
We'll get to it in the next paragraph.

The listResourceInfo command of the helix admin CLI will show the ideal state:
```bash
./helix-admin.sh --zkSvr localhost:2181 --listResourceInfo MYCLUSTER myDB
```
```
IdealState for myDB:
{
  "id" : "myDB",
  "mapFields" : {
  },
  "listFields" : {
  },
  "simpleFields" : {
    "IDEAL_STATE_MODE" : "AUTO",
    "NUM_PARTITIONS" : "6",
    "REBALANCE_MODE" : "SEMI_AUTO",
    "REBALANCE_STRATEGY" : "DEFAULT",
    "REPLICAS" : "0",
    "STATE_MODEL_DEF_REF" : "MasterSlave",
    "STATE_MODEL_FACTORY_NAME" : "DEFAULT"
  }
}

No externalView for myDB
```
The **external view** will be computed by the **Helix controller** from the ideal state, 
the available **Helix participants** and their state. 

For the moment the **external view** is undefined because we need a **Helix controller** and **Helix participants**. 
We will start them in a few minutes.

In an ideal world **ideal state** and **external view** would be equal... but in the real world things will eventually go wrong. 

## Assigning partitions to nodes

This step, called rebalancing, will consist in assigning partitions to nodes.

Rebalancing must be understood like this: computing the ideal state and saving it in Zookeeper.

Before doing so you must decide how many replicas you want for each partition.
This is called the **replication factor** of the cluster.

### Rebalancing

Let's go for 3 replicas: 
```bash
./helix-admin.sh --zkSvr localhost:2181 --rebalance MYCLUSTER myDB 3
```

At this point there is a notion you need to be aware of: the ``MasterSlave`` state model 
defines that you can have only have 1 **Master** and that the number of slave will derive from
the replication factor of the cluster.

Here it means you'll have 1 **Master** and 2 **Slaves** (3 replicas) for each partition.  

### Effect on the ideal state

Let's have a look at the ideal state once again:
```bash
./helix-admin.sh --zkSvr localhost:2181 --listResourceInfo MYCLUSTER myDB
``` 
Before showing the output of the admin command, let's remind that Helix maps tasks to location and state.
This is expressed using the following notation:
```
"TASK_NAME" : {
  "LOCATION" : "STATE"
}
```
In the output below, we will see partitions named after the resource (with a numeral suffix, e.g. ``myDB_0``), 
instance identifiers of nodes (e.g. ``localhost_12913``) and states (e.g. ``SLAVE``).
We will also see that, in this ideal state, we have:
* 2 master partitions per node
* 2 slave partitions per node
* no node with 2 replicas of the same partition (it does not make sense for a node to be a master and a slave for a given partition)
```
IdealState for myDB:
{
  "id" : "myDB",
  "mapFields" : {
    "myDB_0" : {
      "localhost_12913" : "SLAVE",
      "localhost_12914" : "MASTER",
      "localhost_12915" : "SLAVE"
    },
    "myDB_1" : {
      "localhost_12913" : "SLAVE",
      "localhost_12914" : "SLAVE",
      "localhost_12915" : "MASTER"
    },
    "myDB_2" : {
      "localhost_12913" : "MASTER",
      "localhost_12914" : "SLAVE",
      "localhost_12915" : "SLAVE"
    },
    "myDB_3" : {
      "localhost_12913" : "SLAVE",
      "localhost_12914" : "SLAVE",
      "localhost_12915" : "MASTER"
    },
    "myDB_4" : {
      "localhost_12913" : "MASTER",
      "localhost_12914" : "SLAVE",
      "localhost_12915" : "SLAVE"
    },
    "myDB_5" : {
      "localhost_12913" : "SLAVE",
      "localhost_12914" : "MASTER",
      "localhost_12915" : "SLAVE"
    }
  },
  "listFields" : {
    "myDB_0" : [ "localhost_12914", "localhost_12915", "localhost_12913" ],
    "myDB_1" : [ "localhost_12915", "localhost_12914", "localhost_12913" ],
    "myDB_2" : [ "localhost_12913", "localhost_12915", "localhost_12914" ],
    "myDB_3" : [ "localhost_12915", "localhost_12914", "localhost_12913" ],
    "myDB_4" : [ "localhost_12913", "localhost_12914", "localhost_12915" ],
    "myDB_5" : [ "localhost_12914", "localhost_12915", "localhost_12913" ]
  },
  "simpleFields" : {
    "IDEAL_STATE_MODE" : "AUTO",
    "NUM_PARTITIONS" : "6",
    "REBALANCE_MODE" : "SEMI_AUTO",
    "REBALANCE_STRATEGY" : "DEFAULT",
    "REPLICAS" : "3",
    "STATE_MODEL_DEF_REF" : "MasterSlave",
    "STATE_MODEL_FACTORY_NAME" : "DEFAULT"
  }
}

No externalView for myDB
```

Note that there is still no external view because, so far, we only have modified data into Zookeeper.
Helix is indeed using Zookeeper as a **Cluster State Metadata Store**.

The next step is to throw some JVMs into play... they will interact with the metadata store, listen to data changes, etc...

# Controllers and participants

The architecture documentation (https://helix.apache.org/Architecture.html) states that Helix will divide those
JVMs based on their responsibilities:
* Participants: JVMs that actually host the distributed resources
* Spectators: JVMs that simply observe the current state of each Participant (Routers, for example, need to know the instance on which a partition is hosted and its state in order to route the request to the appropriate endpoint)
* Controllers: JVMs that observes and controls the Participant nodes. It is responsible for coordinating all transitions in the cluster and ensuring that state constraints are satisfied while maintaining cluster stability.

## Controller

### Starting a controller 
Let's start a Helix controller:
```bash
./run-helix-controller.sh --zkSvr localhost:2181 --cluster MYCLUSTER
```

This can be done programmatically (see https://helix.apache.org/0.9.8-docs/tutorial_controller.html):
```java
HelixManager manager = HelixManagerFactory.getZKHelixManager(
    "MYCLUSTER", "mycontroller", InstanceType.CONTROLLER, "localhost:2181"
);
manager.connect();
GenericHelixController controller = new GenericHelixController();
manager.addInstanceConfigChangeListener(controller);
manager.addInstanceConfigChangeListener(controller);
manager.addLiveInstanceChangeListener(controller);
manager.addIdealStateChangeListener(controller);
// this one is stated in the doc but does not compile (to be checked)
// manager.addExternalViewChangeListener(controller);
manager.addControllerListener(controller);
```

### Effect on the external view

Then let's have a look at the external view of our resource:
```bash
./helix-admin.sh --zkSvr localhost:2181 --listResourceInfo MYCLUSTER myDB  | grep -B 0 -A 1000 ExternalView
```
```
ExternalView for myDB:
{
  "id" : "myDB",
  "mapFields" : {
  },
  "listFields" : {
  },
  "simpleFields" : {
    "BUCKET_SIZE" : "0",
    "IDEAL_STATE_MODE" : "AUTO",
    "NUM_PARTITIONS" : "6",
    "REBALANCE_MODE" : "SEMI_AUTO",
    "REBALANCE_STRATEGY" : "DEFAULT",
    "REPLICAS" : "3",
    "STATE_MODEL_DEF_REF" : "MasterSlave",
    "STATE_MODEL_FACTORY_NAME" : "DEFAULT"
  }
}
```
We can see that the external view is defined now.

The external view is what the Helix controller has been able to do given the current constraints. 
Since we don't have any participant yet, the controller can't do much... so ``mapFields`` is empty 
(but we would like its content to eventually look like the content of the ideal state).

So let's start a participant. 

## Participants

This is where you (a developer planning to use Helix) come into play. 

As stated in the architecture documentation, a participant is a JVM that actually hosts the distributed resources.
In other words that JVM is your application's (since Helix doesn't know if you're implementing a distributed database or something).

In real life you will have to embed a participant your JVM, and let your application react to
orders like this (order that are given to your application through callbacks set on the participant):
* serve myDB_0 partition as a master
* serve myDB_1 partition as a slave
* drop myDB_3 partition (because it has been reassigned to another node)

Helix is delivered with a mock participant though. It's very nice to see how Helix is working before writing code.

### Starting a participant

So let's start a mock participant.

Please note that the host and port must match one of the nodes we previously added to the cluster:
```bash
./start-helix-participant.sh --zkSvr localhost:2181 --cluster MYCLUSTER --host localhost --port 12913 --stateModelType MasterSlave
```
It is unclear to me why the state model must be defined when starting the participant.

We can see that a participant will add a live instance znode into Zookeeper (zkCli):
```
[zk: localhost:2181(CONNECTED) 21] ls /MYCLUSTER/LIVEINSTANCES
[localhost_12913]
```

### Effect on the external view

We can also see that the Helix controller has assigned all partitions (in the ``MASTER`` state) to the ``localhost_12913`` instance:
```bash
./helix-admin.sh --zkSvr localhost:2181 --listResourceInfo MYCLUSTER myDB  | grep -B 0 -A 1000 ExternalView | grep -B 1000 -A0 simpleFields
```
```
{
  "id" : "myDB",
  "mapFields" : {
    "myDB_0" : {
      "localhost_12913" : "MASTER"
    },
    "myDB_1" : {
      "localhost_12913" : "MASTER"
    },
    "myDB_2" : {
      "localhost_12913" : "MASTER"
    },
    "myDB_3" : {
      "localhost_12913" : "MASTER"
    },
    "myDB_4" : {
      "localhost_12913" : "MASTER"
    },
    "myDB_5" : {
      "localhost_12913" : "MASTER"
    }
  },
  "listFields" : {
  },
  "simpleFields" : {
```

### Starting all participants

Let's add the second participant:
```bash
./start-helix-participant.sh --zkSvr localhost:2181 --cluster MYCLUSTER --host localhost --port 12914 --stateModelType MasterSlave
```

Let's see the external view:
```
./helix-admin.sh --zkSvr localhost:2181 --listResourceInfo MYCLUSTER myDB  | grep -B 0 -A 1000 ExternalView | grep -B 1000 -A0 simpleFields
```
```
ExternalView for myDB:
{
  "id" : "myDB",
  "mapFields" : {
    "myDB_0" : {
      "localhost_12913" : "SLAVE",
      "localhost_12914" : "MASTER"
    },
    "myDB_1" : {
      "localhost_12913" : "SLAVE",
      "localhost_12914" : "MASTER"
    },
    "myDB_2" : {
      "localhost_12913" : "MASTER",
      "localhost_12914" : "SLAVE"
    },
    "myDB_3" : {
      "localhost_12913" : "SLAVE",
      "localhost_12914" : "MASTER"
    },
    "myDB_4" : {
      "localhost_12913" : "MASTER",
      "localhost_12914" : "SLAVE"
    },
    "myDB_5" : {
      "localhost_12913" : "SLAVE",
      "localhost_12914" : "MASTER"
    }
  },
  "listFields" : {
  },
  "simpleFields" : {

```

We can see that the Helix controller has done 2 things:
* evenly dispatched ``MASTER`` partitions on our 2 instances
* created ``SLAVE`` replicas on both instances (making sure a ``SLAVE`` is on a different instance as the ``MASTER``).

Let's start the third participant:
```bash
./start-helix-participant.sh --zkSvr localhost:2181 --cluster MYCLUSTER --host localhost --port 12915 --stateModelType MasterSlave
```
Let's see the external view:
```
{
  "id" : "myDB",
  "mapFields" : {
    "myDB_0" : {
      "localhost_12913" : "SLAVE",
      "localhost_12914" : "MASTER",
      "localhost_12915" : "SLAVE"
    },
    "myDB_1" : {
      "localhost_12913" : "SLAVE",
      "localhost_12914" : "SLAVE",
      "localhost_12915" : "MASTER"
    },
    "myDB_2" : {
      "localhost_12913" : "MASTER",
      "localhost_12914" : "SLAVE",
      "localhost_12915" : "SLAVE"
    },
    "myDB_3" : {
      "localhost_12913" : "SLAVE",
      "localhost_12914" : "SLAVE",
      "localhost_12915" : "MASTER"
    },
    "myDB_4" : {
      "localhost_12913" : "MASTER",
      "localhost_12914" : "SLAVE",
      "localhost_12915" : "SLAVE"
    },
    "myDB_5" : {
      "localhost_12913" : "SLAVE",
      "localhost_12914" : "MASTER",
      "localhost_12915" : "SLAVE"
    }
  },
  "listFields" : {
  },
  "simpleFields" : {

```
Once again the Helix controller has:
* evenly dispatched ``MASTER`` replicas on 3 instances
* taken into consideration the expected replication factor: now we have 2 ``SLAVE`` replicas for each partition

Now the external view is the same as the ideal state.

Now let's observe how Helix will handle failures.

# Failover


## Setup a watch on the external view

Setup a watch (refreshed every second) on the external view and let it run:
```bash
watch -n1 "sh -c './helix-admin.sh --zkSvr localhost:2181 --listResourceInfo MYCLUSTER myDB  | grep -B 0 -A 1000 ExternalView | grep -B 1000 -A0 simpleFields'"
```

## Gently terminating a participant

Let's terminate the last participant:
```bash
kill "$(ps -ef | grep java | grep 12915 | awk -F ' ' '{print $2}')"
```
The watch will show you that the Helix controller will almost instantly update the external view:
```
ExternalView for myDB:
{
  "id" : "myDB",
  "mapFields" : {
    "myDB_0" : {
      "localhost_12913" : "SLAVE",
      "localhost_12914" : "MASTER"
    },
    "myDB_1" : {
      "localhost_12913" : "SLAVE",
      "localhost_12914" : "MASTER"
    },
    "myDB_2" : {
      "localhost_12913" : "MASTER",
      "localhost_12914" : "SLAVE"
    },
    "myDB_3" : {
      "localhost_12913" : "SLAVE",
      "localhost_12914" : "MASTER"
    },
    "myDB_4" : {
      "localhost_12913" : "MASTER",
      "localhost_12914" : "SLAVE"
    },
    "myDB_5" : {
      "localhost_12913" : "SLAVE",
      "localhost_12914" : "MASTER"
    }
  },
  "listFields" : {
  },
  "simpleFields" : {
```

Now start your participant once again:
```bash
./start-helix-participant.sh --zkSvr localhost:2181 --cluster MYCLUSTER --host localhost --port 12915 --stateModelType MasterSlave
```
The watch will show you that the Helix controller will almost instantly update the external view 
to get back to the previous (and ideal) state:
```
ExternalView for myDB:
{
  "id" : "myDB",
  "mapFields" : {
    "myDB_0" : {
      "localhost_12913" : "SLAVE",
      "localhost_12914" : "MASTER",
      "localhost_12915" : "SLAVE"
    },
    "myDB_1" : {
      "localhost_12913" : "SLAVE",
      "localhost_12914" : "SLAVE",
      "localhost_12915" : "MASTER"
    },
    "myDB_2" : {
      "localhost_12913" : "MASTER",
      "localhost_12914" : "SLAVE",
      "localhost_12915" : "SLAVE"
    },
    "myDB_3" : {
      "localhost_12913" : "SLAVE",
      "localhost_12914" : "SLAVE",
      "localhost_12915" : "MASTER"
    },
    "myDB_4" : {
      "localhost_12913" : "MASTER",
      "localhost_12914" : "SLAVE",
      "localhost_12915" : "SLAVE"
    },
    "myDB_5" : {
      "localhost_12913" : "SLAVE",
      "localhost_12914" : "MASTER",
      "localhost_12915" : "SLAVE"
    }
  },
  "listFields" : {
  },
  "simpleFields" : {
```

## Brutally killing a participant

Now let's be brutal and kill (-9) the last participant:
```bash
kill -9 "$(ps -ef | grep java | grep 12915 | awk -F ' ' '{print $2}')"
```
Now it takes a few tens of seconds for the Helix controller to detect the failure:
```
ExternalView for myDB:
{
  "id" : "myDB",
  "mapFields" : {
    "myDB_0" : {
      "localhost_12913" : "SLAVE",
      "localhost_12914" : "MASTER"
    },
    "myDB_1" : {
      "localhost_12913" : "SLAVE",
      "localhost_12914" : "MASTER"
    },
    "myDB_2" : {
      "localhost_12913" : "MASTER",
      "localhost_12914" : "SLAVE"
    },
    "myDB_3" : {
      "localhost_12913" : "SLAVE",
      "localhost_12914" : "MASTER"
    },
    "myDB_4" : {
      "localhost_12913" : "MASTER",
      "localhost_12914" : "SLAVE"
    },
    "myDB_5" : {
      "localhost_12913" : "SLAVE",
      "localhost_12914" : "MASTER"
    }
  },
  "listFields" : {
  },
  "simpleFields" : {

```
This behavior is probably controlled by the session timeout of the Zookeeper client created by Helix (defaulting to 30 seconds).

The ``zk.session.timeout`` system property can be used to change the session timeout.

When a participant started with ``-Dzk.session.timeout=4000`` is brutally killed, the Helix controller can detect it in 4 seconds:
```bash
JAVA_OPTS=-Dzk.session.timeout=4000 ./start-helix-participant.sh --zkSvr localhost:2181 --cluster MYCLUSTER --host localhost --port 12915 --stateModelType MasterSlave
```

It is probably worth mentioning that Zookeeper requires that the client session timeout be a minimum of 2 times the tickTime (as set in the server configuration) and a maximum of 20 times the tickTime
(see https://zookeeper.apache.org/doc/r3.4.14/zookeeperProgrammers.html).

Zookeeper tickTime seems to be around 2 or 3 seconds, so we can imagine 4 seconds is the best one can achieve.

Start your participant once again (the Helix controller detects it and act accordingly):
```bash
./start-helix-participant.sh --zkSvr localhost:2181 --cluster MYCLUSTER --host localhost --port 12915 --stateModelType MasterSlave
```

## Gently terminating the Helix controller and a participant

Now let's terminate the Helix controller:
```
bash
kill "$(ps -ef | grep java | grep HelixControllerMain | awk -F ' ' '{print $2}')"
```
You can do that. Nothing wrong will happen (as long as nothing wrong happens to your participants in the meantime).
In fact the external view is still OK:
```
ExternalView for myDB:
{
  "id" : "myDB",
  "mapFields" : {
    "myDB_0" : {
      "localhost_12913" : "SLAVE",
      "localhost_12914" : "MASTER",
      "localhost_12915" : "SLAVE"
    },
    "myDB_1" : {
      "localhost_12913" : "SLAVE",
      "localhost_12914" : "SLAVE",
      "localhost_12915" : "MASTER"
    },
    "myDB_2" : {
      "localhost_12913" : "MASTER",
      "localhost_12914" : "SLAVE",
      "localhost_12915" : "SLAVE"
    },
    "myDB_3" : {
      "localhost_12913" : "SLAVE",
      "localhost_12914" : "SLAVE",
      "localhost_12915" : "MASTER"
    },
    "myDB_4" : {
      "localhost_12913" : "MASTER",
      "localhost_12914" : "SLAVE",
      "localhost_12915" : "SLAVE"
    },
    "myDB_5" : {
      "localhost_12913" : "SLAVE",
      "localhost_12914" : "MASTER",
      "localhost_12915" : "SLAVE"
    }
  },
  "listFields" : {
  },
  "simpleFields" : {
```

Now let's terminate the last participant (once again... poor guy...):
```bash
kill "$(ps -ef | grep java | grep 12915 | awk -F ' ' '{print $2}')"
```

Now things started going wrong because the external view remained the same.

Let's start Helix controller once again:
```bash
./run-helix-controller.sh --zkSvr localhost:2181 --cluster MYCLUSTER
```
Your watch will show that the external view is instantly updated (the Helix controller is starting very fast):
```
ExternalView for myDB:
{
  "id" : "myDB",
  "mapFields" : {
    "myDB_0" : {
      "localhost_12913" : "SLAVE",
      "localhost_12914" : "MASTER"
    },
    "myDB_1" : {
      "localhost_12913" : "SLAVE",
      "localhost_12914" : "MASTER"
    },
    "myDB_2" : {
      "localhost_12913" : "MASTER",
      "localhost_12914" : "SLAVE"
    },
    "myDB_3" : {
      "localhost_12913" : "SLAVE",
      "localhost_12914" : "MASTER"
    },
    "myDB_4" : {
      "localhost_12913" : "MASTER",
      "localhost_12914" : "SLAVE"
    },
    "myDB_5" : {
      "localhost_12913" : "SLAVE",
      "localhost_12914" : "MASTER"
    }
  },
  "listFields" : {
  },
  "simpleFields" : {
```
Of course, you can restart the participant once again and the controller will detect it then get back to the ideal state.

## No single point of failure

At first sight it looks like the controller is a single point of failure but it is not the case.

The controller documentations (https://helix.apache.org/0.9.8-docs/tutorial_controller.html) states that you can start many controllers.
Only one will be active at a time through a leader election process.
You can even embed a controller in your participant JVMs.

# Implementing a participant

The nice thing is implementing a participant is as easy as this:
```java
HelixManager manager = HelixManagerFactory.getZKHelixManager(
        "MYCLUSTER",
        "localhost_12915",
        InstanceType.PARTICIPANT,
        "localhost:2181");
try {
    StateMachineEngine stateMach = manager.getStateMachineEngine();
    MasterSlaveStateModelFactory stateModelFactory = new MasterSlaveStateModelFactory() {
        
        // there are other callbacks, let's show 2 of them: 
        public void onBecomeSlaveFromMaster(Message message, NotificationContext context) {
            System.out.println("transitioning from " + message.getFromState() + " to "
                    + message.getToState() + " for " + message.getPartitionName());
            // act on your application accordingly

        }

        public void onBecomeMasterFromSlave(Message message, NotificationContext context) {
            System.out.println("transitioning from " + message.getFromState() + " to "
                    + message.getToState() + " for " + message.getPartitionName());
            // act on your application accordingly
        }


    };
    stateMach.registerStateModelFactory(STATE_MODEL_NAME, stateModelFactory);
    manager.connect();
    // run your application
} catch (Exception e) {
    e.printStackTrace();
} finally {
    manager.disconnect();
}
```