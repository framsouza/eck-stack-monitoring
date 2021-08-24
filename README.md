# eck-stack-monitoring

If you are planning to run ECK in production, you are on the right page.
At the ECK version 1.7.0 we simplified how you can monitor your ECK cluster. Instead of deploy beats, you can use monitoring CDR on the production manifest, you just need to specify the name of the monitoring cluster where it should ship the data _(do not do self-monitoring in production)_
With that, is quite simple to set up a central monitoring cluster to handle your production environment metrics.
As a matter of organization, we are going to use a namespace called monitoring, which will contain the Kibana & Elasticsearch monitoring data, quite simple:

```
kind: Elasticsearch
metadata:
  name: monitoring
  namespace: monitoring
spec:
  version: 7.14.0
  nodeSets:
  - name: es
    count: 3
    config:
      node.store.allow_mmap: false
---
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: monitoring
  namespace: monitoring
spec:
  version: 7.14.0
  count: 1
  elasticsearchRef:
    name: monitoring
```

After some minutes, you should have something like this:

```
kubectl get pods -n monitoring
NAME                             READY   STATUS    RESTARTS   AGE
monitoring-es-es-0               1/1     Running   0          18m
monitoring-es-es-1               1/1     Running   0          18m
monitoring-es-es-2               1/1     Running   0          18m
monitoring-kb-585f845cfc-9sxp4   1/1     Running   0          17m
```
If you forward the Kibana port (kubectl port-forward -n monitoring  svc/monitoring-kb-http 5601) and grab the elastic password (kubectl get secret quickstart-es-elastic-user -o yaml) you can access the Stack Monitoring, you won't see anything there yet because we still didn't deploy our production environment.
Now, let's deploy the production environment with the following manifest:

```
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: quickstart
spec:
  version: 7.14.0
  monitoring:
    metrics:
      elasticsearchRefs:
        - name: monitoring
          namespace: monitoring
    logs:
      elasticsearchRefs:
        - name: monitoring
          namespace: monitoring
  nodeSets:
  - name: es
    count: 3
    config:
      node.store.allow_mmap: false
---
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: quickstart
spec:
  version: 7.14.0
  count: 1
  elasticsearchRef:
    name: quickstart
  config:
  monitoring:
    metrics:
      elasticsearchRefs:
        - name: monitoring
          namespace: monitoring
    logs:
      elasticsearchRefs:
        - name: monitoring
          namespace: monitoring
```
No beats are needed, we just need to use the monitoring CRD to specify the monitoring cluster name.
If you now access the Stack Monitoring at your ECK monitoring cluster, you should be able to see ECK metrics from the production cluster.

I hope it helps,
Cheers,
