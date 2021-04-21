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
Static are defined by a manifest file. They run on independant worker node and not managed through the kube-apiserver.

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
Use the following command to find the config file location in the cluster:

```
ps -aux | grep kubelet
```

Note: this type of pod when created has its name suffixed with the node name on which the file is defined.

Example: 
```
kubectl run static-pod --image=busybox --output yaml --dry-run=client --restart=Never  --command -- sleep 1000 >  /etc/kubernetes/manifests/static-pod-1.yaml
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
## Create NodePort to expose pods

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

# 5. Rollout and Versioning
## Rollout
Every first deployment will result in triggering a rollout. A new rollout create a new deployment revision.
### Rollout status
```
$ kubectl rollout status deployment webapp3 
```
### Revision and history
```
$ kubectl rollout history deployment webapp3
``` 
## Deployment Strategy
### Recreate
All pods in a deployment are bring down first and then a new deployment is performed.

***Options in a YAML format that apply:***
```
  strategy:
    type: Recreate

```


### Rolling Update (default deployment strategy in k8s)
N% of pods in a deployment will be taken down and replaced by equivalent few newer version at a time.
Related k8s commands are:

- kubectl apply -f ...
- kubectl set image ...

***Options in a YAML format that apply:***
```
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate

```
-> 25% means 1/4 of total at a time

### Rollback
This operation lets us undo revert to the older version of applications if there have been multiple deployments in the rollout history.

```
$ kubectl rollout history deployment webapp3 
deployment.apps/webapp3 
REVISION  CHANGE-CAUSE
1         <none>
2         <none>

$ kubectl rollout undo deployment webapp3 
deployment.apps/webapp3 rolled back

$ kubectl rollout history deployment webapp3 
deployment.apps/webapp3 
REVISION  CHANGE-CAUSE
2         <none>
3         <none>
```

# 5. ConfigMaps
Use of ConfigMaps is the best way of managing containers variables or environment data in a kubernetes cluster (within pod definition files).

First create ConfigMaps, a central place to manage the configuration data in the form of key-value pairs. Then reference variables in the pod files.

## Imperatively
```
kubectl create configmap configmap-def-1 \
--from-literal=APP1_VAR1="myapp data" \
--from-literal=APP2_VAR2="myapp data2" \
--dry-run -o yaml > configmap-definition.yml
```
***Or***
```
kubectl create configmap configmap-def-1 \
--from-file=app_config_file \
--dry-run -o yaml > configmap-definition.yml
```

## Declaratively
```
apiVersion: v1
kind: ConfigMap
data:
  APP1_VAR1: myapp data
  APP2_VAR2: myapp data2
metadata:
  creationTimestamp: null
  name: configmap-def-1
```

# 6. Secrets
Same logic as configMap to define and store secrets.

## Declaratively
```
apiVersion: v1
data:
  APP1_DB_USER_VAR: bmV3X3VzZXI=
  APP1_DB_PASSWD_VAR: cGE1NXcwcmQ=
kind: Secret
metadata:
  name: secret-def-1
```

***Encode secrets:***
```
echo -n "pa55w0rd" | base64
```
***Decode secrets:***
```
echo -n "cGE1NXcwcmQ=" | base64 --decode
```

# 7. Storage
## Storage drivers
They help manage volumes on images and containers.
Examples: AUFS | ZFS | BTRS | DEVICE MAPPER | OVERLAY

## Volumes drivers
Volumes are handled by volumes drivers plugins.
The default volumes driver is local and help create volumes on docker hosts (``` /var/lib/docker/volumes ```).

Third party solutions are: DigitalOcean Block Storage | Azure File Storage | VMware vSphere | gce-docker | Convoy | GlusterFS | NetApp | RexRay | Portworx ...

## CRI
Container Runtime Interface is a standard that define how an orchestration solutions such as kubernetes will communicate with container runtimes like docker. 

## CSI 
Container Storage Interface was developed to support multiple storage solutions (AWS EBS, GlusterFS, ...) 

## Setup NFS client provisioner for PV
To easy create NFS Persistent Volume on users behalf we make use of nfs-client-provisioner.
The NFS client provisioner is an automatic provisioner for Kubernetes that uses your already configured NFS server, automatically creating Persistent Volumes.
[Read more](https://artifacthub.io/packages/helm/ckotzbauer/nfs-client-provisioner).

An important step to consider bebore proceeding to setup:
- Install helm on the cluster master if it is not yet done
- Install nfs-common on kubelet (all cluster nodes)
- Also in order to support nfs3, we would have to enable statd by starting ```rpcbind service```
followed by ```nfs-common service``` restart.

Now install nfs-client-provisioner using helm chart:
```
$ helm install --set nfs.server=x.x.x.x --set nfs.path=/exported/path_on_remote_server stable/nfs-client-provisioner
```
To have provisioner deployed on other namespaces aimply edit by:

```
kubectl edit deployments.apps nfs-client-provisioner --output yaml > nsf-client-provisionner.yaml
```

And then change the namespace for a new deployment

## Reference link 

[How to Setup Dynamic NFS Provisioning Server For Kubernetes](https://redblink.com/setup-nfs-server-provisioner-kubernetes/).

# 8. Certificates
## Generate keys and certificate signing request
```
openssl genrsa -out ca.key 2048
openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr
```

## Self sign certificates
```
openssl  x509 -req -in ca.csr -signkey ca.key -out ca.crt
```
## Kube admin user certificate
Create first the admin user key and generate certificate signing request. Then sign with the Kubernetes CA certificate.
```
$ openssl genrsa -out admin-key.key 2048
$ openssl req -new -key admin -subj "/CN=kube-admin" -out admin-key.csr
$ openssl x509 -req -in admin-key.csr -CA ca.crt -CAkey ca.key -out admin.crt
```
## View certificate
```
$ openssl x509 -in admin.crt -text -noout
```

## Certificate approval using certificateSigningRequest object
### 1. Export CSR encoded to environment variable
```
export admin_csr_base64=$(cat admin-key.csr | base64 | tr -d '\n')
```
### 2. Create CSR kubernetes object
 mycsr.yaml 
```
---
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: myuser
spec:
  groups:
  - system:authenticated
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - server auth
  - digital signature
  - key encipherment
  request: ${admin_csr_base64}

```
Depending on k8s installation you may need to use this defintion instead:

```
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: myuser
spec:
  groups:
  - system:authenticated
  usages:
  - server auth
  - digital signature
  - key encipherment
  request: ${admin_csr_base64}
```

```
cat mycsr.yaml | envsubst | kubectl apply -f -
```
### 3. Sign the certificate
```
kubectl certificate approve myuser
```

### 4. Get the certificate

```
kubectl get csr myuser -o jsonpath='{.status.certificate}'| base64 -d > myuser.crt
```
