# Deploy the ELK stack and a monitoring cluster

Based on https://www.elastic.co/blog/elastic-stack-monitoring-with-elastic-cloud-on-kubernetes, we'll be deploying:

- An Elasticsearch cluster, APM server and Kibana instance that will be monitored.
- An Elasticsearch cluster and Kibana instance to monitor the stack.
- Beats to collect Kubernetes logs & metrics and send those to the monitored cluster. Those beats will also be monitored.
- A monitoring Metricbeat to collect metrics and a monitoring Filebeat to send logs to the Elasticsearch monitoring cluster.

## Prerequisites

- Follow instructions to create a GKE cluster [here](../README.md#create-a-gke-cluster).
- Install ECK as detailed [here](../README.md#install-eck).
- Apply a license: https://www.elastic.co/guide/en/cloud-on-k8s/1.3/k8s-licensing.html

    - Either start a trial:

        ```yaml
        cat <<EOF | kubectl apply -f -
        apiVersion: v1
        kind: Secret
        metadata:
        name: eck-trial-license
        namespace: elastic-system
        labels:
            license.k8s.elastic.co/type: enterprise_trial
        annotations:
            elastic.co/eula: accepted 
        EOF
        ```

    - Or install your license.
kubectl
        ```shell
        kubectl create secret generic eck-license --from-file=my-license-file.json -n elastic-system
        kubectl label secret eck-license "license.k8s.elastic.co/scope"=operator -n elastic-system
        ```

    - Check the installed license `kubectl -n elastic-system get configmap elastic-licensing -o json | jq .data`.

## Deploy the monitored stack (Elasticsearch, Kibana, APM)

- Deploy the stack with [01-es-kb-apm-monitored.yaml](./01-es-kb-apm-monitored.yaml).

```shell
> kubectl apply -f 01-es-kb-apm-monitored.yml

elasticsearch.elasticsearch.k8s.elastic.co/elasticsearch created
kibana.kibana.k8s.elastic.co/kibana created
apmserver.apm.k8s.elastic.co/apm created
```

- To access Kibana, get the public IP for the load balancer:

```shell
> kubectl get svc --selector='kibana.k8s.elastic.co/name=kibana'

NAME             TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE
kibana-kb-http   LoadBalancer   10.119.251.220  35.246.188.101  5601:31079/TCP   54s
```

- And get elastic user password:

```shell
> echo $(kubectl get secret elasticsearch-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode)

h25eb3gnW5Bj0N30M9qXS6m1
```

- Open your browser in https://EXTERNAL-IP:5601, in the case above https://35.246.188.101:5601

## Deploy the monitoring stack (Elasticsearch, Kibana)

- Deploy an Elasticsearch cluster and Kibana instance for monitoring purposes.

```shell
> kubectl apply -f 02-monitoring-es-kb.yml

elasticsearch.elasticsearch.k8s.elastic.co/elasticsearch-monitoring created
kibana.kibana.k8s.elastic.co/kibana-monitoring created
```

- To access Kibana, get the public IP for the load balancer:

```shell
> kubectl get svc --selector='kibana.k8s.elastic.co/name=kibana-monitoring'

NAME                        TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE
kibana-monitoring-kb-http   LoadBalancer   10.119.243.94   34.89.147.219   5601:30156/TCP   71s
```

- And get elastic user password:

```shell
> echo $(kubectl get secret elasticsearch-monitoring-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode)

j42f6oT8OrOSXC735I885Spu
```

- Open your browser in https://EXTERNAL-IP:5601, in the case above https://34.89.147.219:5601

## Deploy some monitored Beats

- Now we'll deploy few beats: Heartbeat, Filebeat, Metricbeat.

```shell
> kubectl apply -f 03-heartbeat-monitored.yml 
beat.beat.k8s.elastic.co/heartbeat created

> kubectl apply -f 03-metricbeat-monitored.yml
beat.beat.k8s.elastic.co/metricbeat created
clusterrole.rbac.authorization.k8s.io/metricbeat created
serviceaccount/metricbeat created
clusterrolebinding.rbac.authorization.k8s.io/metricbeat created

> kubectl apply -f 03-filebeat-monitored.yml  
beat.beat.k8s.elastic.co/filebeat created
clusterrole.rbac.authorization.k8s.io/filebeat created
serviceaccount/filebeat created
clusterrolebinding.rbac.authorization.k8s.io/filebeat created
```

## Deploy Metricbeat to collect monitoring data and Filebeat to collect logs

- Finally deploy Metricbeat to collect monitoring data.

```shell
> kubectl apply -f 04-monitoring-beats-eck.yml 

beat.beat.k8s.elastic.co/metricbeat-monitoring created
clusterrole.rbac.authorization.k8s.io/metricbeat configured
serviceaccount/metricbeat unchanged
clusterrolebinding.rbac.authorization.k8s.io/metricbeat unchanged
beat.beat.k8s.elastic.co/filebeat-monitoring created
clusterrole.rbac.authorization.k8s.io/filebeat unchanged
serviceaccount/filebeat unchanged
clusterrolebinding.rbac.authorization.k8s.io/filebeat unchanged
```

## Pending

- Add Auditbeat and Packetbeat, and see what security shows.
- Monitor the monitoring cluster (ES and Kibana).
- Upgrade to `7.10.`
