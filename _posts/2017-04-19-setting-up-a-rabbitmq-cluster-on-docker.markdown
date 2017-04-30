---
layout: post
title:  Setting up a RabbitMQ Cluster on Docker
date:   2017-04-19 12:49:11 +0000
categories: docker rabbitmq cluster
image:  /preview.jpg
---
This tutorial will walk you through the steps of setting up a [RabbitMQ](https://www.rabbitmq.com/) cluster on [Docker](https://www.docker.com).

For this tutorial I'm assuming that:
* You are familiar with both Docker and RabbitMQ.
* The nodes will be running on UNIX systems.
* For the sake of learning our cluster will have just two nodes, so you need to servers.
* Your servers responds to a FQDN. (I will be using node-1.rabbit and node-2.rabbit)

### Preparing the environment

When you are working with Docker and you have data that needs to be persistent, it is a good practice to keep it out of your container.
RabbitMQ will save all the data to `/var/lib/rabbitmq` inside your container, so it's a good idea to map this folder to a local folder on your host. Among various reasons, 
I would say that one of the most important is: If your container goes into an inconsistent state you can just re-create the container and map the same data folder to it. Or
if you need to backup the data or move somewhere else, just zip that folder.


At this point all you have to do is create a folder that will soon be mapped to the RabbitMQ containers. For production deployments I usually create a `data` folder on the system root directory tree.
You can pick whatever you want, but if you are on a macOS you will have some [Restrictions](https://docs.docker.com/engine/tutorials/dockervolumes/#mount-a-host-directory-as-a-data-volume).


So go ahead and create your data folder on both servers (like: `mkdir /data`)


### Starting RabbitMQ docker containers

Now it's time for some Docker! First let's start our main RabbitMQ node:
```java
docker run -d -h node-1.rabbit                                      \
           --name rabbit                                            \
           -p "4369:4369"                                           \
           -p "5672:5672"                                           \
           -p "15672:15672"                                         \
           -p "25672:25672"                                         \
           -p "35197:35197"                                         \
           -e "RABBITMQ_USE_LONGNAME=true"                          \
           -e "RABBITMQ_LOGS=/var/log/rabbitmq/rabbit.log"          \
           -v /data:/var/lib/rabbitmq \
           -v /data/logs:/var/log/rabbitmq \
           rabbitmq:3.6.6-management
```

If you are familiar with Docker you probably know what is going on here.
- At the first line, we define the container hostname, which will be used to identify the node.
- At the second line, we define the container name.
- From lines 3 to 7 we map the container ports to the host. That way we can access the container services through `node-1.rabbit:PORT`.
For details about RabbitMQ ports please [See this](https://www.rabbitmq.com/clustering.html#firewall).
- After that we define some environment variables that will be used for RabbitMQ configuration. In this case we are setting just two, you can check what else is available and useful for you [Here](https://www.rabbitmq.com/configure.html#define-environment-variables).
We are setting `RABBITMQ_USE_LONGNAME=true` so each node will communicate with each other using their FQDN. We also set `RABBITMQ_LOGS` so all the output goes to a log file
instead of STDOUT (which may be a problem in production).
- At the 10th and 11th lines, we map the host directory we created for node-1 to the rabbit container.
- Finally we specify which docker image we will be using. In this case RabbitMQ with the management plugin.

That's it! We have RabbitMQ running, easy eh? If you ever had to install RabbitMQ before you probably know that it can be really trick to deal with
a lot of dependencies, Erlang/OTP and so on. You can have an idea by checking [RabbitMQ Dockerfile](https://github.com/docker-library/rabbitmq/blob/1509b142f0b858bb9d8521397f34229cd3027c1e/3.6/debian/Dockerfile).

Want to see it live? Just give it a try. Open your browser and go to `http://node-1.rabbit:15672`. You should see something like this:

[![](http://i.imgur.com/oKsuaTz.png){:width="100%"}](http://i.imgur.com/oKsuaTz.png)

If check the data folder we created before you will see that RabbitMQ has created a log folder for you, and you also should have some data on it.


Now do the same thing on the second server. Don't forget to change the hostname on the docker run command.
```java
docker run -d -h node-2.rabbit                                      \
           --name rabbit                                            \
           -p "4369:4369"                                           \
           -p "5672:5672"                                           \
           -p "15672:15672"                                         \
           -p "25672:25672"                                         \
           -p "35197:35197"                                         \
           -e "RABBITMQ_USE_LONGNAME=true"                          \
           -e "RABBITMQ_LOGS=/var/log/rabbitmq/rabbit.log"          \
           -v /data:/var/lib/rabbitmq \
           -v /data/logs:/var/log/rabbitmq \
           rabbitmq:3.6.6-management
```
<br />
### Configuring our RabbitMQ cluster

#### Erlang Cookie
Each RabbitMQ node should have file named `.erlang.cookie` located on your data folder, in our case `/data`. This file contains a secret (some sort of random hash),
that will be automatically created if it doesn't exist.
This file is used to determine if a node can communicate with another node. The communication will be allowed if both nodes share the same
secret on `.erlang.cookie` file.

Just copy the secret on `.erlang.cookie` from the first node and copy it to the second node. Then restart the Docker container for the second node.


#### RabbitMQ Browser UI

_rabbitmq-management_ plugin provides a browser-based UI to monitor and configure a lot of RabbitMQ features.
You can access it at: `http://node-1.rabbit:15672`

Unfortunately you can't configure 100% of your cluster through the browser interface, for that reason we are going to use only the command line tool `rabbitmqctl`.


#### Disabling guest user

_This step is not mandatory, but you definitely want to do it in production for security reasons._ RabbitMQ will by default create a _guest_ user (password: guest).

From now on we are going to send `rabbitmqctl` commands through the command line to the docker RabbitMQ containers.

Let's get rid of this security branch:
`docker exec rabbit rabbitmqctl delete_user guest`

You just need to do this on the first node. Once we create the cluster and connect the second RabbitMQ node to it all the configurations will be copied.


#### RabbitMQ virtual host

For learning purposes, let's assume that we have just one virtual host `/`. All configuration will belong to that host. Once again, you just need to
do this on the first node.

Create a virtual host:
`docker exec rabbit rabbitmqctl add_vhost /`


#### Create Users
We are going to create two users. One is an administrator user, which can have access to the browser UI and make changes to the cluster configurations. 
The second is an application level user, which will have permissions to interact with RabbitMQ at the application level. It will have read/write permissions to the `/` virtual host.
You can customize it to your needs.
```ruby
# Admin user
docker exec rabbit rabbitmqctl add_user josuelima MyPassword999
docker exec rabbit rabbitmqctl set_user_tags josuelima administrator

# Application user
docker exec rabbit rabbitmqctl add_user ruby_app SuperPassword000
docker exec rabbit rabbitmqctl set_permissions -p / ruby_app ".*" ".*" ".*"
```

Once again, you just need to do this on the first node.

#### Creating our cluster

Now that our first node is setup, we are going to make our second node join the first and start clustering.
All we have to this is stop the second node RabbitMQ (not the docker container) and then connect it to the first node.

Let's move to the second node and do it:
```
docker exec rabbit rabbitmqctl stop_app
docker exec rabbit rabbitmqctl join_cluster rabbit@node-1.rabbit
docker exec rabbit rabbitmqctl start_app
```

Great! Now the second node will copy all the settings from the first node and we should have our cluster running.
You can check it in the browser UI or using the command line.

If you access the browser UI at `http://node-1.rabbit:15672` or `http://node-2.rabbit:15672` and login with the administrator user we just created before, 
you should see something like this:

[![](http://i.imgur.com/mkPkwGS.png){:width="100%"}](http://i.imgur.com/mkPkwGS.png)

On the command line you can do: `docker exec rabbit rabbitmqctl cluster_status`

That's it! Now you have a RabbitMQ Cluster running with Docker. If you want to add more nodes to it just create a new RabbitMQ Docker container and connect it to the cluster.

### Queues behaviour

Now that we have our cluster running it's time to think about how we want it to behave.
_By default, queues within a RabbitMQ cluster are located on a single node (the node on which they were first declared)._ **It is really important to understand this.**

What it means is that your queues will be distributed among your cluster nodes. Is this the kind of scneario you want? When is it a good idea?

#### Distributed Queues

A good use for distributed Queues is when you have really high I/O rates for your queues and want to distribute the load across all the nodes.
Have in mind that a queue will live in one only node, and if one node goes down all the queues living on that node will be unavailable until it's back up.

If this is the scenario you want, you are good to go. Distributed Queues are default.

I would also recommend you having a look at [HAProxy Load Balancer](http://www.haproxy.org), or if you are on AWS [AWS Elastic Load Balancing](https://aws.amazon.com/elasticloadbalancing/). It becomes really useful on a distributed queues scenario, where you have a lot of nodes.

#### Mirrored Queues

You can also have Mirrored Queues, which is a kind of replica-set. In this situation all the nodes will just be a copy of each other. All the messages published
to a queue are replicated to all the cluster nodes.

This is very useful when you need high availability for your queues. That way, if a node goes down you still have all your queues available in the other nodes.

To setup mirrored queues you just need to set a policy to your queues to make them _high available_.

`docker exec rabbit rabbitmqctl set_policy ha "." '{"ha-mode":"all"}'`

In this example we set the policy named `ha` so all queues `.` will be high available in all nodes `{"ha-mode":"all"}`.

You can have various configurations for High Available queues, check the [Documentation](https://www.rabbitmq.com/ha.html).


### Conclusion

Here you got the concepts of setting up a RabbitMQ Cluster on Docker. There's way more that you can do to improve your cluster implementation, like RAM Nodes, Firewall Nodes, Partition, Synchronization and so on. I've listed below two important links:

* [RabbitMQ Clustering docs: https://www.rabbitmq.com/clustering.html](https://www.rabbitmq.com/clustering.html)
* [Mirrored Queues and High Availability: https://www.rabbitmq.com/ha.html](https://www.rabbitmq.com/ha.html)




