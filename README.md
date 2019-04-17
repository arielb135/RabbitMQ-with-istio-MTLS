# RabbitMQ-with-istio-MTLS
RabbitMQ stateful deployment with istio service mesh, and with MTLS enabled.
This repository is a clone of https://github.com/helm/charts/tree/master/stable/rabbitmq-ha with an istio MTLS integration.
All charts are under **helm** in this repository, and installation is similar to rabbitmq-ha.

## Introduction
Istio (atleast 1.0) does not support fully stateful set deployments - which is discussed in many threads around the web.
There were several workarounds that were suggested, and in this respository i'll explain how i managed to deploy a rabbitMQ deployment with istio and MTLS enabled.

> note - This chart was not tested with whole combinations, use with cation.

A typical deployment of this chart can look like this:
![istio](https://github.com/arielb135/RabbitMQ-with-istio-MTLS/blob/master/sketches/istio.png)
1. the istio gateway exposes the UI
2. the "unknown" is another load balancer that exposes only the AMQPS port in case of rabbit implemented MTLS (this should be fixed, not in scope)
3. the handler is a java app that access rabbit in AMQP port (using istio MTLS)

> this dashboard came from kiali, which should be enabled when installing istio, more info: https://istio.io/docs/tasks/telemetry/kiali/

## Installing ISTIO
Istio is pretty easy to install, i've used istio 1.0 and followed the following tutorial, while enabling auto sidecar injection:
* https://istio.io/docs/setup/kubernetes/install/helm/
* https://istio.io/docs/setup/kubernetes/additional-setup/sidecar-injection/#automatic-sidecar-injection

## Labeling namespaces for injection
We will want that the istio sidecar will be injected automatically to rabbitMQ namespace (rabbitns let's say) - so we need to label that namespace (after creating it) to allow it:
```bash
$ kubectl create ns rabbitns
```
Then we will label this namespace so all pods will automatically have the istio sidecar container:
```bash
$ kubectl label namespace rabbitns istio-injection=enabled
```

## Values.yaml istio support
The part of the yaml that supports istio is this:
``` yaml
istio:
  enabled: true
  mtls: true
  ingress:
    enabled: true
    managementHostName: rabbit.mycooldomain.com
```
* ingress part defines a record in the ingress gateway
## Exposing rabbitMQ UI in the ingress gateway
Thankfully, istio gave us an istio gateway that can be used as the entry point if we want to go inside the cluster, we have additional functionality using the gateway, for example - we can terminate TLS there (to the outside world) (not done here).

``` yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  labels:
    app: rabbitmq-ha
    chart: rabbitmq-ha-1.12.1
    heritage: Tiller
    release: rabbitmq
  name: rabbitmq-gw
  namespace: rabbitns
spec:
  selector:
    istio: ingressgateway # use default istio gateway
  servers:
  - hosts:
    - rabbit.mycooldomain.com
    port:
      name: http
      number: 80
      protocol: HTTP
```
* This tells the istio gateway to do something with traffic that comes with the host rabbit.mycooldomain.com on port 80.

To actually perform the routing to the service, we must use a VirtualService to do so:
``` yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  labels:
    app: rabbitmq-ha
    chart: rabbitmq-ha-1.12.1
    heritage: Tiller
    release: rabbitmq
  name: rabbitmq-vs
  namespace: rabbitns
spec:
  gateways:
  - rabbitmq-gw
  hosts:
  - rabbit.mycooldomain.com
  http:
  - route:
    - destination:
        host: rabbitmq-internal.rabbitns.svc.cluster.local
        port:
          number: 15672 # rabbitmq management UI port
```
* In here we're telling to rabbit.mycooldomain.com to go to the internal service rabbitmq-internal.rabbitns.svc.cluster.local (which exposes the rabbit UI) with port 15672 (management UI).

![istio](https://github.com/arielb135/RabbitMQ-with-istio-MTLS/blob/master/sketches/ui.png)

The istio objects that were created come to respond to 3 main issues - 
## Issue 1 - Stateful sets pod discovery
Usually when creating a deployment - we have multiple stateless service that are independent from each other.
Stateful apps such as databases, rabbit are participating in a cluster, which means they require stable DNS of each pod so they can exchange information between them, This is not possible in a regular deployment, as the DNS is not stable (nor the IPs).
this presents us with the "stateful set" option - which assures:

- Stable DNS names for each pod - for example - rabbitmq-0.rabbitmq-discovery.rabbitns.svc.cluster.local
  - rabbitmq-0 - is the name of the statefulset (deployment) with dash and then a running number from 0 to number of replicas - 1 (this assures a known and predictable DNS name)
  - arabbitmq-discovery - name of the headless service that gives the stable DNS to the pods
  - rabbitns - the namespace the app is deployed into
  - Stable storage persistence - which means that when pod is rescheduled, it will reuse the same storage attached to it
- ordered deployment and shutdown (one by one - which is good if a cluster needs to be set up - as the first node creates the cluster, the second joins, etc..)

More info: https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/
* Headless service - this is a service without a cluster IP that is responsible in giving the pods their DNS name.

### Issue + workaround
Istio discovers well regular services (that attach 1 single DNS to all the pods and then they are accessed in round robin style), but it is not familliar with per pod DNS - thus it rejects it.
We will thus create a service entry per pod, and expose all ports that are needed - telling istio that they are inside the service mesh, example of a ServiceEntry
In rabbitMQ - this looks like this:

``` yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  labels:
    app: rabbitmq-ha
    chart: rabbitmq-ha-1.12.1
    heritage: Tiller
    release: rabbitmq
  name: rabbitmq
  namespace: rabbitns
spec:
  hosts:
  - rabbitmq-0.rabbitmq-discovery.rabbitns.svc.cluster.local
  - rabbitmq-1.rabbitmq-discovery.rabbitns.svc.cluster.local
  location: MESH_INTERNAL
  ports:
  - name: http
    number: 15672
    protocol: TCP
  - name: amqp
    number: 5672
    protocol: TCP
  - name: epmd
    number: 4369
    protocol: TCP
  - name: amqps
    number: 5671
    protocol: TCP
  - name: exporter
    number: 9419
    protocol: TCP
  - name: inter-node
    number: 25672
    protocol: TCP
  resolution: NONE
```
* the full DNS of each pod (received from a kubernetes [headless service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services)).
* All ports that the service used need to be declared, so istio can register them.
* The helm chart creates the list of hosts dynamically by replica count.


### more info:
* general istio thread for stateful sets - https://github.com/istio/istio/issues/10659
* Headless service solution - https://github.com/istio/istio/issues/7495

## Issue 2 - Mutual TLS for POD IP communication
### Issue

When enabling mutual TLS, we need to configure the clients - so they will also talk in MTLS to the service that was enabled with MTLS.
This is usually done by DestinationRule. Some applications require to talk to the localhost for some logic they have, istio lets all loopback interfaces pass without MTLS requirement (127.0.0.1 and localhost).

when a pod uses its pod IP for local communication, it will fail as the MTLS policy is default deny - istio yet to support talking locally with POD IP and mtls, Cassandra is a simple example to that:
https://aspenmesh.io/2019/03/running-stateless-apps-with-service-mesh-kubernetes-cassandra-with-istio-mtls-enabled/

### Workaround

To workaround this, we have 2 options:

- change the application to use localhost/127.0.0.1 instead of pod ip - this is possible for example in cassandra, but not in RabbitMQ
- if #1 is not possible, we will need to exclude MTLS communication from this specific service (usually a port), in rabbitMQ it's epmd which is the service that does cluster discovery. This service talks in local pod IP - which causes failure in the MTLS, this issue talks about it: https://github.com/istio/istio/issues/6828

so in rabbitMQ you have to exclude this port (thus, service discovery will not be done encrypted) - always consult with security architect / anyone else to see what it means to the deployment.

**Policy:**
This is done aswell by the policy object, we can see an example
``` yaml
apiVersion: authentication.istio.io/v1alpha1
kind: Policy
metadata:
  labels:
    app: rabbitmq-ha
    chart: rabbitmq-ha-1.12.1
    heritage: Tiller
    release: rabbitmq
  name: default
  namespace: rabbitns
spec:
  peers:
  - mtls: {}
```
* Which enables the mtls to the *whole* rabbitns namespace

We know that we have a problem with the epmd (4369 port) service which tries to access the local pod IP, instead of localhost - so we'll exclude it.
We'll also exclude the secured AMQPS port - as if it's configured - we will now do a TLS over TLS (not needed).
``` yaml
apiVersion: authentication.istio.io/v1alpha1
kind: Policy
metadata:    
  labels:
    app: rabbitmq-ha
    chart: rabbitmq-ha-1.12.1
    heritage: Tiller
    release: rabbitmq
  name: rabbitmq-disable-mtls
  namespace: rabbitns
spec:
  targets:
  - name: rabbitmq-discovery
    ports:
    - number: 4369
    - number: 5671
  - name: rabbitmq
    ports:
    - number: 5671
  - name: rabbitmq-internal
    ports:
    - number: 5671
```
* In here, we are addressing 3 services - the first is the headless service, which we want to exclude the epmd port and also the amqps (probably unecessary). The second is the regular service, that in this chart we expose it via load balancer for example, so we want to omit MTLS from it, third one is the internal service - which we also want to exclude 5671 mtls port.
* Now we have the entire rabbitMQ services require MTLS to work - excluding 4369 and 5671

> Important - there can be only one per namespace policy - we are utilizing it in this chart, meaning - you should deploy rabbit in its own namespace to avoid problems - https://istio.io/docs/concepts/security/#target-selectors

**Destination Rule:**
To configure the "client" (the rabbitMQ pods themselves so they can initiate MTLS connection as a "client") - we need to create several destination rules - this is a bit tricky.
The deployment is divided to 4 destination rules (the "clients"), to avoid conflicts in cluster (even if some of the ports are not used in some services)
* Per pod - ***.rabbitmq-discovery.rabbitns.svc.cluster.local** - which sets a full MTLS for all ports (for the client side)
* exposed through ingress service - **rabbitmq.rabbitns.svc.cluster.local** - full MTLS except amqps port (5671)
* headless service - **rabbitmq-discovery.rabbitns.svc.cluster.local** - full MTLS except amqps port (5671) and epmd port (4369)
* internal service - **rabbitmq-internal.rabbitns.svc.cluster.local** - full MTLS except amqps port (5671)
``` yaml
apiVersion: v1
items:
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  labels:
    app: rabbitmq-ha
    chart: rabbitmq-ha-1.12.1
    heritage: Tiller
    release: rabbitmq
  name: rabbitmq-mtls-per-pod
  namespace: rabbitns
spec:
  host: '*.rabbitmq-discovery.rabbitns.svc.cluster.local'
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
---
apiVersion: v1
items:
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  labels:
    app: rabbitmq-ha
    chart: rabbitmq-ha-1.12.1
    heritage: Tiller
    release: rabbitmq
  name: rabbitmq-mtls
  namespace: rabbitns
spec:
  host: 'rabbitmq.rabbitns.svc.cluster.local'
  trafficPolicy:
    portLevelSettings:
    - port:
        number: 5671
      tls:
        mode: DISABLE
    tls:
      mode: ISTIO_MUTUAL
---
apiVersion: v1
items:
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  labels:
    app: rabbitmq-ha
    chart: rabbitmq-ha-1.12.1
    heritage: Tiller
    release: rabbitmq
  name: rabbitmq-mtls
  namespace: rabbitns
spec:
  host: 'rabbitmq-discovery.rabbitns.svc.cluster.local'
  trafficPolicy:
    portLevelSettings:
    - port:
        number: 4369
      tls:
        mode: DISABLE
    - port:
        number: 5671
      tls:
        mode: DISABLE
    tls:
      mode: ISTIO_MUTUAL
---
apiVersion: v1
items:
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  labels:
    app: rabbitmq-ha
    chart: rabbitmq-ha-1.12.1
    heritage: Tiller
    release: rabbitmq
  name: rabbitmq-mtls-internal
  namespace: rabbitns
spec:
  host: 'rabbitmq-internal.rabbitns.svc.cluster.local'
  trafficPolicy:
    portLevelSettings:
    - port:
        number: 5671
      tls:
        mode: DISABLE
    tls:
      mode: ISTIO_MUTUAL
```
* The above means that we would like a mutual TLS for everything that is directed to either the pod themselves, or 
* We want to avoid conflicts, so we will disable ports that don't require MTLS for each service / host.
** 5671 is the AMQPS port - which we want the client to use it regulary.
* Because we deployed it in the rabbitns namespace, it will apply to whole "clients" of that namespace.

> Note - it's ok to create one destination rule that catchs all ***.rabbitns.svc.cluster.local**, but you will have conflicts when checking tls settings - this might be ok even in istio strict TLS mode - but requires more testing

If we'll have another rabbit client in another namespace (let's say javarabbitclient), we would create the a simpler rule for the internal service , but in that specific namespace:
``` yaml
apiVersion: v1
items:
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  labels:
    app: rabbitmq-ha
    chart: rabbitmq-ha-1.12.1
    heritage: Tiller
    release: rabbitmq
  name: java-dr
  namespace: javarabbitclient
spec:
  host: 'rabbitmq-internal.rabbitns.svc.cluster.local'
  trafficPolicy:
    portLevelSettings:
    - port:
        number: 5671
      tls:
        mode: DISABLE
    tls:
      mode: ISTIO_MUTUAL
```

This sketch illustrates what happens:
![TLS](https://github.com/arielb135/RabbitMQ-with-istio-MTLS/blob/master/sketches/tls.png)
* The destination rule is deployed in the client's namespace, and directs that it wants to talk as MTLS to the service rabbitmq-internal.rabbitns.svc.cluster.local
* The policy says to enable MTLS for the whole namespace, but exclude certain ports

## Issue 3 - Headless service DNS entry cannot participate in MTLS

### Issue
Headless service is the service that gives the pods their stable DNS record.

as kubernetes gives the headless service also a full DNS name (like regular service), which can be used as a round robin style aswell. - it's not a good idea to use that DNS to talk inside the cluster anyway, as it used for distribution of DNS entries to per-pod, and for good order - it's recommended to create a seperate internal service for internal communication.

### Workaround
Create another service without a type (clusterIP), which will not attach it to any load balancer - and you'll have a internal service that we can use (with the DNS - rabbitmq-internal.rabbitns.svc.cluster.local)
istio will discover it well.

``` yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: rabbitmq-ha
    chart: rabbitmq-ha-1.12.1
    heritage: Tiller
    release: rabbitmq
  name: rabbitmq-internal
  namespace: rabbitns
spec:
  ports:
  - name: http
    port: 15672
    protocol: TCP
    targetPort: http
  - name: amqp
    port: 5672
    protocol: TCP
    targetPort: amqp
  - name: amqps
    port: 5671
    protocol: TCP
    targetPort: amqps
  selector:
    app: rabbitmq-ha
    release: rabbitmq
```

## Verifying TLS
taken from here: https://istio.io/docs/tasks/security/mutual-tls/#verify-mutual-tls-configuration

we will use the istioctl tool, with the tls-check option,
``` bash
$ istioctl authn tls-check handler-6f936c59f5-4d88d.javarabbitclient
```
* This checks the tls of the handler pod (our java service) dot namespace ({pod}.{namespace}) in the mesh. as we're using the rabbitmq-internal - we can look at the entries of it:

we can now see the output:
``` bash
HOST:PORT                                             STATUS     SERVER     CLIENT     AUTHN POLICY                   DESTINATION RULE
rabbitmq-internal.rabbitns.svc.cluster.local:5671     OK         HTTP       HTTP       rabbitmq-disable-mtls/rabbitns handler/javarabbitclient
rabbitmq-internal.rabbitns.svc.cluster.local:5672     OK         mTLS       mTLS       default/rabbitns               handler/javarabbitclient
rabbitmq-internal.rabbitns.svc.cluster.local:9419     OK         mTLS       mTLS       default/rabbitns               handler/javarabbitclient
rabbitmq-internal.rabbitns.svc.cluster.local:15672    OK         mTLS       mTLS       default/rabbitns               handler/javarabbitclient
```
We can see several interesting entries here:
* Port 5671 - which is the secured AMQPS port that we decided to exclude from MTLS communication
* Port 5672 - regular unencrypted AMQP port - will now be encrypted under istio mTLS
* Port 9419 - is for metrics scraping (Note - this was not tested if it works, so prometheus integration and istio configuration is out of scope for this charts)
* Port 15672 - the management UI

## Unrelated chart features
### Reoccuring definitions
There's an ability to add reoccuring entries to definitions.json with "definitions_reoccuring":
``` yaml
definitions_reoccuring:
  enabled: true
  replaceString: "#"
  startFrom: 101
  until: 120
## Must be string values ( e.g. ["102", "103"]
  skipIndexes: ["110"] 
  users: |-
    {
      "name": "user#",
      "tags": ""
    }
  permissions: |-
    {
      "user": "user#",
      "vhost": "/",
      "configure": ".*",
      "write": ".*",
      "read": ".*"
    }
```
In the above we will create 19 users (from 101 to 120, excluding 110) that are called user user101, user102 etc...
There's the exclusion field to skip some users.
This supports any entry in definitions.json - and the string to replace with number is #
