# Setting Up Kubernetes Deployment with KinD and Podman

# Table of Contents
1. [Introduction](#introduction)
2. [Prerequisites](#prerequisites)
3. [Steps](#steps)
    - [Ensure the Host is Running with Control Group v2](#ensure-the-host-is-running-with-control-group-v2)
    - [Enabling CPU, CPUSET, and I/O delegation](#enabling-cpu-cpuset-and-io-delegation)
    - [Installing the Kubernetes Command Line Tool (kubectl)](#installing-the-kubernetes-command-line-tool-kubectl)
    - [Creating a kind cluster with Rootless Podman](#creating-a-kind-cluster-with-rootless-podman)
    - [(Optional) Verify the Kubernetes Environment](#optional-verify-the-kubernetes-environment)
4. [Troubleshooting](#troubleshooting)
5. [Conclusion](#conclusion)

## Introduction
This guide will walk you through the steps to set up KinP, which stands for Kubernetes in Podman. KinP allows you to run Kubernetes clusters within Podman containers on your local system.

## Prerequisites
- podman
- kind
- kubectl

## Steps

### Ensure the Host is Running with Control Group v2

kind requires `cgroup v2` to be configured in a specific manner to work. Make sure that the result of the `podman info` command contains `cgroupVersion: v2`. If it prints `cgroupVersion: v1`, try the following steps to accomplish this.

**1. Open a Text Editor:** 

Open a text editor like Notepad or any other text editor of your choice.

**2. Create the .wslconfig File:**

In the text editor, create a new file and save it as `.wslconfig` in your `%UserProfile%` directory. If you're not sure where this is, you can enter `%UserProfile%` in the address bar of File Explorer to navigate to it.

**3. Add Configuration:**

Inside the `.wslconfig` file, add the following content:
```css
[wsl2]
kernelCommandLine = cgroup_no_v1=all
```
**4. Save the File:**

Save the file after adding the configuration.

**5. Shutdown WSL:**

Open PowerShell or Command Prompt as Administrator and run the following command to shut down WSL:
```arduino
wsl.exe --shutdown
```

**6. Restart WSL:**

After shutting down WSL, reopen your terminal or WSL distribution. This will restart WSL with the new configuration.

### Enabling CPU, CPUSET, and I/O delegation

By default, a non-root user can only get memory controller and pids controller to be delegated. To allow delegation of other controllers such as cpu, cpuset, and io, run the following commands:

**1. Open a terminal window on your Linux system** 

**2. Create the configuration file to enable delegation of the CPU controller**
```sh
sudo mkdir /etc/systemd/system/user@.service.d
```

**3. Create a system-wide systemd user unit file to enable cgroup controller delegation**
```sh
cat << EOF | sudo tee /etc/systemd/system/user@.service.d/delegate.conf > /dev/null
[Service]
Delegate=yes
EOF
```

**4. Reload the systemd daemon**
```sh
sudo systemctl daemon-reload
```

After changing the systemd configuration, you need to re-login or reboot the host. Rebooting the host is recommended.

### Installing the Kubernetes Command Line Tool (kubectl)

kubectl is a command-line tool for interacting with Kubernetes clusters. It allows you to deploy and manage applications, inspect and manage cluster resources, and view logs.

This guide provides step-by-step instructions on how to install kubectl on various operating systems.

**1. Download the latest version of kubectl:**
```sh
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```
**2. Install kubectl:**
```sh
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

**3. Verify the kubectl installation**
```sh
kubectl version --client --output=yaml
```

Example Output:
```sh
$ kubectl version --client --output=yaml
clientVersion:
  buildDate: "2024-03-15T00:08:19Z"
  compiler: gc
  gitCommit: 6813625b7cd706db5bc7388921be03071e1a492d
  gitTreeState: clean
  gitVersion: v1.29.3
  goVersion: go1.21.8
  major: "1"
  minor: "29"
  platform: linux/amd64
kustomizeVersion: v5.0.4-0.20230601165947-6ce0bf390ce3
```

### Creating a kind cluster with Rootless Podman

**1. Configure your shell:**

Using rootless Podman to run kind needs an environment variable to be declared. Add it to the BASH environment.
```sh
cat << EOF >> ~/.bashrc
export KIND_EXPERIMENTAL_PROVIDER=podman
EOF
```

**2. Source the ~/.bashrc file to pick up the change:**
```sh
source ~/.bashrc
```

**3. Download and Install kind**
```sh
sudo curl -Lo /usr/local/bin/kind "https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64" && sudo chmod +x /usr/local/bin/kind
```

**4. Confirm the kind version**
```sh
kind --version
```
Example Output:
```sh
$ kind --version
kind version 0.22.0
```

**5. Create a single node kind cluster**
```sh
kind create cluster --name local-kind-cluster --config <(echo '
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  apiServerAddress: 127.0.0.1
  apiServerPort: 6443
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30000
    hostPort: 30000
    listenAddress: "0.0.0.0"
    protocol: tcp
  - containerPort: 30001
    hostPort: 30001
    listenAddress: "0.0.0.0"
    protocol: tcp
  - containerPort: 30002
    hostPort: 30002
    listenAddress: "0.0.0.0"
    protocol: tcp
')
```

Example Output:
```sh
$ kind create cluster --name local-kind-cluster --config <(echo '
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  apiServerAddress: 127.0.0.1
  apiServerPort: 6443
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30000
    hostPort: 30000
    listenAddress: "0.0.0.0"
    protocol: tcp
  - containerPort: 30001
    hostPort: 30001
    listenAddress: "0.0.0.0"
    protocol: tcp
  - containerPort: 30002
    hostPort: 30002
    listenAddress: "0.0.0.0"
    protocol: tcp
')
using podman due to KIND_EXPERIMENTAL_PROVIDER
enabling experimental podman provider
Cgroup controller detection is not implemented for Podman. If you see cgroup-related errors, you might need to set systemd property "Delegate=yes", see https://kind.sigs.k8s.io/docs/user/rootless/
Creating cluster "local-kind-cluster" ...
 ✓ Ensuring node image (kindest/node:v1.29.2) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-local-kind-cluster"
You can now use your cluster with:

kubectl cluster-info --context kind-local-kind-cluster

Have a question, bug, or feature request? Let us know! https://kind.sigs.k8s.io/#community 🙂
```

**6. Confirm that kubectl can connect to the kind-based Kubernetes cluster**
```sh
kubectl cluster-info --context kind-local-kind-cluster
```
Example Output:
```sh
$ kubectl cluster-info --context kind-local-kind-cluster
Kubernetes control plane is running at https://127.0.0.1:6443
CoreDNS is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

**7. Create Deployment YAML**

This YAML file defines a Kubernetes deployment for an Nginx web server with two replicas. It uses the Nginx Docker image and runs a command to modify the index.html file before starting Nginx. The modification replaces occurrences of "nginx" with the hostname of the pod.
```yaml
cat <<EOF > nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
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
        command: ["/bin/sh", "-c"]
        args:
        - sed -i "s/nginx/$(hostname)/" /usr/share/nginx/html/index.html && nginx -g 'daemon off;'
        ports:
        - containerPort: 80
EOF
```

**8. Apply the Deployment**

Then, you can apply these YAML manifests using kubectl:
```sh
kubectl apply -f nginx-deployment.yaml
```

**9. Create Nginx Service**

Access the Nginx service by using the service's Cluster IP or by exposing it externally (if applicable).

```sh
kubectl expose deployment nginx --port=80 --target-port=80 --name=mginx-service --type=ClusterIP
```

Example Output:
```sh
$ kubectl get svc -n nginx
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
nginx-service   ClusterIP   10.96.127.93   <none>        80/TCP    9m25s
```

After following these steps, you should have a Nginx deployment running with two replicas serving different content, exposed as load-balanced services.

Example Output:
```sh
$ kubectl get pod -n nginx
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-5f57746ffb-g82l7   1/1     Running   0          9m16s
nginx-deployment-5f57746ffb-nlpwq   1/1     Running   0          9m16s
```

**10. Access Nginx Service**
To access your service in such cases, you can try port forwarding instead of relying on the external IP. Here's how you can do it for your `nginx-service`:
```sh
kubectl port-forward service/nginx-service 8080:80
```

This command forwards traffic from your local port 8080 to port 80 of the nginx-service. You can then access your service at http://localhost:8080 in your browser or via curl command.

## (Optional) Verify the Kubernetes Environment
Run a few commands to confirm the deployment of a fully functional Kubernetes cluster.

**1. Use kind to check for the details of any running kind clusters**
```sh
kind get clusters
```
Example Output:
```sh
$ kind get clusters
using podman due to KIND_EXPERIMENTAL_PROVIDER
enabling experimental podman provider
local-kind-cluster
```

**2. Check what Podman containers are running**
```sh
podman ps -a
```
Example Output:
```sh
$ podman ps -a
CONTAINER ID  IMAGE                                                                                           COMMAND               CREATED         STATUS             PORTS                                                           NAMES
7ab3b9020454  docker.io/kindest/node@sha256:51a1434a5397193442f0be2a297b488b6c919ce8a3931be0ce822606ea5ca245                        25 minutes ago  Up 25 minutes ago  127.0.0.1:6443->6443/tcp, 0.0.0.0:30000-30002->30000-30002/tcp  local-kind-cluster-control-plane
```

**3. Confirm kubectl knows about the newly creted cluster**
```sh
grep server ~/.kube/config
```

Example Output:
```sh
$ grep server ~/.kube/config
    server: https://127.0.0.1:6443
```

**4. Take a quick look at the kind cluster using the `kubectl get nodes` command**
```sh
kubectl get nodes
```
Example Output:
```sh
$ kubectl get nodes
NAME                               STATUS   ROLES           AGE   VERSION
local-kind-cluster-control-plane   Ready    control-plane   31m   v1.29.2
```

**5. Confirm that a fully functional Kubernetes node was started**
```sh
kubectl get pods -A
```
Example Output:
```sh
$ kubectl get pods -A
NAMESPACE            NAME                                                       READY   STATUS    RESTARTS   AGE
kube-system          coredns-76f75df574-5cc28                                   1/1     Running   0          34m
kube-system          coredns-76f75df574-xpmxj                                   1/1     Running   0          34m
kube-system          etcd-local-kind-cluster-control-plane                      1/1     Running   0          34m
kube-system          kindnet-74jj7                                              1/1     Running   0          34m
kube-system          kube-apiserver-local-kind-cluster-control-plane            1/1     Running   0          34m
kube-system          kube-controller-manager-local-kind-cluster-control-plane   1/1     Running   0          34m
kube-system          kube-proxy-9jwl6                                           1/1     Running   0          34m
kube-system          kube-scheduler-local-kind-cluster-control-plane            1/1     Running   0          34m
local-path-storage   local-path-provisioner-7577fdbbfb-lpvq8                    1/1     Running   0          34m
```

**6. Confirm that containerd (and not Podman) is running the kind cluster**
```sh
kubectl get nodes -o wide
```

Example Output:
```sh
$ kubectl get nodes -o wide
NAME                               STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION                       CONTAINER-RUNTIME
local-kind-cluster-control-plane   Ready    control-plane   36m   v1.29.2   10.89.1.2     <none>        Debian GNU/Linux 12 (bookworm)   5.15.146.1-microsoft-standard-WSL2   containerd://1.7.13
```

## Troubleshooting

### Fixing containers failing to start with systemd

If you’re using either Docker or Podman, the situation is as follows:
```sh
Error: OCI runtime error: chmod `run/shm`: Operation not supported
```

The workaround is replacing the existing OCI runtime binary (`/usr/bin/crun`) with a newer version obtained from the GitHub releases page of the `crun` project. Here's a step-by-step guide to implementing this solution:

**1. Download the Binary**

- Visit the [GitHub releases page](https://github.com/containers/crun/releases/latest) for the `crun` project.
- Download the appropriate binary for your system. Ensure that you select the correct version compatible with your operating system and architecture.

**2. Backup (Optional):**
- As a precaution, you might want to back up the existing `/usr/bin/crun` binary before replacing it. This allows you to revert to the original version if needed.

**3. Replace the Existing Binary:**

- Once downloaded, you'll likely need administrative privileges to replace the existing binary.
- Move the downloaded binary to `/usr/bin/crun`, overwriting the existing one. You may need to use the `sudo` command or switch to the root user for this operation.

**4. Set Permissions:**
- After replacing the binary, ensure that the new binary has the correct permissions set to allow execution.

**5. Verify:**
- Verify that you’re indeed on the latest release now.
```sh
$ crun --version
crun version 1.14.4
commit: a220ca661ce078f2c37b38c92e66cf66c012d9c1
rundir: /run/user/1000/crun
spec: 1.0.0
+SYSTEMD +SELINUX +APPARMOR +CAP +SECCOMP +EBPF +CRIU +YAJL
```

**6. Test:**
- After replacing the binary, test your container runtime to ensure that the issue has been resolved. You can try running containers that were previously failing to see if the error persists.

### Podman automatically sets cniVersion 1.0.0 instead of 0.4.0

If the version of your CNI plugin does not correctly match the plugin version in the config because the config version is later than the plugin version, the containerd log will likely show an error message on startup of a pod similar to:
```sh
WARN[0000] Error validating CNI config file /home/rgill/.config/cni/net.d/kind.conflist: [plugin bridge does not support config version "1.0.0" plugin portmap does not support config version "1.0.0" plugin firewall does not support config version "1.0.0" plugin tuning does not support config version "1.0.0"]
```

The workaround is replacing the existing network in Podman. 

**1. Removing the Existing `kind` Network:**

Use the `podman network rm` command to remove the existing `kind` network.
```sh
$ podman network rm kind
```

**2. Manually Add `kind` Network:**
```sh
$ podman network create kind
```
Example Output:
```sh
podman network create kind
/home/rgill/.config/cni/net.d/kind.conflist
```

**3. Update `CNI Version` in Configuration File**
```sh
$ sed -i 's/"cniVersion": "1.0.0"/"cniVersion": "0.4.0"/g' $HOME/.config/cni/net.d/kind.conflist
```
 Use the sed command to find the string `"cniVersion": "1.0.0"` and replaces it with `"cniVersion": "0.4.0"` in the file 

 This command utilizes `sed` to replace the existing value of `"cniVersion": "1.0.0"` with `"cniVersion": "0.4.0"` within a configuration file located at *$HOME/.config/cni/net.d/kind.conflist*.

**4. Verification**

List all networks:
```sh
$ podman network ls
```
Example Output:
```sh
$ podman network ls
NETWORK ID    NAME                           VERSION     PLUGINS
2f259bab93aa  podman                         0.4.0       bridge,portmap,firewall,tuning
d0653b3921c9  developer_workstation_default  0.4.0       bridge,portmap,firewall,tuning,dnsname
0b27c158feb3  kind                           0.4.0       bridge,portmap,firewall,tuning,dnsname
```
You can see the created `kind` network using the correct version `0.4.0`.

Podman uses two different means for its networking stack, depending on whether the container is rootless or rootfull. When rootfull, defined as being run by the root (or equivalent) user, Podman primarily relies on the [containernetworking plugins](https://github.com/containernetworking/plugins) project. When rootless, defined as being run by a regular user, Podman uses the [slirp4netns](https://github.com/rootless-containers/slirp4netns) project.

## Conclusion
You have successfully set up a local Kubernetes deployment with KinD and Podman involves several steps to ensure smooth operation and compatibility.

By incorporating these tools into your workflow, you can streamline your Kubernetes development process and accelerate the delivery of containerized applications.