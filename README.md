# Manticore Search Docker image

This is the git repo of official [Docker image](https://hub.docker.com/r/manticoresearch/manticore/) for [Manticore Search](https://manticoresearch.com/).

Manticore Search is a powerful free open source search engine with a focus on low latency and high throughput full-text search and high volume stream filtering. It helps thousands of companies from small to large, such as Craigslist, to search and filter petabytes of text data on a single or hundreds of nodes, do stream full-text filtering, add auto-complete, spell correction, more-like-this, faceting and other search-related technologies to their sites.

The default configuration includes a sample Real-Time index and listens on the default ports ( `9306` to connect with a MySQL client, `9308` - to connect via HTTP and `9312` - to connect via a binary protocol).

The image comes with libraries for easy indexing data from MySQL, PostgreSQL XML and CSV files.

# How to run this image

## Quick usage

Start a container with Manticore Search and log in to it via mysql client:
  
```
docker run --name manticore -d manticoresearch/manticore && docker exec -it manticore mysql -w
```

By default, the image is shipped with a sample index:
```

mysql> SHOW TABLES;
+--------+------+
| Index  | Type |
+--------+------+
| testrt | rt   |
+--------+------+
1 row in set (0,00 sec)


mysql> SELECT * FROM testrt;
+---------------------+------+-------------+-----------------------------------------------------------------------------------------------------------------------------+
| id                  | gid  | title       | content                                                                                                                     |
+---------------------+------+-------------+-----------------------------------------------------------------------------------------------------------------------------+
|                   1 |    1 | Hello World | Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. |
+---------------------+------+-------------+-----------------------------------------------------------------------------------------------------------------------------+
1 row in set (0,00 sec)
```


To shutdown the daemon:

```
docker stop manticore
```

# Composing

Create a stack.yml

```
version: '2.2'

services:

  manticore:
    image: manticoresearch/manticore:nodemode
    restart: always
    ulimits:
      nproc: 65535
      nofile:
         soft: 65535
         hard: 65535
      memlock:
        soft: -1
        hard: -1
```

Run `docker-compose -f stack.yml up` and connect with `docker exec -it docker_manticore_1 mysql`



## Mounting points

The image comes with a volume at `/var/lib/manticore/data`, which can be mounted to local folder for persistence.
To use a custom configuration file, mount `/etc/manticoresearch/manticore.conf`.

```
docker run --name manticore -v ~/manticore/etc/manticore.conf:/etc/manticoresearch/manticore.conf -v ~/manticore/data/:/var/lib/manticore/data -p 9306:9306 -d manticoresearch/manticore
```


## Logging

searchd runs with ``--no-detach`` option, sending it's log to `/dev/stdout`, which can be seen with 

```
  docker logs manticore
```

The query log can be diverted to Docker log by passing variable `QUERY_LOG_TO_STD=true`.

# Environment Variables

Several variables can be passed to adjust configuration of the Manticore instance:

### QUERY_LOG_TO_STD
 When set, it will divert the query logging to `/dev/stdout`
 
###  LEGACYMODE

This needs to be set when using old searchd way of having indexes defined in the configuration file.


## Multi-node cluster with replication

A simple `stack.yml` for defining a two node cluster:

```
version: '2.2'

services:

  manticore-1:
    image: manticoresearch/manticore:nodemode
    restart: always
    ulimits:
      nproc: 65535
      nofile:
         soft: 65535
         hard: 65535
      memlock:
        soft: -1
        hard: -1
    networks:
      - manticore
  manticore-2:
    image: manticoresearch/manticore:nodemode
    restart: always
    ulimits:
      nproc: 65535
      nofile:
        soft: 65535
        hard: 65535
      memlock:
        soft: -1
        hard: -1
    networks:
      - manticore
networks:
  manticore:
    driver: bridge
```
We start it with `docker-compose -f stack.yml up` .
Next, we need to create the cluster: we enter on the first instance and defined the cluster and attach the sample index to it:

```
$ docker exec -it docker_manticore-1_1 mysql

mysql> CREATE CLUSTER posts;
Query OK, 0 rows affected (0.24 sec)

mysql> ALTER CLUSTER posts ADD testrt;
Query OK, 0 rows affected (0.07 sec)

```

And on second instance we join it to the cluster:

```
$ docker exec -it docker_manticore-2_1 mysql

mysql> JOIN CLUSTER posts AT 'docker_manticore-1_1:9312';
mysql> INSERT INTO posts:testrt(title,content,gid)  VALUES('hello','world',1);
Query OK, 1 row affected (0.00 sec)
```

If we go back to the first instance we'll see the new record:
```
$ docker exec -it docker_manticore-1_1 mysql

mysql> select * from testrt;
+---------------------+------+-------------+-----------------------------------------------------------------------------------------------------------------------------+
| id                  | gid  | title       | content                                                                                                                     |
+---------------------+------+-------------+-----------------------------------------------------------------------------------------------------------------------------+
|                   1 |    1 | Hello World | Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. |
| 3891545739431510017 |    1 | hello       | world                                                                                                                       |
+---------------------+------+-------------+-----------------------------------------------------------------------------------------------------------------------------+
2 rows in set (0.00 sec)

```

# Memory locking and limits

It's recommended to overwrite the default ulimits of docker for the Manticore instance:

```
--ulimit nofile=65536:65536
```


For best performance, index components can be mlocked into memory. When Manticore is run under Docker, the instance requires additional privileges to allow memory locking. The following options must be added when running the instance:

```
  --cap-add=IPC_LOCK --ulimit memlock=-1:-1 
```


# Legacy mode

Legacy mode which support plain indexes and indexes defined in the configuration can be used with this instance.
In this case, the `manticore.conf` needs to be mounted and variable `LEGACYMODE=true` passed to the container.
```
docker run --name manticore -v ~/manticore/etc/manticore.conf:/etc/manticoresearch/manticore.conf -v ~/manticore/data/:/var/lib/manticore/data -e LEGACYMODE=true -p 9306:9306 -d manticoresearch/manticore
```
`searchd` daemon runs under `manticore`, performing operations on index files (like creating or rotation plain indexes) should be made under `manticore` user (otherwise files will be created under `root` and `searchd` can't manipulate them).
```
docker exec -it manticore gosu manticore indexer --all --rotate
```
 
# Issues

For reporting issues, please use the [issue tracker](https://github.com/manticoresoftware/docker/issues).

