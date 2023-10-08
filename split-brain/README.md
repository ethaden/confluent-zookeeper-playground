# ERRONEOUS configuration for testing

This configuration is erroneous due to having an even number of zookeeper nodes. It might lead to a `split-brain` situation, where two leaders exist.

## Running the demo

Start docker containers, but only three zookeeper nodes for now, use this line for the three nodes
`ZOOKEEPER_SERVERS: zookeeper-1:2888:3888;zookeeper-2:2888:3888;zookeeper-3:2888:3888`:

```bash
docker-compose up -d zookeeper-1 zookeeper-2 zookeeper-3
```

Depending on wether or not you have all tools locally installed, you might enter one of the zookeeper containers, just to have all zookeeper CLI tools available:

```bash
docker-compose exec zookeeper-1 /bin/bash
```

Show zookeeper status for all nodes (independent of how many are running)

```bash
for PORT in 21811 21812 21813 21814 21815 21816; do echo $PORT; (echo stats | nc localhost ${PORT}|grep -E "Mode|current"); done
```

With the initial configuration, there should be one leader and two followers.

Now let's try to create a split-brain situation (where there are two leaders).

Update the configuration file by commenting the original `ZOOKEEPER_SERVERS` line and using the one with all six zookeeper nodes, i.e.
`ZOOKEEPER_SERVERS: zookeeper-1:2888:3888;zookeeper-2:2888:3888;zookeeper-3:2888:3888;zookeeper-4:2888:3888;zookeeper-5:2888:3888;zookeeper-6:2888:3888
`.

Then, (re)start all zookeeper nodes, starting with the new ones, then continuing with one previous followers:

```bash
docker-compose up -d zookeeper-4 zookeeper-5 zookeeper-6
docker-compose up zookeeper-(PUT ID OF ANY ORIGINAL FOLLOWER HERE)
```

Potentially, restart the second previous follower as well. You might see two leaders.

Stop the cluster:

```bash
docker-compose down -v
```

## How to avoid a split-brain: Disabled data dir auto-create for new nodes

Let's assume we have three working nodes, nodes 1 and 2 are in data center 1 (DC1) and node 3 (!) is in data center 2 (DC2). We want to avoid the situation that the two nodes in DC1 just update the Kafka configuration with their majority without having to get a vote from the node(s) in DC2.

Thus, we spin up three additional nodes. However, we make sure these nodes do not auto-create an empty namespace (otherweise they could cause a split-brain situation where the three old nodes have different state than the three new nodes).

In this setup, having three nodes in each data center ensures that quorum requires at least 4 nodes. No data center can decide anything without the other. Obviously, this is great for consistency and durability of our metadata, but bad for availability (if one DC is down, nothing works anymore).

Note: We disabled auto-creation of data dirs for nodes 4, 5 and 6. They need to get their initial state from an existing quorum.

Spin up the initial three nodes by running:

```shell
docker-compose -f docker-compose-no-autocreate.yml up -d zookeeper-1 zookeeper-2 zookeeper-3
```

Check their states by running:

```shell
for PORT in 21811 21812 21813 21814 21815 21816; do echo $PORT; (echo stats | nc localhost ${PORT}|grep -E "Mode|current"); done
```

Bring up a fourth node, which is configured just to know the first three and itself (and does NOT auto-create its data dir):

```shell
docker-compose -f docker-compose-no-autocreate.yml up -d zookeeper-4
``

Check the state again:

```shell
for PORT in 21811 21812 21813 21814 21815 21816; do echo $PORT; (echo stats | nc localhost ${PORT}|grep -E "Mode|current"); done
```

The fourth node should have joined the quorum.

Check the current epoch of each node:

```shell
for ID in 1 2 3 4 5 6; do echo -n "$ID: "; docker exec -t zookeeper-${ID} cat /var/lib/zookeeper/data/version-2/currentEpoch; echo ""; done
```

Now that we have four Zookeeper nodes forming a quorum, we can add the addition two nodes.
In the docker-compose file, update the environment variables of nodes 1, 2, 3 and 4 as follows:
Comment the ZOOKEEPER_SERVERS line where only nodes 1, 2 and 3 are present. Uncomment the line where all six nodes are present.
Then, restart the containers, one by one (if you want to be extra-cautious, you might want to restart followers first and the leader last):

```shell
docker-compose -f docker-compose-no-autocreate.yml up -d zookeeper-4
sleep 2
docker-compose -f docker-compose-no-autocreate.yml up -d zookeeper-3
sleep 2
docker-compose -f docker-compose-no-autocreate.yml up -d zookeeper-2
sleep 2
docker-compose -f docker-compose-no-autocreate.yml up -d zookeeper-1
```

Check the state again:

```shell
for PORT in 21811 21812 21813 21814 21815 21816; do echo $PORT; (echo stats | nc localhost ${PORT}|grep -E "Mode|current"); done
```

All nodes should have joined the quorum.

Check the current epoch of each node:

```shell
for ID in 1 2 3 4 5 6; do echo -n "$ID: "; docker exec -t zookeeper-${ID} cat /var/lib/zookeeper/data/version-2/currentEpoch; echo ""; done
```

We expect to have identical values for each node.

You are done. Congratulations. Please be aware that you are resilient against the loss of two nodes in this setup. If three nodes are in one data center and three nodes in the other, a loss of a data center or a network partitioning between them will cause Zookeeper to loose quorum!
The nodes will stop serving requests. Manual interaction would be required in case the missing nodes are not suspected to come back automatically. You'd need to add at least one node or reduce the nodes mentioned in the server list (the latter is preferred!).

### Explanation

We start with a pre-condition: Nodes 1, 2 and 3 are considered pre-existing nodes which have a non-empty znode namespace already. This is the normal result of setting up those three nodes with auto-creation of the data dir enabled. Nodes 4, 5 and 6 are created with auto-creation of the data dir disabled (!). We need to simulate this in docker by customizing the `run command`. For bare-metal server installations of Zookeeper, just disable auto creation by following the Zookeeper documentation, e.g. set the environment variable ZOO_DATADIR_AUTOCREATE_DISABLE to 1 before running the init script or add `-Dzookeeper.datadir.autocreate=false` to the java command line. Then use the `zkServer-initialize.sh` script to do just a basic initialization of the data dir, without creating a znode namespace. Start the new nodes. The expected behavior is that the set the current epoche to `-1` and try to synchronize from existing Zookeeper nodes, but only if those form a quorum!

The number of nodes in the servers list is used to determine the number of nodes required for a quorum. We start by bringing up the nodes 1, 2 and 3 and let them initialize properly. They know nothing about nodes 4, 5 and 6 yet and can thus easily form a quorum. Instead of adding all new servers at once (which would result in six nodes in total, thus our three properly initializd nodes wouldn't have a quorum), we start by adding only one new node, namely node 4. We make sure that node 4 knows only about the existing three nodes and itself.

Nodes 1-5 each know only about 5 nodes, thus the three properly initialized nodes 1, 2 and 3 are used a source for synchronization from nodes 3 and 4. Only after that we update the configuration again.

## How to avoid a split-brain: Hiearchical quorum and disabled auto-create

In this example we want to achieve a hiearchical setup consisting of two groups of Zookeeper nodes (one per data center), where each group consists of three nodes located in the same data center. The main advantage over having three nodes in one and three nodes in the other data center is that this solution is battle-proved and recommended.

Follow the steps described in the last section to spin up six Zookeeper nodes. Then, edit the docker compose file and enable the lines for configuring the groups for each of the nodes. Restart the nodes by running:

```shell
docker-compose -f docker-compose-no-autocreate.yml up -d zookeeper-1
sleep 2
docker-compose -f docker-compose-no-autocreate.yml up -d zookeeper-2
sleep 2
docker-compose -f docker-compose-no-autocreate.yml up -d zookeeper-4
sleep 2
docker-compose -f docker-compose-no-autocreate.yml up -d zookeeper-3
sleep 2
docker-compose -f docker-compose-no-autocreate.yml up -d zookeeper-5
sleep 2
docker-compose -f docker-compose-no-autocreate.yml up -d zookeeper-6
```

## How to avoid a split-brain: Observers (not working properly)

An alternative configuration is working with observers. Note, that the docker image does not support this out of the box, thus a little hack is required.

```bash
docker-compose -f docker-compose-with-observers.yml up -d
```

The idea is, to have working copy synchronized to the observers, than shut down the original nodes, reconfigure the observers to be full nodes and restart them. Note, that this is UNTESTED.

Finally, stop the cluster by running (IMPORTANT: Make sure to specify the option `-v` for the `down` command to erase the volumes created by docker-compose!):

```bash
docker-compose -f docker-compose-with-observers.yml down -v
```

## How to avoid a split-brain: Dynamic reconfiguration with Observers
Another alternative configuration is working with dynamic reconfiguration and observers. Note, that the docker image does not support this out of the box, thus a little hack is required. The configuration is splitted in two config files: One is configured using the environment variables which are interpreted by the docker image startup scripts to create a static configuration file. This static configuration enables dynamic reconfiguration and defines a dynamic reconfiguration file. This dynamic reconfiguration file is mounted as a volume. Each zookeeper node has a separate dynamic configuration file, to allow individual fine-granular modifications. The servers are defined in the dynamic configuration.

We start with a reasonable dynamic configuration, which you can find in `config/zookeeper.properties.dynamic`.
For this demo, we create a copy of that initial file for each zookeeper nodes. The respective copy for each node is mounted into the docker containers. DO NOT CHANGE the copies manually during operation. They are changed by zookeeper when you issue a `reconfig` command.
You can have a look at the files, though. Create the copies by running the following command in the `split-brain` folder:

```bash
for i in 1 2 3 4 5 6; do cp config/zookeeper.properties.dynamic config/zookeeper.properties.dynamic-zk$i; done

```bash
docker-compose -f docker-compose-with-observers-dynamic-reconfig.yml up -d
```

### Step 1
We assume that initially there are three servers with existing data that form a quorum. We already added three observers in the initial configuration.

### Step 2
We now promote on of the observers to a participant dynamically. The quorum will remain intact. In this demo, we run zookeeper-shell in one (any) of the docker containers, for simplicitly (adapt hostname and port as required):

```bash
docker-compose -f docker-compose-with-observers-dynamic-reconfig.yml exec zookeeper-1 zookeeper-shell localhost:21811
```

Show the current configuration:

```bash
config
```

Note that we use three ports in this example:

* Port 2888 is used as usual for follower connections if a node is a leader
* Port 3888 is used during elections
* The client port is customized because we want it to be accessible from the docker host:
  * Node 1 uses client port 21811
  * Node 2 uses client port 21812
  * Node 3 uses client port 21813
  * Node 4 uses client port 21814
  * Node 5 uses client port 21815
  * Node 6 uses client port 21816

In a different shell, run this command to check the zookeeper ensemble:

```bash
for PORT in 21811 21812 21813 21814 21815 21816; do echo $PORT; (echo stats | nc localhost ${PORT}|grep -E "Mode|current"); done
```


Now we promote one of the observers (namely node 5) to be a participant. Run the following in zookeeper-shell (it will update the dynamic configurations on all all zookeeper nodes):

```bash
reconfig -add server.5=zookeeper-5:2888:3888:participant;21815
```

In a different shell, run this command to check the zookeeper ensemble again:

```bash
for PORT in 21811 21812 21813 21814 21815 21816; do echo $PORT; (echo stats | nc localhost ${PORT}|grep -E "Mode|current"); done
```
