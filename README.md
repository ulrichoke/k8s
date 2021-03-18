# k8s
YAML examples


# My installation 
## For my sefpaced learning
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


