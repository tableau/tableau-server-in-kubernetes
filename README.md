# Tableau Server In Kubernetes
This project consists of documentation and examples demonstrating how to run Tableau Server in an existing Kubernetes cluster. Make sure the Tableau Server image used is sourced from the Tableau Server in a Container Project.

Tableau Server in a Container Project (documentation)

# Project Resources
Kubernetes example templates are stored in the `templates/` directory of this project. There are single-node, multi-node, and upgrade job templates that can be used as starting points.

# Requirements
There are many different kinds of Kubernetes deployments which can affect how Tableau Server might be deployed in a container orchestration system. To account for this variability and give you the information to make the best decision for how to deploy Tableau Server, this section will go over the high-level requirements for what Tableau Server needs in order to run properly in a container orchestration system.

## Single Node Requirements
### Network
Hostnames inside the container must be static and consistent. This means on every restart of the container or pod, the hostname in the container must stay the same. This is one of the primary reasons we recommend using the StatefulSet workload API object to deploy the Tableau Server pod.
Persistent volumes that store Tableau Server state also expect to be used with the same container hostname.
Tableau Server containers receive client traffic on port 8080 by default (8443 for TLS).

## Multi-Node Requirements
### Network
Container short hostnames must be DNS resolvable; Tableau Server containers in a cluster will self-register and discover each other by their short hostname. This means a standard Kubernetes deployment using core-dns or kube-dns will require customizing the container's DNS policy. Check the example Kubernetes multinode configuration to see one way of handling this. The [Kubernetes documentation on DNS lookups](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/) provides more information on this topic.
Using a Headless Service is recommended because every Tableau Server pod will get a DNS entry with that deployment model. The [Kubernetes documentation on headless services](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services) provides more details on how this works.

### Bootstrap file
A bootstrap file must be shared between the initial node and all subsequent workers. We recommend using an NFS mount to facilitate the sharing of this file (this would enable the multi-node deployment to be fully automated). If you are using AWS you can use EFS to the same effect. Check the Multinode bootstrap file section for more details. The Kubernetes multinode template shows one possible way of handling this.

# Documentation
## Status
There are two status checks provided for Tableau Server. These can be used by an orchestration system, like Kubernetes, to determine whether Tableau Server is starting or not.

### Aliveness Check
The aliveness check indicates whether or not TSM services are running. This means it will indicate whether the orchestrated services of Tableau Server are operating and are functioning. This check is callable here:
```
/docker/alive-check
```
Another option is to expose the TSM Controller service (running on port 8850) to provide administrative functions through a web browser. One could periodically check the health of the service by checking the health of the service through TCP health checks.

### Readiness Check
The readiness check indicates whether Tableau Server is running and business services are ready to receive traffic. This can be determined using the following script:
```
/docker/server-ready-check
```
Another option is to use TCP health checks against port 8080 (or whatever port Tableau Server is bound to receive traffic). Sometimes this kind of TCP health check is more reliable than the server-ready-check, as the server-ready-check is based on service status reported to TSM which can sometimes be delayed as service state is updated.

## Resource Limits
We strongly recommend that deployments to Kubernetes set appropriate resource limits for the deployed containers. Note that Tableau Server has significant resource requirements, make sure that the resource limits you specify matching at least the [Tableau Server resource requirements for testing and production](https://help.tableau.com/current/server-linux/en-us/server_hardware_min.htm).

## Network Properties
Tableau Server does not handle container hostname changes well, so it is important to specify the container's internal hostname so it is consistent between container runs.

Tableau Server nodes in a cluster communicate by between nodes by registering their container hostname amongst the other Tableau Server nodes. This means the container hostname must be resolvable by DNS.

## Deployment Properties
We recommend using StatefulSets and persistent volume claims when deploying Tableau Server in Kubernetes. At the moment Tableau Server is a stateful application so appropriate measures should be taken to preserve and backup Tableau's application state. Future advancements will loosen these requirements.
