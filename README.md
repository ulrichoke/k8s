# k8s
YAML examples



# My installation 
## For my self paced learning environment
- Host OS: Mac Big Sur
- K8s installation tool: hyperctl 

hyperkit is a Multi-node (or single-node) Kubernetes on CentOS/Ubuntu in Hyper-V/Hyperkit. It's also made of Docker on Desktop without Docker for Desktop.

The lab is composed of 2 worker nodes and one master node. 

```
cd hyperctl_directory
./hyperctl.sh master node1 node2
ssh master
```
To start and stop the lab

```
./hyperctl start
./hyperctl info
./hyperctl stop
```

## Setup bash-completion 
```
sudo yum install bash-completion
source /usr/share/bash-completion/bash_completion
```

 Source the completion script in your ~/.bashrc file:

```
echo 'source <(kubectl completion bash)' >>~/.bashrc
```

If you have an alias for kubectl, you can extend shell completion to work with that alias:

```
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -F __start_kubectl k' >>~/.bashrc
```

# 1. POD definition
## Generate a pod YAML file
```kubectl run nginx --image=nginx --dry-run -o yaml > pod-definition.yaml```

## Static POD
Static are defined by a manifest file. They run on independant worker node or not managed through the kube-apiserver.

Two ways for its definition:
- In the standard manifests folder:
Create pod YAML file inside  ```/etc/kubernetes/manifests ```

- In a custom path as a kubelet.service configuration option: 
```
Exec-Start=/usr/bin/local/kubelet \\
....
  --config = kubeconfig.yaml \\
....
```
kubeconfig.yaml
```
staticPodPath: /etc/kubernetes/manifests
```


# 2. DEPLOYMENT definition
## Generate a deployment YAML file 
```
kubectl create deployment mydeployment --image=nginx --dry-run=true -o yaml > deployment-definition-1.yml
```

# 3. Imperative commands
## Create deployment
```
$ kubectl create deploy webapp --image=nginx:alpine --replicas=3
```
## Create NodePort service to access deployment

- deployment: webapp-deploy
- pod: webapp-svc
- type: NodePort
- targetPort (cluster port): 8080
- port (deployment port): 8080
- nodePort: 30080
- selector: front-end 
```
$ kubectl expose deployment webapp --name=webapp-svc --type=NodePort --target-port=8080 --port=8080 --dry-run -o yaml > svc.yml
$ vi svc.yml
---
apiVersion: v1
kind: Service
metadata:
  name: webapp-svc
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
    nodePort: 30080
  selector:
    type: front-end
  type: NodePort
status:
  loadBalancer: {}
---

$ kubectl create -f svc.yml

```

## Create ClusterIP service
- targetPort and port: 6379
```
$ kubectl create service clusterip redis-service --tcp=6379:6379 --dry-run=client -o yaml > clusterip-svc.yml
$ kubectl create -f clusterip-svc.yml
```


## Create ClusterIP service to expose redis pod on cluster port 6379

```
$ kubectl expose pod --name=redis-service redis --port 6379
```

## Create a new pod called my-nginx with nginx image and expose it on container port 8080
```
$ kubectl run my-nginx --image=nginx --port=8080
```
## Create namespace dev
```
$ kubectl create ns dev
```

## Create a new deployment called redis-deploy

- namespace= dev 
- image= redis
- 2 replicas
```
$ kubectl create deployment redis-deploy --image=redis --namespace=dev 
$ kubectl scale deployment redis-deploy --replicas=2
```

## Create a pod and service in a few steps

- image: nginx:alpine 
- service type: ClusterIP
- target port: 80

```
kubectl run httpd --image=nginx:alpine --expose --port 80
```


# 4. Scheduling

## Command tips

To check the options that could be used for tolerations for e.g:

```
$ kubectl explain pod --recursive | grep -A5 tolerations
```

To remove a node label:

```
$ kubectl label node nodename labelkey-
```


## Set a taint on a node
```
$ kubectl taint node mynode-name key=value:taint-effect
``` 
**taint-effect** possible value: NoSchedule | PreferNoSchedule | NoExecute

- ***NoSchedule:*** Pods that do not tolerate this taint are not scheduled on the node.
- ***PreferNoSchedule:*** Kubernetes avoids scheduling Pods that do not tolerate this taint onto the node.
- ***NoExecute:*** Pod is evicted from the node if it is already running on the node, and is not scheduled onto the node if it is not yet running on the node.

## Remove taint from a node
e.g.
```
$ kubectl taint node node1 deployment=green:PreferNoSchedule-
```

