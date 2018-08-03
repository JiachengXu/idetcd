# idetcd
[![Documentation](https://img.shields.io/badge/godoc-reference-blue.svg)](https://godoc.org/github.com/jiachengxu/idetcd/idetcd)
[![Build Status](https://img.shields.io/travis/jiachengxu/idetcd/master.svg?label=build)](https://travis-ci.org/jiachengxu/idetcd)
[![Code Coverage](https://img.shields.io/codecov/c/github/jiachengxu/idetcd/master.svg)](https://codecov.io/github/jiachengxu/idetcd?branch=master)
[![Go Report Card](https://goreportcard.com/badge/github.com/jiachengxu/idetcd)](https://goreportcard.com/report/jiachengxu/idetcd)

*idetcd* is a etcd-based [CoreDNS](https://coredns.io/) plugin used for identifying nodes in a cluster without domain name collsion.

## Motivation
In the distributed system, identifying nodes in the cluster is a big challenge since it's quite often that nodes can get down or start/restart in the cluster which contains thousands of nodes and would be quite annoying if rebooting is needed after the membership in the cluster. For tackling this problem, usually requiring some complicated protocols or additional DevOps, what's more, in most case it needs to customize the configuration for every different node which requires lots of work and also adds some risks to manage the system.

Distributed TensorFlow is also encountering the similar problems[[1]](https://github.com/jiachengxu/idetcd/blob/master/README.md#reference). In the older version of distributed TensorFlow, adding a node to the cluster is not that easy. First of all, you need to bring the whole system down, and then customize a configuration for the new node, after that restart the whole system. It did require additional DevOps work and it's also not "friendly" for some machine learning lovers to set up their own distributed TensorFlow clusters because, for people who are not familiar with distributed system, it would be great if she/he can add/delete a node from the system by one or two commands.

In practice, there are some approaches to solve this problem, for example, as mentioned before, building some protocols on the top of current Tensorflow codebase definitely can achieve this goal, but it probably is not a good way since may need to change the structure of Tensorflow, and bring some unnecessary complexities. And also, people who are not familiar with the infrastructure of TensorFlow still can not do anything if they still meet some problems when they deploy their own clusters. A more flexible way to do this is adding a separate module like DNS server, and nodes expose themselves through DNS.

CoreDNS has a plugin-based architecture and it is a really lightweight, flexible and extendable DNS server which can easily enable the customized plugin. For solving this issue, we can set up the CoreDNS plus customized plugin on every node in the TensorFlow cluster, and be using the plugin to write/read DNS records in a distributed key-value store, like zookeeper and etcd. And this is what *idetcd* does.

## How it works
![deploy](https://github.com/jiachengxu/idetcd/blob/master/fig/deploy.png)
The figure[[2]](https://github.com/jiachengxu/idetcd/blob/master/README.md#reference) above shows the scenario of how it works. The idea is quite simple: **Set up CoreDNS server on every node, and node exposes itself by taking the free domain name.**

In details, before the cluster is started, we set up CoreDNS server on every node in the cluster, for every node we just use the same configuration(See [example] below for details)which specifies the domain name pattern of nodes in this cluster, like worker{{.ID}}.tf.local., the maximum number of node allowed in this cluster, and etcd endpoints. Then we just start up all the nodes. That's it!

Notice, at the starting time, all the nodes haven't exposed to other nodes. Then we just start CoreDNS server on every node, and nodes will try to find free slots in the etcd to expose. For example, the node may first try to take worker1.tf.local., and then it will try to figure out if this domain name already exists in the etcd, if the answer is yes, then the node will try to increase the id to 2 and look into etcd again, otherwise, it will just take the name, and write it to the etcd. In this way, every node can dynamically find a domain name for itself without any collision. And also we don't need to customize the configuration for every node, instead, we use the same configuration and let the nodes to expose themselves!
## Usage

### Syntax

CoreDNS uses a configuration file called Corefile to specify the configuration, please go to [CoreDNS github repo](https://github.com/coredns/coredns) for more details. Here is a snippet for *idetcd* syntax:

~~~
idetcd {
	endpoint ENDPOINT...
	limit LIMIT
	pattern PATTERN
}
~~~

* `endpoint` **ENDPOINT** the etcd endpoints. Defaults to "http://localhost:2379".
* `limit` **LIMIT** the maximum limit of the node number in the cluster, if some nodes is going to expose itself after the node number in the cluster hits this limit, it will fail.
* `pattern` **PATTERN** the domain name pattern that every node follows in the cluster. And here we use golang template for the pattern.

### Example
In following example, we are going to start up a cluster which contains 5 nodes, on every node we can get this project by:

```
$ go get -u github.com/jiachengxu/idetcd
```

Before you move to the next step, make sure that you've **already set up a etcd instance**, and don't forget to write down the endpoints.

Then you need to add a Corefile which specifys the configuration of the CoreDNS server in the same directory of `main.go`, an simple Corefile example is as follows, please go to [CoreDNS github repo](https://github.com/coredns/coredns) for more details.

 ~~~ corefile
 . {
     idetcd {
         endpoint ETCDENDPOINTS
         limit 10
         pattern worker{{.ID}}.tf.local.
     }
 }
 ~~~

And then you can generate binary file by:
```sh
$ go build -v -o coredns
```

Alternatively, if you have docker installed, you could also execute the following to build:
```sh
$ docker run --rm -i -t -v $PWD:/go/src/github.com/jiachengxu/idetcd \
      -w /go/src/github.com/jiachengxu/idetcd golang:1.10 go build -v -o coredns
```

Then run it by:
```sh
$ ./coredns
```

After that, all nodes in the cluster are trying to find free slots in the etcd to expose themselves, once they are successed, you can get the domain name of every node on every node in the same cluster by:
```
$ dig +short worker4.tf.local @localhost
```
Also ipv6 is supported:
```
$ dig +short worker4.tf.local AAAA @localhost
```



## Reference
[[1]Dynamic RPC Address Resolution](https://groups.google.com/a/tensorflow.org/forum/#!msg/developers/s8MJ2vqQ1z0/mWoVaAMvCwAJ;context-place=forum/developers)  
[[2]Lightning Talk: Scaling Distributed Deep Learning with Service Discovery: How CoreDNS Helps Distributed TensorFlow Tasks - Yong Tang, Infoblox Inc.](https://www.youtube.com/watch?v=WBAP91g7Yd0)
