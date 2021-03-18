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

# POD definition
## Generate a pod YAML file
```kubectl run nginx --image=nginx --dry-run -o yaml > pod-definition.yaml```
