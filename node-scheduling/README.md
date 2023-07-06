## Outline

- Part 1 - Setting up the Kubernetes Cluster

- Part 2 - Scheduling Pods

- Part 3 - nodeName

- Part 4 - nodeSelector

- Part 5 - Node Affinity

- Part 6 - Pod Affinity

- Part 7 - Taints and Tolerations




## Part 3 - nodeName

- For this lesson, we have two instances. The one is the controlplane and the other one is the worker node. As a practice, kube-scheduler doesn't assign a pod to a controlplane. Let's see this.

- Create yaml file named `clarus-deploy.yaml` and explain its fields.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    environment: dev
spec:
  replicas: 15
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

- Create the deployment with `kubectl apply` command.

```bash
kubectl apply -f clarus-deploy.yaml
```

- List the pods and notice that all pods are assigned to the master node.

```bash
kubectl get po -o wide
```

- Delete the deployment.

```bash
kubectl delete -f clarus-deploy.yaml 
```

- For this lesson, we need two worker nodes. Instead of creating an addition node, we will use the controlplane node as both the controlplane and worker node. So, we will arrange a controlplane with the following command.

```bash
kubectl taint nodes kube-master node-role.kubernetes.io/master:NoSchedule-
kubectl taint nodes kube-master node-role.kubernetes.io/control-plane:NoSchedule-
```

- Create the clarus-deploy again.

```bash
kubectl apply -f clarus-deploy.yaml
```

- List the pods and notice that pods are assigned to both the master node and worker node.

```bash
kubectl get po -o wide
```

- Delete the deployment.

```bash
kubectl delete -f clarus-deploy.yaml 
```

- `nodeName` is the simplest form of node selection constraint, but due to its limitations, it is typically not used. nodeName is a field of PodSpec. If it is non-empty, the scheduler ignores the pod and the kubelet running on the named node tries to run the pod. Thus, if nodeName is provided in the PodSpec, it takes precedence over the other methods for node selection.

- Some of the limitations of using nodeName to select nodes are:

  - If the named node does not exist, the pod will not be run, and in some cases may be automatically deleted.
  - If the named node does not have the resources to accommodate the pod, the pod will fail and its reason will indicate why, for example, OutOfmemory or OutOfcpu.
  - Node names in cloud environments are not always predictable or stable.

- Let's try this. First list the names of nodes.

```bash
kubectl get no
```

- Update the clarus-deploy.yaml as below.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    environment: dev
spec:
  replicas: 15
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
      nodeName: kube-master  #just add this line 
```

- Create the clarus-deploy again.

```bash
kubectl apply -f clarus-deploy.yaml
```

- List the pods and notice that the pods are assigned to the only worker node.

```bash
kubectl get po -o wide
```

- Delete the deployment.

```bash
kubectl delete -f clarus-deploy.yaml 
```

## Part 4 - nodeSelector

- `nodeSelector` is the simplest recommended form of node selection constraint. nodeSelector is a field of PodSpec. It specifies a map of key-value pairs. For the pod to be eligible to run on a node, the node must have each of the indicated key-value pairs as labels (it can have additional labels as well). The most common usage is one key-value pair.

- Let's learn how to use nodeSelector.

- First, we will add a label to controlplane node with the following command. 

```bash
kubectl label nodes <node-name> <label-key>=<label-value>
```

- For example, let's assume that we have some applications that require different requirements. And we also have nodes that have different capacities. For this, we want to assign large pods to large nodes. For this, we add a label to controlplane node as below.

```bash
kubectl label nodes kube-master size=large
```

- We can check that the node now has a label with the following command. 

```bash
kubectl get nodes --show-labels
```

- We can also use `kubectl describe node "nodename"` to see the full list of labels of the given node.

- Now, we will update clarus-deploy.yaml as below. We just add `nodeSelector` field.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    environment: dev
spec:
  replicas: 15
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
      nodeSelector:         # This part is added.
        size: large
```

- Create the clarus-deploy again.

```bash
kubectl apply -f clarus-deploy.yaml
```

- List the pods and notice that the pods are assigned to the only master node.

```bash
kubectl get po -o wide
```

- Delete the deployment.

```bash
kubectl delete -f clarus-deploy.yaml 
```

## Part 5 - Node Affinity

- `Node affinity`  is conceptually similar to nodeSelector, but it greatly expands the types of constraints we can express. It allows us to constrain which nodes our pod is eligible to be scheduled on, based on labels on the node.

- There are currently three types of node affinity:

  - requiredDuringSchedulingIgnoredDuringExecution
  - preferredDuringSchedulingIgnoredDuringExecution
  - requiredDuringSchedulingRequiredDuringExecution

- Let's analyze this long sentence.

| Types                                           | DuringScheduling | DuringExecution |
| ------------------------------------------------| ---------------- | --------------- |
| requiredDuringSchedulingIgnoredDuringExecution  | required         | Ignored         |
| preferredDuringSchedulingIgnoredDuringExecution | preferred        | Ignored         |
| requiredDuringSchedulingRequiredDuringExecution | required         | required        |


- The first one (requiredDuringSchedulingIgnoredDuringExecution) specifies rules that must be met for a pod to be scheduled onto a node (similar to nodeSelector but using a more expressive syntax), while the second one (preferredDuringSchedulingIgnoredDuringExecution) specifies preferences that the scheduler will try to enforce but will not guarantee. For example, we specify labels for nodes and nodeSelectors for pods.
Later, the labels of the node are changed. If we use the first one (requiredDuringSchedulingIgnoredDuringExecution), the pod is not be scheduled. Let's see this.

- We will update clarus-deploy.yaml as below. We will add an affinity field instead of nodeSelector.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    environment: dev
spec:
  replicas: 15
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: size
                operator: In # NotIn, Exists and DoesNotExist
                values:
                - large
                - medium
```

- This node affinity rule says the pod can only be placed on a node with a label whose key is size and whose value is large or medium.

- We have already labeled the controlplane node with `size=large` key-value pair. Let's see.

```bash
kubectl get nodes --show-labels
```

- Create the clarus-deploy.

```bash
kubectl apply -f clarus-deploy.yaml
```

- List the pods and notice that the pods are running on the controlplane.

```bash
kubectl get po -o wide
```

- Delete the deployment.

```bash
kubectl delete -f clarus-deploy.yaml 
```

- Delete `size=large` label from `kube-master` node.

```bash
kubectl label node kube-master size-
```

- Create the clarus-deploy again.

```bash
kubectl apply -f clarus-deploy.yaml
```

- List the pods and notice that the pods are pending state. Because there is no node labeled with size=large.

```bash
kubectl get po -o wide
```

- Delete the deployment.

```bash
kubectl delete -f clarus-deploy.yaml 
```

- This time we will test `preferredDuringSchedulingIgnoredDuringExecution`. Update clarus-deploy.yaml as below. This time, we change `requiredDuringSchedulingIgnoredDuringExecution` with  `preferredDuringSchedulingIgnoredDuringExecution`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    environment: dev
spec:
  replicas: 15
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:   # This field is changed.
          - weight: 1                                        # This field is changed.         
            preference:                                      # This field is changed.   
              matchExpressions:
              - key: size
                operator: In
                values:
                - large
                - medium
```

- Create the clarus-deploy again.

```bash
kubectl apply -f clarus-deploy.yaml
```

- List the pods and notice that the pods are running on both the master node and worker node. Because this time it specifies preferences that the scheduler will try to enforce but will not guarantee. If there is no node labeled, the scheduler will assign randomly.

```bash
kubectl get po -o wide
```

- Delete the deployment.

```bash
kubectl delete -f clarus-deploy.yaml 
```

- The `IgnoredDuringExecution` part of the names means that similar to how nodeSelector works, if labels on a node change at runtime such that the affinity rules on a pod are no longer met, the pod continues to run on the node. In the future, the Kubernetes community plans to offer `requiredDuringSchedulingRequiredDuringExecution` which will be identical to `requiredDuringSchedulingIgnoredDuringExecution` except that it will evict pods from nodes that cease to satisfy the pods' node affinity requirements.