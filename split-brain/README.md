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

## How to avoid a split-brain: Observers

A separate configuration is working with observers. Note, that the docker image does not support this out of the box, thus a little hack is required.

```bash
docker-compose -f docker-compose-with-observers.yml up -d
```

The idea is, to have working copy synchronized to the observers, than shut down the original nodes, reconfigure the observers to be full nodes and restart them. Note, that this is UNTESTED.

Finally, stop the cluster by running:

```bash
docker-compose -f docker-compose-with-observers.yml down
```

