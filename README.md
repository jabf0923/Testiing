# CheatSheet Kubectl

## Commands

Getting lists of pods and nodes

1. I guess you are all aware of how to get a list of pods across all Kubernetes namespaces using the --all-namespaces flag. Many people are so used to it that they have not noticed the emergence of its shorter version, -A (it exists since at least Kubernetes 1.15).

2. How do you find all non-running pods (i.e., with a state other than Running)?

~~~
kubectl get pods -A --field-selector=status.phase!=Running | grep -v Complete
~~~

By the way, examining the --field-selector flag more closely (see the relevant documentation) might be a good general recommendation.

3.Here is how you can get the list of nodes and their memory size:

```bash
kubectl get no -o json | \
  jq -r '.items | sort_by(.status.capacity.memory)[]|[.metadata.name,.status.capacity.memory]| @tsv'
  ```

4.Getting the list of nodes and the number of pods running on them:

```bash
kubectl get po -o json --all-namespaces | \
  jq '.items | group_by(.spec.nodeName) | map({"nodeName": .[0].spec.nodeName, "count": length}) | sort_by(.count)'
  ````

5.Sometimes, DaemonSet does not schedule a pod on a node for whatever reason. Manually searching for them is a tedious task, so here is a mini-script to get a list of such nodes:

```bash
ns=my-namespace
pod_template=my-pod
kubectl get node | grep -v \"$(kubectl -n ${ns} get pod --all-namespaces -o wide | fgrep ${pod_template} | awk '{print $8}' | xargs -n 1 echo -n "\|" | sed 's/[[:space:]]*//g')\"
```

6.This is how you can use kubectl top to get a list of pods that eat up CPU and memory resources:

```bash
kubectl top pods -A | sort --reverse --key 3 --numeric
```

```bash
kubectl top pods -A | sort --reverse --key 4 --numeric
```

7.Sorting the list of pods (in this case, by the number of restarts):

```bash
kubectl get pods --sort-by=.status.containerStatuses[0].restartCount
```

Of course, you can sort them by other fields, too (see PodStatus and ContainerStatus for details).
Getting lists of pods and nodes


1. When tuning the Ingress resource, we inevitably go down to the service itself and then search for pods based on its selector. I used to look for this selector in the service manifest, but later switched to the -o wide flag:

```bash
kubectl -n jaeger get svc -o wide
```
```bash
NAME                            TYPE        CLUSTER-IP        EXTERNAL-IP   PORT(S)                                  AGE   SELECTOR
jaeger-cassandra                ClusterIP   None              none        9042/TCP                                 77d   app=cassandracluster,cassandracluster=jaeger-cassandra,cluster=jaeger-cassandra
```

As you can see, in this case, we get the selector used by our service to find the appropriate pods.

2. Here is how you can easily print limits and requests of each pod:

```bash
kubectl get pods -n my-namespace -o=custom-columns='NAME:spec.containers[*].name,MEMREQ:spec.containers[*].resources.requests.memory,MEMLIM:spec.containers[*].resources.limits.memory,CPUREQ:spec.containers[*].resources.requests.cpu,CPULIM:spec.containers[*].resources.limits.cpu'
```

3. The kubectl run command (as well as create, apply, patch) has a great feature that allows you to see the expected changes without actually applying them — the --dry-run flag. When it is used with -o yaml, this command outputs the manifest of the required object. For example:

```bash
kubectl run test --image=grafana/grafana --dry-run -o yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    run: test
  name: test
spec:
  replicas: 1
  selector:
    matchLabels:
      run: test
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        run: test
    spec:
      containers:
      - image: grafana/grafana
        name: test
        resources: {}
status: {}
```

All you have to do now is to save it to a file, delete a couple of system/unnecessary fields, et voila.

NB: Please note that the kubectl run behavior has been changed in Kubernetes v1.18 (now, it generates Pods instead of Deployments). 

4.Getting a description of the manifest of a given resource:

```bash
kubectl explain hpa
```

```yaml
KIND:     HorizontalPodAutoscaler
VERSION:  autoscaling/v1
DESCRIPTION:
     configuration of a horizontal pod autoscaler.
FIELDS:
   apiVersion string
     APIVersion defines the versioned schema of this representation of an
     object. Servers should convert recognized schemas to the latest internal
     value, and may reject unrecognized values. More info:
     <https://git.k8s.io/community/contributors/devel/api-conventions.md#resources>
kind    string
     Kind is a string value representing the REST resource this object
     represents. Servers may infer this from the endpoint the client submits
     requests to. Cannot be updated. In CamelCase. More info:
     <https://git.k8s.io/community/contributors/devel/api-conventions.md#types-kinds>
metadata    Object
     Standard object metadata. More info:
     <https://git.k8s.io/community/contributors/devel/api-conventions.md#metadata>
spec    Object
     behaviour of autoscaler. More info:
     <https://git.k8s.io/community/contributors/devel/api-conventions.md#spec-and-status>.
status    Object
     current information about the autoscaler.
```

Well, that is a piece of extensive and very helpful information, I must say.

Networking

1.Here is how you can get internal IP addresses of cluster nodes:

```bash
kubectl get nodes -o json | \
  jq -r '.items[].status.addresses[]? | select (.type == "InternalIP") | .address' | \
  paste -sd "\n" -
```

2.And this way, you can print all services and their respective nodePorts:

```bash
kubectl get --all-namespaces svc -o json | \
  jq -r '.items[] | [.metadata.name,([.spec.ports[].nodePort | tostring ] | join("|"))]| @tsv'
```

3.In situations where there are problems with the CNI (for example, with Flannel), you have to check the routes to identify the problem pod. Pod subnets that are used in the cluster can be very helpful in this task:

```bash
kubectl get nodes -o jsonpath='{.items[*].spec.podCIDR}' | tr " " "\n"
```

Logs

1. Print logs with a human-readable timestamp (if it is not set):

```bash
kubectl -n my-namespace logs -f my-pod --timestamps
```

```bash
2020-07-08T14:01:59.581788788Z fail: Microsoft.EntityFrameworkCore.Query[10100]
```

Logs look so much better now, don’t they?

2. You do not have to wait until the entire log of the pod’s container is printed out — just use --tail:

```bash
kubectl -n my-namespace logs -f my-pod --tail=50
```

3. Here is how you can print all the logs from all containers of a pod:

```bash
kubectl -n my-namespace logs -f my-pod --all-containers
```

4. Getting logs from all pods using a label to filter:

```bash
kubectl -n my-namespace logs -f -l app=nginx
```

5. Getting logs of the “previous” container (for example, if it has crashed):

```bash
kubectl -n my-namespace logs my-pod --previous
```

Other quick actions

1. Here is how you can quickly copy secrets from one namespace to another:

```bash
kubectl get secrets -o json --namespace namespace-old | \
  jq '.items[].metadata.namespace = "namespace-new"' | \
  kubectl create-f  -
```

2. Run these two commands to create a self-signed certificate for testing:

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=grafana.mysite.ru/O=MyOrganization"
kubectl -n myapp create secret tls selfsecret --key tls.key --cert tls.crt.
```
