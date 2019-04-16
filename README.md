# RabbitMQ-with-istio-MTLS
RabbitMQ stateful deployment with istio service mesh, and with MTLS enabled.
This repository is a clone of https://github.com/helm/charts/tree/master/stable/rabbitmq with an istio MTLS integration.

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
Thankfully, istio gave us an istio gateway that can be used as the entry point if we want to go inside the cluster, we can terminate TLS there (to the outside world) for example.

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
  name: carabbitmq
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

This is done aswell by the policy object, we can see an example:
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
  name: carabbitmq-disable-mtls
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
```
* In here, we are addressing 2 services - the first is the headless service, which we want to exclude the epmd port and also the amqps (probably unecessary). The second is the regular service, that in this chart we expose it via load balancer for example, so we want to omit MTLS from it.
* Now we have the entire rabbitMQ services require MTLS to work - excluding 4369 and 5671

To configure the "client" (the rabbitMQ pods themselves so they can initiate MTLS connection as a "client") - we need to set the destination rule:
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
  name: rabbitmq-dr
  namespace: rabbitns
spec:
  host: '*.rabbitns.svc.cluster.local'
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
```
* The above means that we would like a mutual TLS for everything that is directed to "*.rabbitns.svc.cluster.local".
* Because we deployed it in the rabbitns namespace, it will apply to whole "clients" of that namespace.

If we'll have another rabbit client in another namespace (let's say javarabbitclient), we would create the same rule, but in that specific namespace:
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
  host: '*.rabbitns.svc.cluster.local'
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
```

This sketch illustrates what happens:
![TLS](https://github.com/arielb135/RabbitMQ-with-istio-MTLS/blob/master/sketches/tls.png)
* The destination rule is deployed in the client's namespace, and directs that it wants to talk as MTLS to the host *.rabbitns.svc.cluster.local
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
$ istioctl authn tls-check handler-6f936c59f5-4d88d.javarabbitclient rabbitmq-internal.rabbitns.svc.cluster.local
```
* This checks the tls of the handler pod (our java service) dot namespace ({pod}.{namespace}) to the service it requests (which is actually the rabbitmq-internal internal service that is exposed in the rabbit namespace)

we can now see the output:
``` bash
HOST:PORT                                             STATUS     SERVER     CLIENT     AUTHN POLICY         DESTINATION RULE
rabbitmq-internal.rabbitns.svc.cluster.local:5671     OK         mTLS       mTLS       default/rabbitmq     java-dr/javaservice
```
which means both Server (policy) and Client (destination rule) are configured for mTLS for the 5671 port.
