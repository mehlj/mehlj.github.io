---
layout: post
title: Docker Swarm
published: true
category: infrastructure
---

Docker containers are simple to manage when you have one container or
even [multiple containers on one host](https://mehlj.github.io/DockerCompose/). 

But what if you have tens of containers that you have to manage? What if you had to host these containers on several
 Docker hosts?

That is where [container orchestration](https://www.redhat.com/en/topics/containers/what-is-container-orchestration) 
tools come in. Docker Swarm is the orchestrator native to the Docker toolset. 

## Use Case
You should consider using an orchestrator anytime:
* You run many containers at a time
* You intend to run many containers across several different Docker hosts
* You are concerned with the availability of the services provided by these containers

Docker Swarm and Kubernetes are both orchestrators and provide generally the same function, but Kubernetes is generally 
considered to be more robust, more feature rich, and more efficient at larger scale. The tradeoff is that Kubernetes is 
also considered to be more complex and difficult to manage. 

Docker Swarm is a good idea if you have a relatively small cluster of containers and hosts, and do not need the rich
features that Kubernetes provides.

## Example
Docker services can be deployed on swarms using standard `docker-compose.yml` format. This is a simple example of an 
out-of-the-box `nginx` web server replicated four times across all our swarm hosts.
```yaml
version: '3'

services:
  web:
    image: nginx
    ports:
      - "8080:80"
    deploy:
      mode: replicated
      replicas: 4
```

## Swarm Initialization
Before we deploy our example service, we need to create our swarm cluster. In this example, I will use two similar
CentOS 8 servers. 

On one of the servers, we need to initialize the swarm:
```
[mehlj@docker ~]$ sudo docker swarm init
Swarm initialized: current node (y56cl9ypgxbxf9inqlyb1tsh3) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-07x0dsyasrut2mxgxnyoyzk0qly552mv2cmge9i80k5llijeo5-cdgdiwd08gwqd7ffvfzca7b65 192.168.1.71:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

As it stands, we have a single node cluster, with this server acting as a `manager`. Managers handle the scheduling
and management of other `workers` in the cluster. Workers actually handle running the containers. By default, managers
can act as workers simultaneously. 
```
[mehlj@docker ~]$ sudo docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
y56cl9ypgxbxf9inqlyb1tsh3 *   docker              Ready               Active              Leader              18.09.1
[mehlj@docker ~]$ 
```

We will join our other server as a `worker` to this swarm. The output of the `init` provided us with a command line
to run for this purpose, so let's run it on the other server:
```
[mehlj@dockerworker ~]$ sudo docker swarm join --token SWMTKN-1-07x0dsyasrut2mxgxnyoyzk0qly552mv2cmge9i80k5llijeo5-cdgdiwd08gwqd7ffvfzca7b65 192.168.1.71:2377
This node joined a swarm as a worker.
```

Now, we can confirm that we have two nodes in our cluster:
```
[mehlj@docker ~]$ sudo docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
y56cl9ypgxbxf9inqlyb1tsh3 *   docker              Ready               Active              Leader              18.09.1
nsq40rxf0jvrwrmj70k3ivh5z     dockerworker        Ready               Active                                  18.09.1
```

## Deployment
Deploying our service is simple using `docker stack deploy`:
```
[mehlj@docker ~]$ sudo docker stack deploy -c docker-stack.yml web
Creating network web_default
Creating service web_web
```

We can confirm the service was deployed using `docker service ls`:
```
[mehlj@docker ~]$ sudo docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
2k1kkf96purp        web_web             replicated          4/4                 nginx:latest        *:8080->80/tcp
```

According to that, all four replicas are running. We can confirm this by diving a little deeper with `docker service ps`:
```
[mehlj@docker ~]$ sudo docker service ps web_web
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
s8b5y1plt2kd        web_web.1           nginx:latest        docker              Running             Running 6 seconds ago                       
xec0krlvgdb0        web_web.2           nginx:latest        dockerworker        Running             Running 5 seconds ago                       
jncx6toiw4wv        web_web.3           nginx:latest        docker              Running             Running 6 seconds ago                       
q834tm690c0f        web_web.4           nginx:latest        dockerworker        Running             Running 5 seconds ago                       
```

This tells us that Docker has split up our replicas evenly across our two nodes - `docker` and `dockerworker`. 

Now, we test our service by sending a `GET` request to 192.168.1.71:8080 or 192.168.1.70:8080:
```
[mehlj@workstation ~]$ curl http://192.168.1.70:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

## Scaling
One major benefit of using an orchestrator is scaling. If our workload is abnormally high, or we simply want more
redundancy for our service, we can nearly-instantly scale our application using a simple command line. Similarly,
we can scale our application down if the replicas are unnecessary/wasting resources. All with no downtime to the app.

### Scaling up
We started with four replicas, so we can scale that up to five:
```
[mehlj@docker ~]$ sudo docker service scale web_web=5
web_web scaled to 5
overall progress: 5 out of 5 tasks 
1/5: running   [==================================================>] 
2/5: running   [==================================================>] 
3/5: running   [==================================================>] 
4/5: running   [==================================================>] 
5/5: running   [==================================================>] 
verify: Service converged 
[mehlj@docker ~]$ 
```
```
[mehlj@docker ~]$ sudo docker service ps web_web 
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
s8b5y1plt2kd        web_web.1           nginx:latest        docker              Running             Running 22 minutes ago                       
xec0krlvgdb0        web_web.2           nginx:latest        dockerworker        Running             Running 22 minutes ago                       
jncx6toiw4wv        web_web.3           nginx:latest        docker              Running             Running 22 minutes ago                       
q834tm690c0f        web_web.4           nginx:latest        dockerworker        Running             Running 22 minutes ago                       
yd5ybceu83do        web_web.5           nginx:latest        dockerworker        Running             Running 48 seconds ago                       
```

### Scaling down
Similarly, we can scale it down back to four:
```
[mehlj@docker ~]$ sudo docker service scale web_web=4
web_web scaled to 4
overall progress: 4 out of 4 tasks 
1/4: running   [==================================================>] 
2/4: running   [==================================================>] 
3/4: running   [==================================================>] 
4/4: running   [==================================================>] 
verify: Service converged 
```
```
[mehlj@docker ~]$ sudo docker service ps web_web 
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
s8b5y1plt2kd        web_web.1           nginx:latest        docker              Running             Running 23 minutes ago                       
xec0krlvgdb0        web_web.2           nginx:latest        dockerworker        Running             Running 23 minutes ago                       
jncx6toiw4wv        web_web.3           nginx:latest        docker              Running             Running 23 minutes ago                       
q834tm690c0f        web_web.4           nginx:latest        dockerworker        Running             Running 23 minutes ago                       
```

## Redundancy
Just to validate the added redundancy we gain by using swarm, we can simulate our Docker worker going down by simply
stopping the Docker service. After that, we can see if our application stays online. The swarm manager should offload
the replicas to itself seamlessly.

First, we can start by killing the containers on the worker:
```
[mehlj@dockerworker ~]$ sudo service docker stop
Redirecting to /bin/systemctl stop docker.service
```
Now, on the manager, we can see that the worker shows `offline`:
```
[mehlj@docker ~]$ sudo docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
y56cl9ypgxbxf9inqlyb1tsh3 *   docker              Ready               Active              Leader              18.09.1
nsq40rxf0jvrwrmj70k3ivh5z     dockerworker        Down                Active                                  18.09.1
```

We can also see that the `dockerworker` containers have `shutdown`, but are reinstated on the manager:
```
[mehlj@docker ~]$ sudo docker service ps web_web 
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
s8b5y1plt2kd        web_web.1           nginx:latest        docker              Running             Running 35 minutes ago                       
relqxoue01a0        web_web.2           nginx:latest        docker              Running             Running 12 seconds ago                       
xec0krlvgdb0         \_ web_web.2       nginx:latest        dockerworker        Shutdown            Running 35 minutes ago                       
jncx6toiw4wv        web_web.3           nginx:latest        docker              Running             Running 35 minutes ago                       
7tvpw162vj4s        web_web.4           nginx:latest        docker              Running             Running 12 seconds ago                       
q834tm690c0f         \_ web_web.4       nginx:latest        dockerworker        Shutdown            Running 35 minutes ago                       
```

We can also see that our application is still working correctly, despite the turmoil:
```
[mehlj@docker ~]$ curl http://192.168.1.71:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```