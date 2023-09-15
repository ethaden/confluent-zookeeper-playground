# ERRONEOUS configuration for testing

This configuration is erroneous due to having an even number of zookeeper nodes. It might lead to a `split-brain` situation, where two leaders exist.

## Running the demo

Start docker containers, but only three zookeeper nodes for now:

```bash
docker-compose up -d zookeeper-1 zookeeper-2 zookeeper-3 kafka-1 kafka-2 kafka-3
```

Depending on wether or not you have all tools locally installed, you might enter one of the zookeeper containers, just to have all zookeeper CLI tools available:

```bash
docker-compose exec zookeeper-1 /bin/bash
```

Show zookeeper status

```bash
for PORT in 21811 21812 21813 21814 21815 21816; do echo $PORT; (echo stats | nc localhost ${PORT}|grep -E "Mode|current"); done
```

Show brokers

```bash
zookeeper-shell localhost:21811 ls /brokers/ids
```
