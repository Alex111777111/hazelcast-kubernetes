hazelcast-kubernetes
====================

Hazelcast clustering for Kubernetes made easy. It includes a lean (98MB) Hazelcast with Kubernetes discovery support container image, based on Alpine Linux.

[![Docker Repository on Quay](https://quay.io/repository/pires/hazelcast-kubernetes/status "Docker Repository on Quay")](https://quay.io/repository/pires/hazelcast-kubernetes)

## Software

* JRE 8u121
* Hazelcast 3.8.1

## Pre-requisites

* Docker
* Kubernetes cluster, version 1.4 or newer

## Kubernetes cluster

You can test with a local cluster. Check [this other repository from yours truly](https://github.com/pires/kubernetes-vagrant-coreos-cluster).

## Docker image

The image is already available at [quay.io/pires](https://quay.io/repository/pires/hazelcast-kubernetes)

## Cloud Native Deployments of Hazelcast using Kubernetes

The following document describes the development of a _cloud native_ [Hazelcast](http://hazelcast.org/) deployment on Kubernetes.  When we say _cloud native_ we mean an application which understands that it is running within a cluster manager, and uses this cluster management infrastructure to help implement the application. In particular, in this instance, a custom Hazelcast ```bootstrapper``` is used to enable Hazelcast to dynamically discover Hazelcast nodes that have already joined the cluster.

Any topology changes are communicated and handled by Hazelcast nodes themselves.

This document also attempts to describe the core components of Kubernetes, _Pods_, _Services_ and _Replication Controllers_.

### Prerequisites
This example assumes that you have a Kubernetes cluster installed and running, and that you have installed the `kubectl` command line tool somewhere in your path.  Please see the [getting started](https://github.com/GoogleCloudPlatform/kubernetes/tree/master/docs/getting-started-guides) for installation instructions for your platform.

### A note for the impatient
This is a somewhat long tutorial.  If you want to jump straight to the "do it now" commands, please see the [tl; dr](#tl-dr) at the end.

### Sources

Source is freely available at:
* Hazelcast Discovery - https://github.com/pires/hazelcast-kubernetes-bootstrapper
* Dockerfile - https://github.com/pires/hazelcast-kubernetes
* Docker Trusted Build - https://registry.hub.docker.com/u/pires/hazelcast-k8s

### Simple Single Pod Hazelcast Node
In Kubernetes, the atomic unit of an application is a [_Pod_](http://docs.k8s.io/pods.md). A Pod is one or more containers that _must_ be scheduled onto the same host. All containers in a pod share a network namespace, and may optionally share mounted volumes.

In this case, we shall not run a single Hazelcast pod, because the discovery mechanism now relies on a service definition.


### Adding a Hazelcast Service
In Kubernetes a _Service_ describes a set of Pods that perform the same task. For example, the set of nodes in a Hazelcast cluster. An important use for a Service is to create a load balancer which distributes traffic across members of the set. But a _Service_ can also be used as a standing query which makes a dynamically changing set of Pods available via the Kubernetes API. This is actually how our discovery mechanism works, by relying on the service to discover other Hazelcast pods.

Here is the service description:
```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    name: hazelcast
  name: hazelcast
spec:
  ports:
    - port: 5701
  selector:
    name: hazelcast
```

The important thing to note here is the `selector`. It is a query over labels, that identifies the set of _Pods_ contained by the _Service_. In this case the selector is `name: hazelcast`. If you look at the Replication Controller specification below, you'll see that the pod has the corresponding label, so it will be selected for membership in this Service.

Create this service as follows:
```sh
$ kubectl create -f service.yaml
```

### Adding replicated nodes
The real power of Kubernetes and Hazelcast lies in easily building a replicated, resizable Hazelcast cluster.

In Kubernetes a _Deployment_ is responsible for replicating sets of identical pods. Like a _Service_ it has a selector query which identifies the members of its set.  Unlike a _Service_ it also has a desired number of replicas, and it will create or delete _Pods_ to ensure that the number of _Pods_ matches up with its desired state.

Deployments will "adopt" existing pods that match their selector query, so let's create a Deployment with a single replica to adopt our existing Hazelcast Pod.

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: hazelcast
  labels:
    name: hazelcast
spec:
  template:
    metadata:
      labels:
        name: hazelcast
    spec:
      containers:
      - name: hazelcast
        image: quay.io/pires/hazelcast-kubernetes:3.8
        imagePullPolicy: Always
        env:
        - name: "DNS_DOMAIN"
          value: "cluster.local"
        ports:
        - name: hazelcast
          containerPort: 5701
```

You may note that we tell Kubernetes that the container exposes the `hazelcast` port. Finally, we tell the cluster manager that we need 1 cpu core.

The bulk of the replication controller config is actually identical to the Hazelcast pod declaration above, it simply gives the controller a recipe to use when creating new pods.  The other parts are the `selector` which contains the controller's selector query, and the `replicas` parameter which specifies the desired number of replicas, in this case 1.

Last but not least, we set `DNS_DOMAIN` environment variable according to your Kubernetes clusters DNS configuration.

Create this controller:

```sh
$ kubectl create -f deployment.yaml
```

After the controller provisions successfully the pod, you can query the service endpoints:
```sh
$ kubectl get endpoints hazelcast -o yaml
apiVersion: v1
kind: Endpoints
metadata:
  creationTimestamp: 2017-03-14T20:18:40Z
  labels:
    name: hazelcast
  name: hazelcast
  namespace: default
  resourceVersion: "4971"
  selfLink: /api/v1/namespaces/default/endpoints/hazelcast
  uid: 69eae4eb-08f3-11e7-9da7-080027e7656f
subsets:
- addresses:
  - ip: 10.244.60.2
    nodeName: 172.17.8.103
    targetRef:
      kind: Pod
      name: hazelcast-4195412960-1v7vx
      namespace: default
      resourceVersion: "4969"
      uid: 6e1501a2-08f3-11e7-9da7-080027e7656f
  ports:
  - port: 5701
    protocol: TCP
```

You can see that the _Service_ has found the pod created by the replication controller.

Now it gets even more interesting. Let's scale our cluster to 2 pods:
```sh
$ kubectl scale deployment hazelcast --replicas 2
```

Now if you list the pods in your cluster, you should see two Hazelcast pods:

```sh
$ kubectl get deployment,pods
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deploy/hazelcast   2         2         2            2           4m

NAME                            READY     STATUS    RESTARTS   AGE
po/hazelcast-4195412960-1v7vx   1/1       Running   0          4m
po/hazelcast-4195412960-hn7vd   1/1       Running   0          30s
```

To prove that this all works, you can use the `log` command to examine the logs of one pod, for example:

```sh
kubectl logs po/hazelcast-4195412960-1v7vx
2017-03-14 20:19:24.219  INFO 7 --- [           main] com.github.pires.hazelcast.Application   : Starting Application on hazelcast-4195412960-1v7vx with PID 7 (/bootstrapper.jar started by root in /)
2017-03-14 20:19:24.241  INFO 7 --- [           main] com.github.pires.hazelcast.Application   : No active profile set, falling back to default profiles: default
2017-03-14 20:19:24.408  INFO 7 --- [           main] s.c.a.AnnotationConfigApplicationContext : Refreshing org.springframework.context.annotation.AnnotationConfigApplicationContext@14514713: startup date [Tue Mar 14 20:19:24 GMT 2017]; root of context hierarchy
2017-03-14 20:19:26.029  INFO 7 --- [           main] o.s.j.e.a.AnnotationMBeanExporter        : Registering beans for JMX exposure on startup
2017-03-14 20:19:26.104  INFO 7 --- [           main] c.g.p.h.HazelcastDiscoveryController     : Asking k8s registry at https://kubernetes.default.svc.cluster.local..
2017-03-14 20:19:27.105  INFO 7 --- [           main] c.g.p.h.HazelcastDiscoveryController     : Found 1 pods running Hazelcast.
2017-03-14 20:19:27.387  INFO 7 --- [           main] c.h.instance.DefaultAddressPicker        : [LOCAL] [someGroup] [3.8] Interfaces is disabled, trying to pick one address from TCP-IP config addresses: [10.244.60.2]
2017-03-14 20:19:27.389  INFO 7 --- [           main] c.h.instance.DefaultAddressPicker        : [LOCAL] [someGroup] [3.8] Prefer IPv4 stack is true.
2017-03-14 20:19:27.412  INFO 7 --- [           main] c.h.instance.DefaultAddressPicker        : [LOCAL] [someGroup] [3.8] Picked [10.244.60.2]:5701, using socket ServerSocket[addr=/0:0:0:0:0:0:0:0,localport=5701], bind any local is true
2017-03-14 20:19:27.453  INFO 7 --- [           main] com.hazelcast.system                     : [10.244.60.2]:5701 [someGroup] [3.8] Hazelcast 3.8 (20170217 - d7998b4) starting at [10.244.60.2]:5701
2017-03-14 20:19:27.453  INFO 7 --- [           main] com.hazelcast.system                     : [10.244.60.2]:5701 [someGroup] [3.8] Copyright (c) 2008-2017, Hazelcast, Inc. All Rights Reserved.
2017-03-14 20:19:27.454  INFO 7 --- [           main] com.hazelcast.system                     : [10.244.60.2]:5701 [someGroup] [3.8] Configured Hazelcast Serialization version : 1
2017-03-14 20:19:28.062  INFO 7 --- [           main] c.h.s.i.o.impl.BackpressureRegulator     : [10.244.60.2]:5701 [someGroup] [3.8] Backpressure is disabled
2017-03-14 20:19:29.274  INFO 7 --- [           main] com.hazelcast.instance.Node              : [10.244.60.2]:5701 [someGroup] [3.8] Creating TcpIpJoiner
2017-03-14 20:19:29.657  INFO 7 --- [           main] c.h.s.i.o.impl.OperationExecutorImpl     : [10.244.60.2]:5701 [someGroup] [3.8] Starting 2 partition threads
2017-03-14 20:19:29.668  INFO 7 --- [           main] c.h.s.i.o.impl.OperationExecutorImpl     : [10.244.60.2]:5701 [someGroup] [3.8] Starting 3 generic threads (1 dedicated for priority tasks)
2017-03-14 20:19:29.688  INFO 7 --- [           main] com.hazelcast.core.LifecycleService      : [10.244.60.2]:5701 [someGroup] [3.8] [10.244.60.2]:5701 is STARTING
2017-03-14 20:19:29.721  INFO 7 --- [           main] com.hazelcast.system                     : [10.244.60.2]:5701 [someGroup] [3.8] Cluster version set to 3.8
2017-03-14 20:19:29.724  INFO 7 --- [           main] com.hazelcast.cluster.impl.TcpIpJoiner   : [10.244.60.2]:5701 [someGroup] [3.8]


Members [1] {
	Member [10.244.60.2]:5701 - f4854490-8b54-4e03-becd-6cd7620689f1 this
}

2017-03-14 20:19:29.839  INFO 7 --- [           main] com.hazelcast.core.LifecycleService      : [10.244.60.2]:5701 [someGroup] [3.8] [10.244.60.2]:5701 is STARTED
2017-03-14 20:19:29.845  INFO 7 --- [           main] com.github.pires.hazelcast.Application   : Started Application in 6.522 seconds (JVM running for 7.459)
2017-03-14 20:23:02.482  INFO 7 --- [thread-Acceptor] c.h.nio.tcp.SocketAcceptorThread         : [10.244.60.2]:5701 [someGroup] [3.8] Accepting socket connection from /10.244.76.3:44009
2017-03-14 20:23:02.502  INFO 7 --- [cached.thread-2] c.h.nio.tcp.TcpIpConnectionManager       : [10.244.60.2]:5701 [someGroup] [3.8] Established socket connection between /10.244.60.2:5701 and /10.244.76.3:44009
2017-03-14 20:23:09.491  INFO 7 --- [ration.thread-1] c.h.internal.cluster.ClusterService      : [10.244.60.2]:5701 [someGroup] [3.8]

Members [2] {
	Member [10.244.60.2]:5701 - f4854490-8b54-4e03-becd-6cd7620689f1 this
	Member [10.244.76.3]:5701 - be71c245-5a15-4592-9510-ddfb2ab48eee
}
```

Now let's scale our cluster to 4 nodes:
```sh
$ kubectl scale deployment hazelcast --replicas 4
```

Examine the status again by checking a node's logs and you should see the 4 members connected. Something like:
```
(...)

Members [4] {
	Member [10.244.60.2]:5701 - f4854490-8b54-4e03-becd-6cd7620689f1 this
	Member [10.244.76.3]:5701 - be71c245-5a15-4592-9510-ddfb2ab48eee
	Member [10.244.76.4]:5701 - 8c2a16bc-2c96-420f-90ca-66fafbdcbeda
	Member [10.244.60.3]:5701 - bb02eb25-d524-4076-ad24-a6f6f666e34d
}
```

### tl; dr;
For those of you who are impatient, here is the summary of the commands we ran in this tutorial.

```sh
kubectl create -f service.yaml
kubectl create -f deployment.yaml
kubectl scale deployment hazelcast --replicas 2
kubectl scale deployment hazelcast --replicas 4
```

### Hazelcast Discovery Source

```java
/**
 * Licensed under the Apache License, Version 2.0 (the "License"); you may not
 * use this file except in compliance with the License. You may obtain a copy of
 * the License at
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
 * WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
 * License for the specific language governing permissions and limitations under
 * the License.
 */
package com.github.pires.hazelcast;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.hazelcast.config.*;
import com.hazelcast.core.Hazelcast;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Controller;

import javax.net.ssl.*;
import java.io.IOException;
import java.net.URL;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.security.KeyManagementException;
import java.security.NoSuchAlgorithmException;
import java.security.SecureRandom;
import java.security.cert.X509Certificate;
import java.util.List;
import java.util.UUID;
import java.util.concurrent.CopyOnWriteArrayList;

/**
 * Read from Kubernetes API all Hazelcast service bound pods, get their IP and connect to them.
 */
@Controller
public class HazelcastDiscoveryController implements CommandLineRunner {

    private static final Logger log = LoggerFactory.getLogger(HazelcastDiscoveryController.class);

    @JsonIgnoreProperties(ignoreUnknown = true)
    static class Address {
        public String ip;
    }

    @JsonIgnoreProperties(ignoreUnknown = true)
    static class Subset {
        public List<Address> addresses;
        public List<Address> notReadyAddresses;
    }

    @JsonIgnoreProperties(ignoreUnknown = true)
    static class Endpoints {
        public List<Subset> subsets;
    }

    private static String getServiceAccountToken() throws IOException {
        String file = "/var/run/secrets/kubernetes.io/serviceaccount/token";
        return new String(Files.readAllBytes(Paths.get(file)));
    }

    private static String getEnvOrDefault(String var, String def) {
        final String val = System.getenv(var);
        return (val == null || val.isEmpty())
                ? def
                : val;
    }

    // TODO: Load the CA cert when it is available on all platforms.
    private static TrustManager[] trustAll = new TrustManager[]{
            new X509TrustManager() {
                public void checkServerTrusted(X509Certificate[] certs, String authType) {
                }

                public void checkClientTrusted(X509Certificate[] certs, String authType) {
                }

                public X509Certificate[] getAcceptedIssuers() {
                    return null;
                }
            }
    };
    private static HostnameVerifier trustAllHosts = new HostnameVerifier() {
        public boolean verify(String hostname, SSLSession session) {
            return true;
        }
    };

    @Override
    public void run(String... args) {
        final String serviceName = getEnvOrDefault("HAZELCAST_SERVICE", "hazelcast");
        final String namespace = getEnvOrDefault("POD_NAMESPACE", "default");
        final String path = String.format("/api/v1/namespaces/%s/endpoints/", namespace);
        final String domain = getEnvOrDefault("DNS_DOMAIN", "cluster.local");
        final String host = getEnvOrDefault("KUBERNETES_MASTER", "https://kubernetes.default.svc.".concat(domain));
        log.info("Asking k8s registry at {}..", host);

        final List<String> hazelcastEndpoints = new CopyOnWriteArrayList<>();
        try {
            final String token = getServiceAccountToken();

            final SSLContext ctx = SSLContext.getInstance("SSL");
            ctx.init(null, trustAll, new SecureRandom());

            final URL url = new URL(host + path + serviceName);
            final HttpsURLConnection conn = (HttpsURLConnection) url.openConnection();
            // TODO: remove this when and replace with CA cert loading, when the CA is propogated
            // to all nodes on all platforms.
            conn.setSSLSocketFactory(ctx.getSocketFactory());
            conn.setHostnameVerifier(trustAllHosts);
            conn.addRequestProperty("Authorization", "Bearer " + token);

            final ObjectMapper mapper = new ObjectMapper();
            final Endpoints endpoints = mapper.readValue(conn.getInputStream(), Endpoints.class);
            if (endpoints != null) {
                if (endpoints.subsets != null && !endpoints.subsets.isEmpty()) {
                    endpoints.subsets.forEach(subset -> {
                    	if (subset.addresses != null && !subset.addresses.isEmpty()) {
                        	subset.addresses.forEach(
                                addr -> hazelcastEndpoints.add(addr.ip));
                        } else if (subset.notReadyAddresses != null && !subset.notReadyAddresses.isEmpty()) {
                        	// in case of a full cluster restart
                        	// no address might be ready, in order to allow the cluster
                        	// to start initially, we will use the not ready addresses
                        	// as fallback
                        	subset.notReadyAddresses.forEach(
                                addr -> hazelcastEndpoints.add(addr.ip));
                        } else {
                        	log.warn("Could not find any hazelcast nodes.");
                        }
                    });
                }
            }
        } catch (IOException | NoSuchAlgorithmException | KeyManagementException ex) {
            log.warn("Request to Kubernetes API failed", ex);
        }

        log.info("Found {} pods running Hazelcast.", hazelcastEndpoints.size());

        runHazelcast(hazelcastEndpoints);
    }

    private void runHazelcast(final List<String> nodes) {
        // configure Hazelcast instance
        final Config cfg = new Config();
        cfg.setInstanceName(UUID.randomUUID().toString());
        // group configuration
        final String HC_GROUP_NAME = getEnvOrDefault("HC_GROUP_NAME", "someGroup");
        final String HC_GROUP_PASSWORD = getEnvOrDefault("HC_GROUP_PASSWORD",
                "someSecret");
        final int HC_PORT = Integer.parseInt(getEnvOrDefault("HC_PORT", "5701"));
        final String HC_REST_ENABLED = getEnvOrDefault("HC_REST_ENABLED", "false");
        cfg.setGroupConfig(new GroupConfig(HC_GROUP_NAME, HC_GROUP_PASSWORD));
        cfg.setProperty("hazelcast.rest.enabled", HC_REST_ENABLED);
        // network configuration initialization
        final NetworkConfig netCfg = new NetworkConfig();
        netCfg.setPortAutoIncrement(false);
        netCfg.setPort(HC_PORT);
        // multicast
        final MulticastConfig mcCfg = new MulticastConfig();
        mcCfg.setEnabled(false);
        // tcp
        final TcpIpConfig tcpCfg = new TcpIpConfig();
        nodes.forEach(tcpCfg::addMember);
        tcpCfg.setEnabled(true);
        // network join configuration
        final JoinConfig joinCfg = new JoinConfig();
        joinCfg.setMulticastConfig(mcCfg);
        joinCfg.setTcpIpConfig(tcpCfg);
        netCfg.setJoin(joinCfg);
        // ssl
        netCfg.setSSLConfig(new SSLConfig().setEnabled(false));
        // set it all
        cfg.setNetworkConfig(netCfg);
        // run
        Hazelcast.newHazelcastInstance(cfg);
    }

}
```
