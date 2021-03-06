# Federated Travels with OpenShift Service Mesh

Demo based on [Travels](../travels) application to show the federation capabilities of OpenShift Service Mesh.

## Platform Install

This demo has been tested using [CodeReady Containers 1.34](https://access.redhat.com/documentation/en-us/red_hat_codeready_containers/1.34/html/getting_started_guide) and [OpenShift Service Mesh 2.1](https://docs.openshift.com/container-platform/4.9/service_mesh/v2x/servicemesh-release-notes.html)

### CodeReady Containers

Set up your host machine for CodeReady Containers:

```bash
crc setup
```

Start the CodeReady Containers virtual machine with the recommended amount of RAM (16Gb):

```bash
crc start --memory 16384
```

Run the following command to add the cached oc executable to your PATH:

```bash
eval $(crc oc-env)
```

Log in as a cluster admin user to install the operators needed:

```bash
oc config use-context crc-admin
```

### OpenShift Service Mesh 2.1

Subscribe to the required operators (Jaeger, Kiali and OpenShift Service Mesh):

```bash
oc apply -f https://raw.githubusercontent.com/kiali/demos/master/federated-travels/ossm-subs.yaml
```

Install two independent meshes (*east* and *west*):

```bash
oc create namespace east-mesh-system
oc apply -n east-mesh-system -f https://raw.githubusercontent.com/kiali/demos/master/federated-travels/east/east-ossm.yaml

oc create namespace west-mesh-system
oc apply -n west-mesh-system -f https://raw.githubusercontent.com/kiali/demos/master/federated-travels/west/west-ossm.yaml
```

Create the namespaces for the Travels application:

```bash
oc create namespace east-travel-agency
oc create namespace east-travel-portal
oc create namespace east-travel-control
oc create namespace west-travel-agency
```

Wait for both control planes to be ready:

```bash
oc wait --for condition=Ready -n east-mesh-system smmr/default --timeout 300s
oc wait --for condition=Ready -n west-mesh-system smmr/default --timeout 300s
```

Create a configmap in each mesh namespace containing the root certificate that is used to validate client certificates in the trust domain used by the other mesh:

```bash
oc get configmap istio-ca-root-cert -o jsonpath='{.data.root-cert\.pem}' -n east-mesh-system > east-cert.pem
oc create configmap east-ca-root-cert --from-file=root-cert.pem=east-cert.pem -n west-mesh-system

oc get configmap istio-ca-root-cert -o jsonpath='{.data.root-cert\.pem}' -n west-mesh-system > west-cert.pem
oc create configmap west-ca-root-cert --from-file=root-cert.pem=west-cert.pem -n east-mesh-system
```

Configure the federation resources for travels application:

```bash
oc apply -n east-mesh-system -f https://raw.githubusercontent.com/kiali/demos/master/federated-travels/east/east-federation.yaml
oc apply -n west-mesh-system -f https://raw.githubusercontent.com/kiali/demos/master/federated-travels/west/west-federation.yaml
```
For more information about how federation works, visit the [documentation](https://docs.openshift.com/container-platform/4.9/service_mesh/v2x/ossm-federation.html).

## Application Install

Create the application resources:

```bash
oc apply -n east-travel-agency -f https://raw.githubusercontent.com/kiali/demos/master/federated-travels/east/east-travel-agency.yaml
oc apply -n east-travel-portal -f https://raw.githubusercontent.com/kiali/demos/master/federated-travels/east/east-travel-portal.yaml
oc apply -n east-travel-control -f https://raw.githubusercontent.com/kiali/demos/master/federated-travels/east/east-travel-control.yaml
oc apply -n west-travel-agency -f https://raw.githubusercontent.com/kiali/demos/master/federated-travels/west/west-travel-agency.yaml
```

## Demo Design

This demo creates two independent meshes (*east* and *west*) on the same OpenShift cluster and federates them to import/export services in both meshes.

The majority of the services are deployed on the *east* mesh and the *discounts* service is deployed on both *east* and *west* meshes.

The *east* mesh imports the *discounts* service from the *west* mesh. 

The services from *east* mesh consume both instances of *discounts*.

Each mesh has a Kiali instance: 

* East Kiali: https://kiali-east-mesh-system.apps-crc.testing
* West Kiali: https://kiali-west-mesh-system.apps-crc.testing

To observe the full topology, open both side by side, log in as kubeadmin user and go to the graph section:

![federated-travels](./federated-travels.png)

To obtain the kubeadmin user credentials, execute:

```bash
crc console --credentials
```

As an optional step, the following configuration will load balance the outcoming traffic from *cars* service including the *cars* workload deployed in *west* mesh, which in turn will consume the MySQL database from *east* mesh:

```bash
oc apply -n east-mesh-system -f https://raw.githubusercontent.com/kiali/demos/master/federated-travels/east/east-federation-opt.yaml
oc apply -n west-mesh-system -f https://raw.githubusercontent.com/kiali/demos/master/federated-travels/west/west-federation-opt.yaml
```

## Cleanup

Delete all namespaces:

```bash
oc delete namespace east-travel-agency east-travel-control east-travel-portal east-mesh-system west-travel-agency west-mesh-system 
```

Delete CodeReady Containers instance:

```bash
crc delete
```