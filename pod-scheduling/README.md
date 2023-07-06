Pod Affinity

- Pod Affinity allow you to constrain which nodes your Pods can be scheduled on based on the labels of Pods already running on that node, instead of the node labels.

- We have a DB pod and frontend deployment. We want to schedule DB pods and frontend pods on the same node.

- create yaml file and named `clarus-db.yaml`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: clarus-db
  labels:
    tier: db
spec:
  containers:
  - name: clarus-db
    image: clarusway/clarusdb
    imagePullPolicy: IfNotPresent
  restartPolicy: Always
```

- Create the pod with `kubectl apply` command.

```bash
kubectl apply -f clarus-db.yaml
```

- List the pods.

```bash
kubectl get po -o wide
```

-  Create yaml file named `clarusshop-deploy.yaml`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: clarusshop
  labels:
    app: clarusshop
spec:
  replicas: 5
  selector:
    matchLabels:
      app: clarusshop
  template:
    metadata:
      labels:
        app: clarusshop
    spec:
      containers:
      - name: clarusshop
        image: clarusway/clarusshop
        ports:
        - containerPort: 80
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: tier
                operator: In
                values:
                - db
            topologyKey: "kubernetes.io/hostname"
```

- Create the deployment with `kubectl apply` command.

```bash
kubectl apply -f clarusshop-deploy.yaml
```

- List the pods and notice that the clarusshop pods are assigned to the same nod with clarus-db pod.

```bash
kubectl get po -o wide
```

- Delete the deployment and pods.

```bash
kubectl delete -f clarusshop-deploy.yaml
kubectl delete -f clarus-db.yaml
```
