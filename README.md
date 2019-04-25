# Keycloak Cluster Setup And Configuration

This repo is just for knowledge sharing.

Through some efforts we achieved keycloak cluster in some scenes, maybe you guys have done this with some solutions but still hope this can help you or give you some idea.

We are using 4.8.3.Final docker image in our deployment, using this version because RH-SSO 7.3.0.GA is derived from 4.8.3.Final(please refer to https://www.keycloak.org/support.html) so we believe this is a stable and LTS build.

We add two cli script files based on the [keycloak image](https://hub.docker.com/r/jboss/keycloak/) as per the [guide](https://github.com/jboss-dockerfiles/keycloak/blob/master/server/README.md#adding-custom-discovery-protocols), these two files are the most important matter, here are the two files [TCPPING.cli](https://raw.githubusercontent.com/zhangliqiang/keycloak-cluster-setup-and-configuration/master/src/TCPPING.cli) and [JDBC_PING.cli](https://raw.githubusercontent.com/zhangliqiang/keycloak-cluster-setup-and-configuration/master/src/JDBC_PING.cli).

![0](https://raw.githubusercontent.com/zhangliqiang/keycloak-cluster-setup-and-configuration/master/src/0.jpg)

First of all we should know that for a keycloak cluster, all keycloak instances should use same database and this is very simple, another thing is about cache(generally there are two kinds of cache in keycloak, the 1st is persistent data cache read from database aim to improve performance like realm/client/user, the 2nd is the non-persistent data cache like sessions/clientSessions, the 2nd is very important for a cluster) which is a little bit complex to configure, we have to make sure the consistent of cache in a cluster view.

Totally we have 3 solutions for clustering, and all of the solutions are base on the discovery protocols of [JGroups](http://jgroups.org/), as keycloak use a distributed cache called [Infinispan](http://infinispan.org/) and the Infinispan use JGroups to discover nodes.


## 1. PING
[PING](http://jgroups.org/manual/#PING) is the default enabled clustering solution of keycloak using UDP protocol, and you don't need to do any configuration for this.

But this solution is only available when multicast network is enabled and port 55200 should be exposed, e.g. bare metals, VMs, docker containers in the same host.
![1](https://raw.githubusercontent.com/zhangliqiang/keycloak-cluster-setup-and-configuration/master/src/1.png)
We tested this by two keycloak containers in same host.
![2](https://raw.githubusercontent.com/zhangliqiang/keycloak-cluster-setup-and-configuration/master/src/2.png)

As you see from logs, the two keycloak instances discovered each other and clustered.

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
[JDBC_PING](http://jgroups.org/manual/#_jdbc_ping) use TCP protocol and 7600 port should be exposed which is similar as TCPPING, but the difference between them is, TCPPING require you configure the IP and port of all instances,  for JDBC_PING you just need to configure the IP and port of current instance, this is because in JDBC_PING solution each instance insert its own information into database and the instances discover peers by the ping data which is from database.


We tested this by two keycloak containers cross host.


![3](https://raw.githubusercontent.com/zhangliqiang/keycloak-cluster-setup-and-configuration/master/src/3.png)

And for our own solution we need to set two below environment variables for containers.

```
#IP address of this host
JGROUPS_DISCOVERY_EXTERNAL_IP=172.21.48.39
#protocol
JGROUPS_DISCOVERY_PROTOCOL=JDBC_PING
```

After started the ping data of all instances haven been saved in database, and from logs we can see the keycloak instances discovered each other and clustered.

![5](https://raw.githubusercontent.com/zhangliqiang/keycloak-cluster-setup-and-configuration/master/src/5.png)
![6](https://raw.githubusercontent.com/zhangliqiang/keycloak-cluster-setup-and-configuration/master/src/6.png)

## One more thing
I believe the above solutions are available for most scenes, but this is not enough for some other scene, e.g.kubernetes.

In kubernetes we can use [DNS_PING](https://github.com/jboss-dockerfiles/keycloak/blob/master/server/README.md#openshift-example-with-dnsdns_ping) and [KUBE_PING](http://jgroups.org/manual/#_kube_ping) which work quite well in [practice](https://github.com/helm/charts/blob/master/stable/keycloak/templates/statefulset.yaml#L92). 

Besides DNS_PING and KUBE_PING, we tried another solution for kubernetes. 

In kubernetes multicast is available only for the containers in same node and it's not working if cross node, furthermore a pod has no static ip which can be used to configure TCPPING or JDBC_PING.

But in the JDBC_PING.cli mentioned above we have handled this, if you don't set the JGROUPS_DISCOVERY_EXTERNAL_IP env, the pod ip will be used, that means in kubernetes you can simply set JGROUPS_DISCOVERY_PROTOCOL=JDBC_PING then your keycloak cluster is ok.
