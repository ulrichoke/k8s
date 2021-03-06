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
$ echo 'source <(kubectl completion bash)' >>~/.bashrc
```

If you have an alias for kubectl, you can extend shell completion to work with that alias:

```
$ echo 'alias k=kubectl' >>~/.bashrc
$ echo 'complete -F __start_kubectl k' >>~/.bashrc
```

# 1. POD definition
## Generate a pod YAML file
```$ kubectl run nginx --image=nginx --dry-run -o yaml > pod-definition.yaml```

## Static POD
Static are defined by a manifest file. They run on independant worker node and not managed through the kube-apiserver.

Two ways for its definition:
- In the standard manifests folder:
Create pod YAML file inside  ```/etc/kubernetes/manifests ```

- In a custom path as defined through kubelet.service configuration options: 
```
Exec-Start=/usr/bin/local/kubelet \\
....
  --config = kubeconfig.yaml \\
....
```

Look at kubelet configuration file (kubeconfig.yaml _kubernetes the hard way setup case_) to find manifest full path:
```
staticPodPath: /etc/kubernetes/manifests
```
or get it through API

```
$ kubectl proxy
$ curl "http://localhost:8001/api/v1/nodes/node1/proxy/configz" | python -m json.tool | grep PodPath
```

Use the following command to find the config file location in the cluster:

```
ps -aux | grep kubelet
```

Note: this type of pod when created has its name suffixed with the node name on which the file is defined.

Example: 
```
$ kubectl run static-pod --image=busybox --output yaml --dry-run=client --restart=Never  --command -- sleep 1000 >  /etc/kubernetes/manifests/static-pod-1.yaml
```

# 2. DEPLOYMENT definition
## Generate a deployment YAML file 
```
$ kubectl create deployment mydeployment --image=nginx --dry-run=true -o yaml > deployment-definition-1.yml
```

## Create deployment with Imperative command
```
$ kubectl create deploy webapp --image=nginx:alpine --replicas=3
```
# 3. Expose pods with imperative command _(Service type NodePort)_

Example:
- deployment : webapp-deploy
- pod: webapp-deploy
- type: NodePort
- targetPort (cluster port): 8080
- port (deployment port): 8080
- nodePort: 30080
- selector: front-end 
```
$ kubectl expose deployment webapp-deploy --name=webapp-svc --type=NodePort --target-port=8080 --port=8080 --dry-run -o yaml > svc.yml
$ cat svc.yml
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

# 4. Create ClusterIP service _(Service type ClusterIP)_

_kubelet_ handle networking through kube-proxy (daemonset) component. ClusterIP service is a cluster wide concept and a virtual object that doesn't realy exist like a service.

- targetPort and port: 6379
```
$ kubectl create service clusterip redis-service --tcp=6379:6379 --dry-run=client -o yaml > clusterip-svc.yml
$ kubectl create -f clusterip-svc.yml
```

## Set clusterIP ip range in service configuration (default 10.0.0.0/24)


Edit the kube-controller-manager-master pod (kubeadmn deployment) or service configuration file (from scrash).

```
$ kubectl -n kube-system edit pod kube-controller-manager-master
```

_Custom clusterIP range configuration_ 

```
kube-apiserver --service-cluster-ip-range=[ your_custom_ip_cidr ] ...
```

_Custom NodePort range configuration_ 

```
kube-apiserver --service-node-port-range=[ your-node-port-range ] ...
```

> It can also be defined at kubeadmn initialisation 


## To check the clusterip range:

```
$ ps aux | grep cluster-ip
```

## To look for proxy mode:

```
kubectl logs -n kube-system kube-proxy-xxxxx -c kube-proxy | grep mode
```


# 5. Create ClusterIP service to expose redis pod on cluster port 6379 

```
$ kubectl expose pod --name=redis-service redis --port 6379
```


# 6. Create a new pod _my-nginx_ with _nginx_ image and expose it on container port 8080
```
$ kubectl run my-nginx --image=nginx --port=8080
```
# 7. Create namespace dev 
```
$ kubectl create ns dev
```

To access a pod in a specific namespace 

```
[ xx-xx-xx-xx ].dev.pod.cluster.local
```
XX -> ip octets.

To access a service cluster in a specific namespace 

```
[ my-service-name ].dev.svc.cluster.local
```

# 8. Create a new deployment _redis-deploy_

- namespace= dev 
- image= redis
- 2 replicas
```
$ kubectl create deployment redis-deploy --image=redis --namespace=dev 
$ kubectl scale deployment redis-deploy --replicas=2
```

# 9. Create a pod and service in a few steps

- image: nginx:alpine 
- service type: ClusterIP
- target port: 80

```
$ kubectl run httpd --image=nginx:alpine --expose --port 80
```


# 10. Scheduling

## Using label selectors 

Create node labels
```
kubectl label nodes node2 disk=hdd
kubectl describe nodes node2 | grep -A6 Labels
```

Generate pod defintion file
```
kubectl run --image=nginx nginx --generator=run-pod/v1  --dry-run -o yaml > nginx.yaml
```

Edit and set _nodeSelector_ in _spec_  to schedule pod with definition file:
```
spec:
  ...
    nodeSelector:
        disk: hdd

  ...
```

## Command tips

To check the options that could be used for tolerations for e.g:

```
$ kubectl explain pod --recursive | grep -A5 tolerations
```


To remove a node label:

```
$ kubectl label node nodename labelkey-
```

To set deployment ressources

```
kubectl set resources deployment nginx --requests=cpu=0.1,memory=1Gi 

```


## Set a taint on a node
```
$ kubectl taint node mynode-name key=value:[ taint-effect ]
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

# 11. Rollout and Versioning
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

# 12. ConfigMaps
Use of ConfigMaps is the best way of managing containers variables or environment data in a kubernetes cluster (within pod definition files).

First create ConfigMaps, a central place to manage the configuration data in the form of key-value pairs. Then reference variables in the pod files.

## Imperatively
```
$ kubectl create configmap configmap-def-1 \
   --from-literal=APP1_VAR1="myapp data" \
   --from-literal=APP2_VAR2="myapp data2" \
   --dry-run -o yaml > configmap-definition.yml
```
***Or***
```
$ kubectl create configmap configmap-def-1 \
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

# 13. Secrets
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

# 14. Storage
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
To easily create NFS Persistent Volume on users behalf we make use of nfs-client-provisioner.
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
$ kubectl edit deployments.apps nfs-client-provisioner --output yaml > nsf-client-provisionner.yaml
```

And then change the namespace for a new deployment

## Reference link 

[How to Setup Dynamic NFS Provisioning Server For Kubernetes](https://redblink.com/setup-nfs-server-provisioner-kubernetes/).

# 15. Security
## a. Generate keys and certificate signing request
```
$ openssl genrsa -out ca.key 2048
$ openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr
```

## b. Self sign certificates
```
$ openssl  x509 -req -in ca.csr -signkey ca.key -out ca.crt
```
## c. Kube admin user certificate
Create first the admin user key and generate certificate signing request. Then sign with the Kubernetes CA certificate.
```
$ openssl genrsa -out admin-key.key 2048
$ openssl req -new -key admin -subj "/CN=kube-admin" -out admin-key.csr
$ openssl x509 -req -in admin-key.csr -CA ca.crt -CAkey ca.key -out admin.crt
```
 View certificate
```
$ openssl x509 -in admin.crt -text -noout
```
 Test user certificate
```
$ curl https://my-kube-server:6443/api/v1/pods \
     --key admin.key  \
     --cert admin.crt \
     --cacert ca.crt
```
or
```
$ kubectl get pods \
    --server my-kube-playground:6443 \
    --client-key admin.key \
    --client-certificate admin.crt \
    --certificate-authority ca.crt 
```

## d. Certificate approval using certificateSigningRequest object
### 15.d.1. Export CSR encoded to environment variable
```
$ export admin_csr_base64=$(cat admin-key.csr | base64 | tr -d '\n')
```
### 15.d.2. Create CSR kubernetes object
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
### 15.d.3. Sign the certificate
```
kubectl certificate approve myuser
```

### 15.d.4. Get the certificate

```
$ kubectl get csr myuser -o jsonpath='{.status.certificate}'| base64 -d > myuser.crt
```

## e. Organize cluster access with Kubeconfig

Use kubeconfig files to organize information about clusters, users, namespaces, and authentication mechanisms. The kubectl command-line tool uses kubeconfig files to find the information it needs to choose a cluster and communicate with the API server of a cluster

### 15.e.1. Define clusters, users, and contexts
Suppose you have two clusters, one for development work and one for scratch work. In the development cluster, your frontend developers work in a namespace called frontend, and your storage developers work in a namespace called storage. In your scratch cluster, developers work in the default namespace, or they create auxiliary namespaces as they see fit. Access to the development cluster requires authentication by certificate. Access to the scratch cluster requires authentication by username and password.

(cf. security/kubeconfig)

### 15.e.2. Get kubeconfig
```
$ kubectl config --kubeconfig=security/my-kubeconfig view
```

### 15.e.3. Change the current context 
```
$ kubectl config --kubeconfig=security/my-kubeconfig use-context exp-scratch
```

## f. Authorization mechanisms
### 15.f.1. Node
Any request comming from user with the name ```system:node:node-name``` and part of the system node group will be handled by the node authorizer.
### 15.f.2. ABAC
Attribute Based Access Control is an external authorization where a user or a group of users are associated with a set of permissions (view, create, delete pods ...). So you need to create a **policy definition** file for each set of permissions.
These files are meant to be update manually for any permission change.

### 15.f.3. RBAC
Instead of associating permission to a user RBAC help to easy the management by creating role with a set of permissions.
e.g. develper role with view, create and delete pods permissions in a specific namespace. In this case change are immediately reflected to the user or set of users associated. 
```
$ kubectl apply -f security/developer-role.yaml
$ kubectl apply -f security/developer-binding.yaml
$ kubectl get role binding developer
$ kubectl get rolebinding devuser-developer
$ kubectl describe role developer
$ kubectl describe rolebinding devuser-developer
```

Imperative commands:

```
$ kubectl create role developer --verb=get --verb=list --verb=watch --resource=pods --namespace=blue -o yaml --dry-run > security/developer-role.yaml

$ kubectl create rolebinding devuser-developer --role=developer --user=dev-user --namespace=blue --output yaml --dry-run > security/developer-binding.yaml
```

To check access
```
$ kubectl auth can-i create pods
$ kubectl auth can-i delete pods
$ kubectl auth can-i create pods --as dev-user

```

**Note :**
_Role and RoleBinding kubernetes objects are limited to namespaces. ClusterRole are just like role except they are cluster scope resources._

```
$ kubectl create clusterrole cluster-administrator --verb=get --verb=list --verb=delete --verb=create --resource=nodes -o yaml --dry-run > security/admin-clusterrole.yaml
$ kubectl create clusterrole storage-administrator --verb=get --verb=list --verb=delete --verb=create --resource=pv,pvc -o yaml --dry-run > security/storage-clusterrole.yaml

$ kubectl create clusterrolebinding adminuser-clusteradmin --clusterrole=cluster-administrator --user=admin-user --output yaml --dry-run > security/admin-clusterrolebinding.yaml
$ kubectl create clusterrolebinding storage-clusteradmin --clusterrole=cluster-administrator --user=storage-administrator --output yaml --dry-run > security/storage-clusterrolebinding.yaml
```

To list all available cluster scope resources:
```
$ kubectl api-resources --namespaced=false
```

### 15.f.4. Webhook
Webhook is an external authorization mechanism outside of kubernetes cluster. Third party tool such as Open Policy Agent can help with admission control and authorization. So you can have kubernetes making API call to the Open Policy Agent with the information about the user and his access requirements. The OPA will then decide if the user should be permited or not.
### AlwaysAllow / AlwaysDeny
These authorization modes allows or deny all requests without performing any authorization checks.
To set authorization mode in kubernetes service config file:
```
ExecStart=/usr/local/bin/kube-apiserver \\
  ...
  --authorization-mode=Node,RBAC,Webhook
  ...
```
### 15.f.5. Image security
Implementing docker registry authentication
``` 
$ kubectl create secret docker-registry regcred \
   --docker-server=private-registry.io \
   --docker-username=registry-user \
   --docker-password=registry-pass \
   --docker-email=registry-user@domain.org   
```

Use ```??magePullSecret``` at spec level to apply the secret.

```
...
imagePullSecrets: 
  - name: registry-secrets
...
```
Then create the pods with the resulting definition file.

```
$ kubectl apply -f security/private-registry.yaml
```

```docker-registry``` is a builting secret type for docker credentials storing.

### 15.f.6. Security context
Docker security options can be set through k8s pods or deployments at pod or containers level. 

**Capabilities are only supported at containers level**

```
...
spec
  securityContext:
     runAsUser: 1000
...
```

### 15.f.7. Network Policy
Use NetworkPolicy objet and link it to the pods with ```podSelector``` to set incoming or outgoing traffic limitations (ingress/egress).
At the time of this writing the compatible network solutions are:
- Kube-router
- Calico
- Romana
- Weave-net

# 16. CNI
## a. Container Network Interface plugin

Where are located cni plugins (default ```/opt/cni/bin```)
```
$ ps aux | grep kubelet | grep network-plugin
```
Find the cni configuration dir (default ```/etc/cni/net.d/```)
```
$ ps aux | grep kubelet | grep cni-conf-dir
```

## b. Deploy Weave plugin

```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
kubectl logs weave-net-xxxxx weave ???n kube-system
```

?????? _Tips_

To check the weave ip allocation range:
```
kubectl logs weave-net-xxxxx weave ???n kube-system | grep ipalloc-range
```

# 17. DNS

Name resolution is base on mapping service name and its IP address.
By default the cluster root domain is ```cluster.local``` 
[ Hostname ].[ Namespace ].[ Type ].[ root domain ]

e.g: 

db-service.dev.svc.cluster.local

DNS records for pods are not explicitily enabled by default.

## DNS service in K8s
- Old k8s installation: kube-dns
- k8s > 1.12 : coreDNS

CoreDNS configuration file located at ``` /etc/coredns/Corefile ``` (by default) contains plugin directives:


e.g.:
```
.:53 {
  errors
  health
  kubernetes cluster.local in-addr.arpa ip6.arpa {
    pods insecure
    upstream
    fallthrough in-addr.arpa ip6.arpa
    
  }
  prometheus :9153
  proxy . /etc/resolv.conf
  cache 30
  reload
  
}
```

``` kubernetes ``` : where the core domain name is setup (default cluster.local)
``` pods insecure ``` : enable pod name records

- Get corefile location
```
kubectl -n kube-system describe deployments.apps coredns | grep -A2 Args | grep Corefile
```

Refer to _coredns and kubelet-config-1.XX_ configmap objects to make change to the coredns component.

- Name of the service created for accessing CoreDNS (default is kube-dns):
```
$ kubectl get service -n kube-system
```
# 18. INGRESS

Ingress help to access k8s applications through a single externaly accessible url and route to different services within the cluster base on url path alongside with the security setup.

INGRESS = single layer 7 loadbalancer builtin to the k8s cluster

## Setup process:
- Deploy: selecting an ingress controller (Nginx proxy, HAProxy, traefik, lstio, GCP HTTP LB, ...)
- Configure: configure ingress resources (url route and rules, ssl certificate, ...)

The ingress controllers have additional intelligence to monitor cluster and configure the nginx server when something is changed (like domain name, rules, ...). This will require a ___ServiceAccount___ objet to do so. The configuration parameters (timeout, keep-alive, err-log-path, ssl-protocols, ...)  are set by using a ___ConfigMap___ object.

## k8s object used:
- Deployment definition
- Service definition to link to the deployment
- ConfigMap for configuration
- ServiceAccount to set right permissions and authentication + Roles, ClusterRole and RoleBinding.

## Ingress resource:
### @ Rules:
- Route traffic based on domain name
- Route traffic based on url path
- Route all traffic to a specific pod 
