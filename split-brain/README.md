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
docker-compose down
```

## How to avoid a split-brain: Observers (not working properly)

An alternative configuration is working with observers. Note, that the docker image does not support this out of the box, thus a little hack is required.

```bash
docker-compose -f docker-compose-with-observers.yml up -d
```

The idea is, to have working copy synchronized to the observers, than shut down the original nodes, reconfigure the observers to be full nodes and restart them. Note, that this is UNTESTED.

Finally, stop the cluster by running:

```bash
docker-compose -f docker-compose-with-observers.yml down
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