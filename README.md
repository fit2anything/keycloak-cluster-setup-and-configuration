# Keycloak Cluster Setup And Configuration

Through some efforts we achieved keycloak cluster setting in some scenes, maybe you guys have done this by some solutions but hope this can help you anyway.

We are using 4.8.3.Final docker image in our deployment, using this version because RH-SSO 7.3.0.GA is derived from 4.8.3.Final(please refer to https://www.keycloak.org/support.html) and we believe this is a rather stable build.

Also we did some extensions base on the official docker image, including custom theme, custom user federation, but the two cli script files is the most important matter for this mail, here are the two files [TCPPING.cli](https://raw.githubusercontent.com/zhangliqiang/keycloak-cluster-setup-and-configuration/master/src/TCPPING.cli) and [JDBC_PING.cli](https://raw.githubusercontent.com/zhangliqiang/keycloak-cluster-setup-and-configuration/master/src/JDBC_PING.cli).
![0](https://raw.githubusercontent.com/zhangliqiang/keycloak-cluster-setup-and-configuration/master/src/0.jpg)

First of all we have to know that for a keycloak cluster, all keycloak instances should use same database and this is very simple, another thing is about cache(generally there are two kinds of cache in keycloak, 1st is persistent data cache read from database aim to improve performance like realm/client/user, 2nd is the non-persistent data cache like sessions/clientSessions, the 2nd is very important for a cluster) which is a little bit complex to configure, we have to make sure the consistent of cache data in a cluster view.

Totally we have 3 solutions for clustering, and all of the solutions are base on the discovery protocols of [JGroups](http://jgroups.org/), as keycloak use a distributed cache called [Infinispan](http://infinispan.org/) and the Infinispan use JGroups to discover nodes.


## 1. PING
[PING](http://jgroups.org/manual/#PING) is the default enabled clustering solution of keycloak using UDP protocol, and you don't need to do any configuration for this.

But this solution is only available when multicast network is enabled and port 55200 should be exposed, e.g. bare metals, VMs, docker containers in same host.
![1](https://raw.githubusercontent.com/zhangliqiang/keycloak-cluster-setup-and-configuration/master/src/1.png)
We tested this by two keycloak containers in same host.
![2](https://raw.githubusercontent.com/zhangliqiang/keycloak-cluster-setup-and-configuration/master/src/2.png)

As you see from logs, after started the two keycloak instances discovered each other and clustered.

## 2. TCPPING
[TCPPING](http://jgroups.org/manual/#TCPPING_Prot) use TCP protocol and 7600 port should be exposed.

This solution can be used when multicast is not available, e.g. deployments cross DC, containers cross host.


We tested this by two keycloak containers cross host.


![3](https://raw.githubusercontent.com/zhangliqiang/keycloak-cluster-setup-and-configuration/master/src/3.png)


And for our own solution we need to set three below environment variables for containers.

```
#IP address of this host
JGROUPS_DISCOVERY_EXTERNAL_IP=172.21.48.39
#protocol
JGROUPS_DISCOVERY_PROTOCOL=TCPPING
#IP and Port of all host
JGROUPS_DISCOVERY_PROPERTIES=initial_hosts="172.21.48.4[7600],172.21.48.39[7600]"
```

After started we can see the keycloak instances discovered each other and clustered.
![4](https://raw.githubusercontent.com/zhangliqiang/keycloak-cluster-setup-and-configuration/master/src/4.png)

## 3. JDBC_PING
[JDBC_PING](http://jgroups.org/manual/#_jdbc_ping) use TCP protocol and 7600 port should be expose which is similar as TCPPING, but the difference between them is, TCPPING require you configure the IP and port of all instances,  for JDBC_PING you just need to configure the IP and port of current instance, this is because in JDBC_PING solution each instance insert its own information into database and the instances discover peers by the ping data which is from database.


We tested this by two keycloak containers cross host.


![3](https://raw.githubusercontent.com/zhangliqiang/keycloak-cluster-setup-and-configuration/master/src/3.png)
```
And for our own solution we need to set two below environment variables for containers.
#IP address of this host
JGROUPS_DISCOVERY_EXTERNAL_IP=172.21.48.39
#protocol
JGROUPS_DISCOVERY_PROTOCOL=JDBC_PING
```

After started the ping data of all instances haven been saved in database, and from logs we can see the keycloak instances discovered each other and clustered.

![5](https://raw.githubusercontent.com/zhangliqiang/keycloak-cluster-setup-and-configuration/master/src/5.png)
![6](https://raw.githubusercontent.com/zhangliqiang/keycloak-cluster-setup-and-configuration/master/src/6.png)

---
I believe the above solutions are available for most scenes, but for some other scene this is not enough, e.g. kubernetes.

In kubernetes, multicast is available only for the containers in same node, for the pods cross node multicast is not working, furthermore for a pod there is no static ip that can be used to configure TCPPING or JDBC_PING.

But that's ok because we can use [KUBE_PING](http://jgroups.org/manual/#_kube_ping) in kubernetes. And also don't worry, KUBE_PING is not the only choice, actually JDBC_PING is another option. In the attached JDBC_PING.cli we have handled this,  if you don't set the JGROUPS_DISCOVERY_EXTERNAL_IP environment variable, the pod ip will be used, that means in kubernetes you can just set JGROUPS_DISCOVERY_PROTOCOL=JDBC_PING then your keycloak cluster is ok.
