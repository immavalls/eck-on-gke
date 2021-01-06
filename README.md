# Deploying the Elastic Stack with ECK on GKE

This project contains instructions on how to install [ECK](https://www.elastic.co/guide/en/cloud-on-k8s/current/index.html) on Google's managed kubernetes ([GKE](https://console.cloud.google.com/kubernetes/)) and test a few of its capabilities.

It's intended as a step by step guide to further investigate ECK capabilities on GKE, not for production.

## Pre-requisites

- [Google Cloud SDK](https://cloud.google.com/sdk/install) configured on your laptop/workstation.
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/). At the moment, ECK requires version 1.11+. See [ECK requirements](https://www.elastic.co/guide/en/cloud-on-k8s/1.3/k8s-quickstart.html).
- [jq](https://stedolan.github.io/jq//) (*Optional*). Command-line JSON processor.
- [kubernetic](https://kubernetic.com/) (*Optional*). Kubernetes Desktop Client.

## Deploy a GKE cluster and install ECK

To run the following examples, follow [these steps](./install-gke-and-eck.md) to deploy a GKE cluster and install ECK.

## Examples

1. [Basic Elastic Stack Deployment](./basic/basic-elastic-stack.md)
2. [Deploy the Elastic Stack and a dedicated Monitoring Cluster](./monitoring/monitoring-stack.md).
3. TODO - Hot/Warm/Cold deployment in a regional cluster.
4. TODO - Spring petclinic with APM. Based on [elastic/spring-petclinic](https://github.com/elastic/spring-petclinic).

## Clean-up

When we are done with the testing, it is recommended to follow the uninstall process for each example, to liberate k8s resources like Load Balancers or Persistent Volumes Claims. 

Once this is done for each example, we can remove the operator.

    ```shell
    kubectl delete -f https://download.elastic.co/downloads/eck/1.3.1/all-in-one.yaml
    ```

And delete the GKE cluster. Login to Google Cloud Console and remove the cluster you created.

Alternatively, use the script:

```bash
#!/bin/bash
## Deletes a GKE cluster

# Cluster attributes (GCP project, zone, name and sizing)
gcp_project="immas-k8s-project"
gcp_zone="europe-west3-c"
gke_cluster_name="imma-gke"

## Destroy cluster
if [[ $(gcloud container clusters list 2> /dev/null --project ${gcp_project} | grep ${gke_cluster_name} | wc -l) -gt 0 ]]
then gcloud container clusters delete ${gke_cluster_name} --project ${gcp_project} --zone=${gcp_zone} --quiet;
else echo "cluster ${gke_cluster_name} not found"
fi

## Remove kubectl context
kubectl config unset contexts.${gke_cluster_name}
```
