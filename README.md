# Lab1 NFS backed Volumes on Kubernetes
This project will walk through how to setup a simple Network File System (NFS) on a standalone server and then use that NFS share to for your persistent volume claims in Kubernetes. This was configured in a homelab on commodity hardware.

*Note this lab is not for production use.*

### Lab configuration
Below you will find the specifications for the environment used to run this lab. I am confident the lab is able to run on much less hardware or even on a set of Raspberry Pi's.

#### Virtualization Environment
| Physical Server  | Hypervisor | Physical CPU | Physical Memory | Physical Storage |
| ------------- | ------------- | ------------- | ------------- | ------------- |
| Dell R420  | ProxMox Virtual Environment 6.2-4 | 8  | 32G | 2TB RAID 5 |
| Dell R420  | ProxMox Virtual Environment 6.2-4 | 24  | 32G | 4TB RAID 5 |

#### Host Machines

| Hostname  | Operating System | vCPU | Memory | Storage |
| ------------- | ------------- | ------------- | ------------- | ------------- |
| kubernetes-controller-1  | Ubuntu 20.04  | 4  | 8G | 100G |
| kubernetes-worker-1  | Ubuntu 20.04  | 2  | 4G  | 32G |
| kubernetes-worker-2  | Ubuntu 20.04  | 2  | 4G  | 32G |
| kubernetes-worker-3  | Ubuntu 20.04  | 2  | 4G  | 32G |
| nfs-server-1  | CentOS 7 | 4  | 4G  | 250G |

### Pre-requisites
- A Kubernetes cluster running v1.19.4.
  - This lab has 3 worker nodes with 4GiB memory and 2 vCPU's.
- Access to the Kubernetes cluster.
- Kubectl already configured for use.
- A virtual machine with

## Installation

#### NFS Server
Connect to the virtual machine that will be used as the NFS server.
Once connected install the following packages.
```Shell
$ sudo yum install nfs-utils nfs-utils-lib -y
```
Start the NFS Service.
```Shell
$sudo systemctl start nfs
```

Enable the NFS server. This will make sure the service starts on boot.
```Shell
$sudo systemctl enable nfs
```
Now create a directory for where you would like your NFS to live. You may note the directory`pv0003` being created as well. This would typically be just `/nfsshare` however we are going to create a separate directory on the NFS server for the persistent volume we will be creating later on.
```Shell
$sudo mkdir -p /nfsshare/pv0003
```
If you have firewalld enabled you will need to open up some ports.
```Shell
$sudo firewall-cmd --permanent --add-service=nfs
$sudo firewall-cmd --permanent --add-service=mountd
$sudo firewall-cmd --permanent --add-service=rpc-bind
$sudo firewall-cmd --reload
```
Now that the ports are exposed we can configure the NFS server. Edit the file `/etc/exports` on the NFS server.
Add the appropriate mount point. In this example `/nfsshare` and the IP address of the servers you wish to share the mount with. This example shares it with the entire network CIDR `192.168.1.1/24`. Alternatively you may add individual IP addresses for each of your workers and controller.
```
/nfsshare 192.168.1.1/24(rw,sync,no_root_squash)
```

Restart the NFS service.
```Shell
$ sudo systemctl restart nfs.service
```

#### NFS Client
On each of the worker nodes and the controller make sure the NFS client is installed.
```Shell
$sudo apt install nfs-common
```

#### Testing The NFS Mount
You can test the NFS mount by attempting to mount it on one of the workers. This requires the worker to be in the `/etc/exports` of the NFS server. Run the mount command below replacing the IP address with the IP (or hostname if DNS configured) of the NFS server. If you used a different NFS directory than the example you will also need to change that.
```Shell
$sudo mount -t nfs 192.168.1.195:/nfsshare /mnt
$sudo df -h /mnt
```
If the NFS is configured correctly you should see something similar.
```
Filesystem               Size  Used Avail Use% Mounted on
192.168.1.195:/nfsshare   50G  1.5G   49G   3% /mnt
```

#### Namespace
Namespaces provide a way to virtualy separate different environments by still using the same physical cluster. For more information [visit the namespace documentation.](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)

We will create a namespace for our lab. I will be calling my namespace `lab1`.

There are two ways you can create your namespace. The first way is by running the create namespace command.
```shell
$kubectl create namespace lab1
```

The second way is to create a file called `hello-namespace.yml` and adding the yaml below.
```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: lab1
```

Now apply the namespace to create it.

```shell
$ kubectl apply -f hello-namespace.yml
namespace/lab1 created

$ kubectl get namespaces
NAME              STATUS   AGE
default           Active   4d21h
kube-node-lease   Active   4d21h
kube-public       Active   4d21h
kube-system       Active   4d21h
lab1              Active   10s
```

You can see that a namespace called `lab1` now exists.

#### Persistent Volume
As taken from the [Kubernetes doucmentation](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

*A PersistentVolume (PV) is a piece of storage in the cluster that has been provisioned by an administrator or dynamically provisioned using Storage Classes. It is a resource in the cluster just like a node is a cluster resource. PVs are volume plugins like Volumes, but have a lifecycle independent of any individual Pod that uses the PV. This API object captures the details of the implementation of the storage, be that NFS, iSCSI, or a cloud-provider-specific storage system.*

We will be creating a PV that uses our NFS server as backend storage. Create a `.yml` file called `hello-pv.yml`. In the persistent volume we are
setting the name to be `pv0003-nfs`. You may recall that we created a directory named `pv0003` on the NFS server earlier in the lab. We are setting the storage capacity to be `5Gi`. We have set the `storageClassName` to be `pv0003-nfs`. You will need to replace the servers IP address with your NFS servers IP address. If you created a separate path you will also need to change that.


```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003-nfs
  namespace: lab1
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: pv0003-nfs
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /nfsshare/pv0003
    server: 192.168.1.195
```

Next apply the file to create the persistent volume. You can see if it was created by running the `kubectl get pv` command. This will list all persistent volumes.

```shell
$ kubectl apply -f hello-pv.yml
persistentvolume/pv0003-nfs created
$ kubectl get pv
NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
pv0003-nfs   5Gi        RWO            Recycle          Available           pv0003-nfs              7s
```

#### Persistent Volume Claim
As taken from the [Kubernetes documentation](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

*A PersistentVolumeClaim (PVC) is a request for storage by a user. It is similar to a Pod. Pods consume node resources and PVCs consume PV resources. Pods can request specific levels of resources (CPU and Memory). Claims can request specific size and access modes (e.g., they can be mounted ReadWriteOnce, ReadOnlyMany or ReadWriteMany, see AccessModes).*

We will be creating a persistent volume claim for our newly created persistent volume. Create a file called `hello-pvc.yml`.

Here we set the name of the claim to be `pv0003-nfs` and the storageClassName to be `pv0003-nfs` which is the same as what we set in the PV.

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pv0003-nfs
  namespace: lab1
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: pv0003-nfs
  resources:
    requests:
      storage: 5Gi
```

Once created apply the persistent volume claim.

```shell
$ kubectl apply -f hello-pvc.yml
persistentvolumeclaim/pv0003-nfs created
$ kubectl get pvc -n lab1
NAME         STATUS   VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pv0003-nfs   Bound    pv0003-nfs   5Gi        RWO            pv0003-nfs     11s
````
#### Deployment
As taken from the Kubernetes documentation

*A Deployment provides declarative updates for Pods and ReplicaSets.
You describe a desired state in a Deployment, and the Deployment Controller changes the actual state to the desired state at a controlled rate. You can define Deployments to create new ReplicaSets, or to remove existing Deployments and adopt all their resources with new Deployments.*

Here we are creating a simple deployment for a `Hello World!` application. Create a file called `hello-deployment.yml`. To use the PVC that we just created we must add to the `spec` a `volumes` entry as seen below. We also added under `containers:` a field called `volumeMounts`. This specifies where in the file we would like to mount our persistent volume. The name of the volume mount is what we specified previously under `volumes:`.


```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
  namespace: lab1
  labels:
    app: hello-world
spec:
  selector:
    matchLabels:
      run: load-balancer-example
  replicas: 1
  template:
    metadata:
      labels:
        run: load-balancer-example
    spec:
      volumes:
        - name: helloworld-pv-storage
          persistentVolumeClaim:
            claimName: pv0003-nfs
      containers:
        - name: hello-world
          image: gcr.io/google-samples/node-hello:1.0
          ports:
            - containerPort: 8080
              protocol: TCP
          volumeMounts:
            - mountPath: "/mnt"
              name: helloworld-pv-storage
```

Create the deployment by running the apply command. Here we can see a deployment called `hello-world` was created. We can also see that the container has started and is actively running.

```shell
$ kubectl apply -f hello-deployment.yml
deployment.apps/hello-world created
$ kubectl -n lab1 get deployments
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
hello-world   1/1     1            1           6s
$ kubectl -n lab1 get pods -o wide
NAME                           READY   STATUS    RESTARTS   AGE   IP            NODE                  NOMINATED NODE   READINESS GATES
hello-world-699884bffc-f7fjc   1/1     Running   0          19s   10.244.3.10   kubernetes-worker-3   <none>           <none>
```

#### Validation

If you would like to validate that the NFS is mounted you may execute the command below. Here we can see that it is actively mounted by our container.

```shell
$ kubectl -n lab1 describe pvc pv0003-nfs
Name:          pv0003-nfs
Namespace:     lab1
StorageClass:  pv0003-nfs
Status:        Bound
Volume:        pv0003-nfs
Labels:        <none>
Annotations:   pv.kubernetes.io/bind-completed: yes
               pv.kubernetes.io/bound-by-controller: yes
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      5Gi
Access Modes:  RWO
VolumeMode:    Filesystem
Mounted By:    hello-world-699884bffc-f7fjc
Events:        <none>
```

We can further test this by connecting to the container and creating a file on the mounted persistent volume. In the steps below we are connecting to the container. Changing to the directory we mounted the volume `/mnt`, creating and a file. Once the file has been created we delete the pod so it no longer exists. A new pod will be created. In this example that new pod was called `hello-world-699884bffc-f5kkj`. When we connect to the new pod we can see the file persisted.

```shell
$ kubectl -n lab1 exec -it hello-world-699884bffc-f7fjc sh
$ cd /mnt
$ echo "this is a test file" > testfile.txt
$ exit
$ kubectl -n lab1 delete pod hello-world-699884bffc-f7fjc
pod "hello-world-699884bffc-f7fjc" deleted
$ kubectl -n lab1 get pods -o wide
NAME                           READY   STATUS    RESTARTS   AGE   IP           NODE                  NOMINATED NODE   READINESS GATES
hello-world-699884bffc-f5kkj   1/1     Running   0          49s   10.244.2.8   kubernetes-worker-2   <none>           <none>
$ kubectl -n lab1 exec -it hello-world-699884bffc-f5kkj sh
$ cat /mnt/testfile.txt
this is a test file
```

Furthermore if we list the files on the NFS server from a different host. We can see the file that was created.
```shell
$ ls -l /mnt/nfsshare/pv0003/
total 4
-rw-r--r--. 1 root root 20 Nov 26 14:55 testfile.txt
$ cat /mnt/nfsshare/pv0003/testfile.txt
this is a test file
```

#### Bringing It All Together
Instead of having separate files for each of the components we created. We can instead have one file that will create everything. Create a file called `full-deployment.yml` and add the contents below. You can see we separate each block of yaml with `---`.

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: lab1
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003-nfs
  namespace: lab1
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: pv0003-nfs
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /nfsshare/pv0003
    server: 192.168.1.195
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pv0003-nfs
  namespace: lab1
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: pv0003-nfs
  resources:
    requests:
      storage: 5Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
  namespace: lab1
  labels:
    app: hello-world
spec:
  selector:
    matchLabels:
      run: load-balancer-example
  replicas: 1
  template:
    metadata:
      labels:
        run: load-balancer-example
    spec:
      volumes:
        - name: helloworld-pv-storage
          persistentVolumeClaim:
            claimName: pv0003-nfs
      containers:
        - name: hello-world
          image: gcr.io/google-samples/node-hello:1.0
          ports:
            - containerPort: 8080
              protocol: TCP
          volumeMounts:
            - mountPath: "/mnt"
              name: helloworld-pv-storage
```

You can create everything by running the create command like so.

```shell
$ kubectl create -f full-deployment.yml
namespace/lab1 created
persistentvolume/pv0003-nfs created
persistentvolumeclaim/pv0003-nfs created
deployment.apps/hello-world created
```
### Uninstall
To remove everything we created you can run the delete command against the `full-deployment.yml` file.

```shell
$ kubectl delete -f full-deployment.yml
namespace "lab1" deleted
persistentvolume "pv0003-nfs" deleted
persistentvolumeclaim "pv0003-nfs" deleted
deployment.apps "hello-world" deleted
```
