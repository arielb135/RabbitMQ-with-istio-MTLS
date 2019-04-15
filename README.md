# RabbitMQ-with-istio-MTLS
RabbitMQ stateful deployment with istio service mesh, and with MTLS enabled.
This repository is a clone of https://github.com/helm/charts/tree/master/stable/rabbitmq with an istio MTLS integration.

## Introduction
Istio (atleast 1.0) does not support fully stateful set deployments - which is discussed in many threads around the web.
There were several workarounds that were suggested, and in this respository i'll explain how i managed to deploy a rabbitMQ deployment with istio and MTLS enabled.

The deployment had 3 main issues i've encountered:
## Issue 1 - Stateful sets pod discovery
Usually when creating a a deployment - we have multiple stateless service that are independent from each other.
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

Workaround

To workaround this, we have 2 options:

change the application to use localhost/127.0.0.1 instead of pod ip
if #1 is not possible, exclude MTLS communication from this specific communication
in rabbitMQ - epmd which is the service that does cluster discovery talks in this local pod IP - which causes failure in the MTLS, this issue talks about it:

https://github.com/istio/istio/issues/6828

so in rabbitMQ you have to exclude this port (thus, service discovery will not be done encrypted) - always consult with security to see what it means to us.



this means:

Disable mtls for service called "rabbitmq" (in default namespace - as there's no full DNS name specified) - of port 4369 (the problematic one that talks locally in pod IP)
Headless service global DNS cannot participate in MTLS
Issue

Headless service is the service that gives the pods their IP.

as kubernetes gives the headless service also a full DNS name (like regular service) - it's not a good idea to use that DNS to talk inside the cluster

Workaround

Create another service without a type, which will not attach it to any load balancer - and you'll have a internal service that we can use (with the DNS - carabbitmq-internal.[namespace].svc.cluster.local)

istio will discover it well.
