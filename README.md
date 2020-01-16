# Deploying the Elastic Stack with ECK on GKE

This project contains instructions on how to install [ECK](https://www.elastic.co/guide/en/cloud-on-k8s/current/index.html) on Google's managed kubernetes ([GKE](https://console.cloud.google.com/kubernetes/)) and test a few of its capabilities.

It's intended as a step by step guide to further investigate ECK capabilities on GKE, not for production.

## Pre-requisites

- [Google Cloud SDK](https://cloud.google.com/sdk/install) configured on your laptop.
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/). At the moment, ECK requires version 1.11+. See [ECK requirements](https://www.elastic.co/guide/en/cloud-on-k8s/1.0/k8s-quickstart.html).
- [kubernetic](https://kubernetic.com/) (*Optional*). Kubernetes Desktop Client.

## Create a GKE cluster

- Go to [google cloud console](https://console.cloud.google.com), select [Kubernetes Engine](https://console.cloud.google.com/kubernetes/list) on the drop-down menu and create a GKE cluster.
- Choose the options depending on your needs. If you want to get automatic k8s upgrades, choose release channel and the speed at which to update (rapid, regular, stable). Otherwise, choose a specific version. Install on a kubernetes 1.12+. If you create a small cluster, you might need to update the size to run this example.

    ![Kubernetes cluster creation](./img/gke-creation.png)

- Once the cluster is up and running, connect kubectl. Replace with the appropiate cluster name (in the example `imma-k8s-cluster`) and zone (`europe-west1-b` in the example). Hitting the `connect` button in the console will also give you this command.

    ```shell
    gcloud container clusters get-credentials imma-k8s-cluster --zone europe-west1-b
    ```

- This command will fetch your k8s cluster endpoint and configure kubectl accordingly. The output should look like this:

    ```shell
    Fetching cluster endpoint and auth data.
    kubeconfig entry generated for imma-k8s-cluster.
    ```

- We should now be able to inspect the cluster using [kubectl commands](https://kubernetes.io/docs/reference/kubectl/cheatsheet/). For the purpose of this guide, we'll use kubernetic, and we'll also provide the kubectl commands.

    ![Kubernetes cluster overview](./img/kubernetic-1.png)

## ECK

- Installing ECK is a two step process. First we install the operator, and then we deploy our elastic stack (Elasticsearch, Kibana, APM). At the moment logstash or beats are not managed by the operator.
- After the stack is installed, we will demonstrate a few of the operations available at the moment, like scaling and upgrading.

### Install ECK

- In order to deploy the operator, we have to setup google cloud RBAC with the following command:

    ```shell
    kubectl create clusterrolebinding \
    cluster-admin-binding \
    --clusterrole=cluster-admin \
    --user=$(gcloud auth list --filter=status:ACTIVE --format="value(account)")
    ```

- Deploy the operator. For Kubernetes clusters running version 1.13 or higher:

    ```shell
    kubectl apply -f https://download.elastic.co/downloads/eck/1.0.0/all-in-one.yaml
    ```

- This will create the custom resources, cluster roles, namespace elastic-system, the operator, etc. for us.
- We can have a look at the operator logs:

    ```shell
    kubectl -n elastic-system logs -f statefulset.apps/elastic-operator
    ```

- Or get a description of the installed operator and each CustomResourceDefinition.

    ```shell
    kubectl -n elastic-system get statefulset.apps/elastic-operator

    kubectl -n elastic-system describe statefulset.apps/elastic-operator
    
    kubectl describe crd elasticsearch
    
    kubectl describe crd kibana
    
    kubectl describe crd apm
    ```

- Alternatively, we can inspect with kubernetic the namespace `elastic-system`. Select it at the top (change `default` to `elastic-system`) and navigate the different resources that the operator created.

    ![ECK Stateful Set](./img/kubernetic-2.png)

- The Stateful Sets contains our operator, and clicking on the operator pod we can view its logs.

    ![Operator](./img/kubernetic-3.png)
    ![Operator Logs](./img/kubernetic-4.png)

- Feel free to inspect all the other resources that the operator created (service accounts, secrets, config map). With kubernetic, this is easier to explore.

### Installing the Elastic Stack

- We will now install an Elastic Stack defined in the following file: [basic-complete-elastic-stack.yaml](basic-complete-elastic-stack.yaml).
- It's a simple definition for an Elastic Stack version 7.3.2, with a one-node Elasticsearch cluster, an APM a server and a single Kibana instance.
    - The Elasticsearch nodes are configured to limit container resources to 4G or RAM and 1 CPU.
    - The [Pod Template](https://www.elastic.co/guide/en/cloud-on-k8s/0.9/k8s-pod-template.html) would allow us to configure additional parameters like the Elasticsearch heap. 
    - The deployment will also mount on a 5Gb volume claim. Check the documentation for [Volume Claim Templates](https://www.elastic.co/guide/en/cloud-on-k8s/1.0/k8s-volume-claim-templates.html).

    ```shell
    kubectl apply -f basic-complete-elastic-stack.yaml
    ```

- We can monitor the deployment via kubectl:

    ```shell
    kubectl get elasticsearch,kibana,apmserver
    ```

- And check on the associated pods:

    ```shell
    kubectl get pods --selector='elasticsearch.k8s.elastic.co/cluster-name=elasticsearch-sample'
    kubectl get pods --selector='kibana.k8s.elastic.co/name=kibana-sample'
    kubectl get pods --selector='apm.k8s.elastic.co/name=apm-server-sample'
    ```

- Or with kubernetic. Since we did not specify a `namespace`, the elastic stack was deployed on the `default` namespace (remember to change the selected namespace at the top). We can first visit our `Services` and make sure they are all started.

    ![Elastic Services](./img/kubernetic-5.png)

- Since we deployed kibana with a service type LoadBalancer, we should be able to retrieve the external public IP GKE provisioned for us and access Kibana.

    ```yaml
    http:
        service:
        spec:
            type: LoadBalancer
    ```

- In order to do that, either get the external IP executing `kubectl get svc`. In the example:

    ```shell
    NAME                              TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)          AGE
    apm-server-sample-apm-http        ClusterIP      10.0.32.177   <none>          8200/TCP         102s
    elasticsearch-sample-es-default   ClusterIP      None          <none>          <none>           97s
    elasticsearch-sample-es-http      ClusterIP      10.0.36.228   <none>          9200/TCP         102s
    kibana-sample-kb-http             LoadBalancer   10.0.32.255   35.187.78.222   5601:30882/TCP   99s
    kubernetes                        ClusterIP      10.0.32.1     <none>          443/TCP          142m
    ```

- Or get it with kubernetic, viewing the `kibana-sample-kb-http` service.  When the load balancer is provisioned, we will see the external IP under `status.loadBalancer.ingress.ip`.

    ![Kibana external IP](./img/kubernetic-6.png)

- Once the external IP is available, we can visit our kibana at https://<EXTERNAL_IP>:5601/. In the example: https://35.187.78.222:5601/

- The certificate presented to us is self-signed one. We could have assigned a valid http certificate. See the [docs](https://www.elastic.co/guide/en/cloud-on-k8s/1.0/k8s-custom-http-certificate.html).

- The operator has created a secret for our superuser elastic. Let's go get it. Two options: via kubectl retrieve the password (remove the % at the end):

    ```shell
    kubectl get secret elasticsearch-sample-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode
    ```

    ```shell
    2bjlw2lfbwtjcd75nzbzpvkp%
    ```

- Or visit the secrets section using kubernetic. Filter for `elastic-user` (top right) and get the password in plain text under the `Specifications` section.

    ![Elastic User secret](./img/kubernetic-7.png)
    ![Elastic password](./img/kubernetic-8.png)

- Now we can log-in to our kibana with user `elastic`, and the retrieved password. We recommend [loading some kibana sample data](https://www.elastic.co/guide/en/kibana/7.3/add-sample-data.html), and enabling monitoring, so we can use the monitoring data in the following sections.

    ![Kibana monitoring](./img/kibana-monitoring-1.png)

### Scaling Elasticsearch

- Now that we have our stack up and running, we can scale Elasticsearch from 1 to 3 nodes. If we look at the pod section in kubernetes, or at the deployed pods, we will see that our Elasticsearch has only one node. If using kubernetic, don't forget to delete the filter on the top-left, or it will filter our pods and we won't see any.

    ```shell
    kubectl get pods
    ```

    ```shell
    NAME                                            READY   STATUS    RESTARTS   AGE
    apm-server-sample-apm-server-59788755b4-bfsx9   1/1     Running   0          8m3s
    elasticsearch-sample-es-default-0               1/1     Running   0          8m4s
    kibana-sample-kb-9f4cb94cd-9g5lz                1/1     Running   0          8m4s
    ```

    ![GKE pods](./img/kubernetic-9.png)

- To scale the Elastic Stack, edit the file [basic-complete-elastic-stack.yaml](basic-complete-elastic-stack.yaml) and set the Elasticsearch `nodeCount`to 3.

    ```yaml
    apiVersion: elasticsearch.k8s.elastic.co/v1
    kind: Elasticsearch
    metadata:
    name: elasticsearch-sample
    spec:
    version: 7.3.2
    nodeSets:
    - name: default
        count: 3
    ```

- Apply the changes:

    ```shell
    kubectl apply -f basic-complete-elastic-stack.yaml
    ```

- And monitor until the 3 pods are up and running. There is different options. Using kubectl command line, in the pods section of kubernetic, or in Kibana monitoring.

    ```shell
    kubectl get pods --selector='elasticsearch.k8s.elastic.co/cluster-name=elasticsearch-sample'

    NAME                                READY   STATUS     RESTARTS   AGE
    elasticsearch-sample-es-default-0   1/1     Running    0          9m37s
    elasticsearch-sample-es-default-1   0/1     Init:0/1   0          7s
    ```

    ![GKE pods](./img/kubernetic-10.png)

    ![GKE Elasticsearch service](./img/kubernetic-14.png)

    ![Kibana monitoring](./img/kibana-monitoring-3.png)

- When ready, we will have a 3-node elasticsearch cluster, and the health should now be green (all index replicas assigned).

### Upgrading the Elastic Stack

- We can now proceed to upgrade the whole stack. It will just require to edit the file [basic-complete-elastic-stack.yaml](basic-complete-elastic-stack.yaml) and replace all the `version: 7.3.2` with, for example, `version: 7.4.2` in the 3 services (elasticsearch, apm, kibana).

    ```shell
    kubectl apply -f basic-complete-elastic-stack.yaml
    ```

- When we apply the changes, the operator will take care of the dependencies. It will first update Elasticsearch and APM, and wait for Elasticsearch to complete to update kibana. We can follow the process of pod creation using kubectl or kubernetic.

    ```shell
    kubectl get pods
    ```

    ![GKE pods during upgrade](./img/kubernetic-15.png)

- As a default, the operator will upgrade Elasticsearch one instance at a time.
    - ECK uses StatefulSet-based orchestration from version 1.0+. StatefulSets with ECK allow for even faster upgrades and configuration changes, since upgrades use the same persistent volume, rather than replicating data to the new nodes.
    - We could also have changed the default [update strategy](https://www.elastic.co/guide/en/cloud-on-k8s/1.0/k8s-update-strategy.html) or the [Pod disruption budget](https://www.elastic.co/guide/en/cloud-on-k8s/1.0/k8s-pod-disruption-budget.html).

- After upgrading Elasticsearch, ECK will take care of upgrading the APM server and Kibana. For those, it will create new pods in version 7.4.2 to replace the old ones version 7.3.2.

- We can check the deployed instances using kubernetic. Visualizing any of the pod specifications we can see that they are now running version 7.4.2.

    ![Elasticsearch version 7.4.2](./img/kubernetic-12.png)

- We can also see in the kibana monitoring UI the health of our cluster (green) with 3 nodes on version 7.4.2.

    ![Kibana monitoring](./img/kibana-monitoring-3-upgraded.png)

## Uninstall process

When we are done with the testing, it is recommended to follow the uninstall process to liberate resources.

### Delete elastic resources

- Remove elastic resources from all namespaces.

    ```shell
    kubectl delete elastic --all --all-namespaces
    ```
​
- Remove the operator.

    ```shell
    kubectl delete -f https://download.elastic.co/downloads/eck/1.0.0/all-in-one.yaml
    ```

### Delete GKE cluster

Go into google cloud console and remove your Google Kubernetes Engine.
