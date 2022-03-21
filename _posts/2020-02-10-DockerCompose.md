---
layout: post
title: Docker Compose
published: true
category: infrastructure
---

Docker is software designed to package an application into a lightweight, portable, and fast self-contained environment. 
This environment is known as a _container_. 

Docker Compose is a native Docker tool that allows you to manage applications comprised of several Docker containers.
It allows you to create a full application, spread amongst several containers, through one configuration file (YAML) and
one command. 

Each Docker container has a unique `Dockerfile`, and a single Docker Compose file `docker-compose.yml` defines what
images to use and their dependency upon each other. The application's containers can then be managed using the 
`docker-compose` command line interface.

## Use Case
A best practice is to have a _single process_ per Docker container - a process that will kill the container when it dies, and 
vice versa. This is to promote [speed, security, and portability of containers](
https://devops.stackexchange.com/questions/447/why-it-is-recommended-to-run-only-one-process-in-a-container).

The issue with this is modern software often has multiple components. For example, most web servers need databases to
store non-volatile data. It would be inefficient to run both, say, Apache and MySQL in the same container, so it should
be split into two containers. But, they still need to communicate with eachother and be managed at the same time. That 
is where `docker-compose` comes in. 

## Example
[Solution Stack](https://en.wikipedia.org/wiki/Solution_stack) software is commonly used today, and one example is the
[ELK stack](https://www.elastic.co/what-is/elk-stack). 

One advantage of Docker and Docker Compose is the ability to spin up instances of these stacks in seconds, as opposed to
hours or days of manual configuration, troubleshooting, and fiddling. 

A FOSS project known as [`docker-elk`](https://github.com/deviantony/docker-elk) does the heavy lifting of writing each 
`Dockerfile` and packaging it all up. It then leverages Docker Compose to create all ELK stack components at once. 

This is the `docker-compose.yml`:
```yaml
version: '3.2'

services:
  elasticsearch:
    build:
      context: elasticsearch/
      args:
        ELK_VERSION: $ELK_VERSION
    volumes:
      - type: bind
        source: ./elasticsearch/config/elasticsearch.yml
        target: /usr/share/elasticsearch/config/elasticsearch.yml
        read_only: true
      - type: volume
        source: elasticsearch
        target: /usr/share/elasticsearch/data
    ports:
      - "9200:9200"
      - "9300:9300"
    environment:
      ES_JAVA_OPTS: "-Xmx256m -Xms256m"
      ELASTIC_PASSWORD: changeme
      # Use single node discovery in order to disable production mode and avoid bootstrap checks
      # see https://www.elastic.co/guide/en/elasticsearch/reference/current/bootstrap-checks.html
      discovery.type: single-node
    networks:
      - elk

  logstash:
    build:
      context: logstash/
      args:
        ELK_VERSION: $ELK_VERSION
    volumes:
      - type: bind
        source: ./logstash/config/logstash.yml
        target: /usr/share/logstash/config/logstash.yml
        read_only: true
      - type: bind
        source: ./logstash/pipeline
        target: /usr/share/logstash/pipeline
        read_only: true
    ports:
      - "5000:5000/tcp"
      - "5000:5000/udp"
      - "9600:9600"
    environment:
      LS_JAVA_OPTS: "-Xmx256m -Xms256m"
    networks:
      - elk
    depends_on:
      - elasticsearch

  kibana:
    build:
      context: kibana/
      args:
        ELK_VERSION: $ELK_VERSION
    volumes:
      - type: bind
        source: ./kibana/config/kibana.yml
        target: /usr/share/kibana/config/kibana.yml
        read_only: true
    ports:
      - "5601:5601"
    networks:
      - elk
    depends_on:
      - elasticsearch

networks:
  elk:
    driver: bridge

volumes:
  elasticsearch:
```

## Running
We can begin by cloning the Github project:
```
[mehlj@docker compose]$ git clone https://github.com/deviantony/docker-elk.git
Cloning into 'docker-elk'...
remote: Enumerating objects: 10, done.
remote: Counting objects: 100% (10/10), done.
remote: Compressing objects: 100% (9/9), done.
remote: Total 1530 (delta 2), reused 4 (delta 1), pack-reused 1520
Receiving objects: 100% (1530/1530), 374.45 KiB | 5.85 MiB/s, done.
Resolving deltas: 100% (599/599), done.
```
According to the `README.md`, we must modify the SELinux context of the new directory:
```
[mehlj@docker compose]$ sudo chcon -R system_u:object_r:admin_home_t:s0 docker-elk/
```
If we do not have `docker-compose` installed yet, we can do this by first downloading the latest version:
```
[mehlj@docker docker-elk]$ sudo curl -L "https://github.com/docker/compose/releases/download/1.25.3/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   617  100   617    0     0   3485      0 --:--:-- --:--:-- --:--:--  3485
100 16.4M  100 16.4M    0     0  11.9M      0  0:00:01  0:00:01 --:--:-- 20.4M
```
Change the permissions:
```
[mehlj@docker docker-elk]$ sudo chmod +x /usr/local/bin/docker-compose
```
Add it to our `$PATH`:
```
[mehlj@docker docker-elk]$ sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```
Confirm it works:
```
[mehlj@docker docker-elk]$ docker-compose --version
docker-compose version 1.25.3, build d4d1b42b
```
Now, we can start our ELK stack using the one-liner `docker-compose up` (`-d` for background):
```
[mehlj@docker docker-elk]$ docker-compose up -d
```
Visit `http://localhost:9200` for raw ElasticSearch data:
```json
{
  "name" : "d00c8cf2b493",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "W3SqhjoOS3KygA2CTLlT3A",
  "version" : {
    "number" : "7.5.1",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "3ae9ac9a93c95bd0cdc054951cf95d88e1e18d96",
    "build_date" : "2019-12-16T22:57:37.835892Z",
    "build_snapshot" : false,
    "lucene_version" : "8.3.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```
Visit `http://localhost:5601` for Kibana access, and so forth.

After usage, we can quickly destroy our stack using `docker-compose` (`-v` flag deletes the persistent volume data):
```
[mehlj@docker docker-elk]$ docker-compose down -v
```
