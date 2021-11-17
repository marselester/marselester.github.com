title: BPF Go program in Kubernetes
date: 2021-11-17
tags: bpf, golang, kubernetes
category: Go
slug: bpf-go-program-in-kubernetes

BPF opens a lot of possibilities of making observability tools running in Kubernetes.
One can start with [BCC libbpf-tools](https://github.com/iovisor/bcc/tree/master/libbpf-tools) written in C,
e.g., launch tcpconnlat program and process its stdout with another program to
detect cases when it took too long to establish a TCP connection.
For example, `curl http://example.com` took 47.8 milliseconds to establish a connection
where source address is `10.0.2.15` and destination address is `93.184.216.34`.

```console
PID    COMM         IP SADDR            DADDR            DPORT LAT(ms)
21500  curl         4  10.0.2.15        93.184.216.34    80    47.80
```

Another option is to use Go version of
[tcpconnlat](https://github.com/marselester/libbpf-tools/blob/master/cmd/tcpconnlat/main.go)
and modify it as you wish, e.g., write the events into Kafka for further analysis.

I have already published a Docker image
[marselester/go-libbpf-tools](https://github.com/marselester/libbpf-tools/blob/master/Dockerfile)
containing tcpconnlat and tried to run it on Mac, that didn't go well though.

```console
﹪ docker run --rm -it --privileged marselester/go-libbpf-tools:latest bash
root@552a159ce901:/opt/libbpf-tools＃ ./tcpconnlat
failed to load BPF programs and maps: field TcpRcvStateProcess: program tcp_rcv_state_process: CO-RE relocations: no BTF for kernel version 5.10.47-linuxkit: not supported
```

Minikube with Virtualbox driver didn't help either because it uses an old kernel,
hopefully it will be upgraded soon [#10501](https://github.com/kubernetes/minikube/issues/10501).

```console
﹪ minikube start --driver=virtualbox
﹪ minikube ssh
﹩ uname -nr
Linux minikube 4.19.202
```

Luckily there is another option called [Kubespray](https://kubespray.io).

## Kubespray

Kubespray sets up a Kubernetes cluster of 3 nodes using Vagrant and Ansible.
Clone the repository and install Python dependencies for provisioning tasks.

```console
﹪ git clone https://github.com/kubernetes-sigs/kubespray
﹪ cd ./kubespray/
﹪ virtualenv venv
﹪ source ./venv/bin/activate
(venv) ﹪ pip install -r requirements.txt
```

From looking at [the Vagrantfile](https://github.com/kubernetes-sigs/kubespray/blob/master/Vagrantfile)
we see that Kubespray supports Fedora Linux 34, so BTF and CO-RE technologies should be there.

```console
﹪ mkdir vagrant
﹪ echo '$os = "fedora34"' > ./vagrant/config.rb
﹪ vagrant up
﹪ export KUBECONFIG=$(pwd)/.vagrant/provisioners/ansible/inventory/artifacts/admin.conf
﹪ kubectl get nodes
NAME    STATUS   ROLES                  AGE     VERSION
k8s-1   Ready    control-plane,master   8m20s   v1.22.3
k8s-2   Ready    control-plane,master   7m56s   v1.22.3
k8s-3   Ready    <none>                 6m58s   v1.22.3
﹪ vagrant ssh k8s-1
﹩ uname -nr
k8s-1 5.11.12-300.fc34.x86_64
```

See [vagrant.md](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/vagrant.md)
if the cluster wasn't provisioned.

## DaemonSet

An observability tool should run on each node,
and a [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)
ensures that all nodes run a copy of a pod.
Let's try to launch tcpconnlat on all 3 nodes.

```console
﹪ kubectl apply -f - <<<'
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: tcpconnlat-daemon
spec:
  selector:
    matchLabels:
      app: tcpconnlat
  template:
    metadata:
      labels:
        app: tcpconnlat
    spec:
      containers:
        - name: libbpf-tools
          image: marselester/go-libbpf-tools:latest
          command:
            - /opt/libbpf-tools/tcpconnlat
'
```

Unfortunately pods have crashed because the containers didn't have privileged mode.

```console
﹪ kubectl get pods
NAME                      READY   STATUS             RESTARTS        AGE
tcpconnlat-daemon-646h2   0/1     CrashLoopBackOff   5 (2m9s ago)    5m13s
tcpconnlat-daemon-hwnzt   0/1     CrashLoopBackOff   5 (118s ago)    5m13s
tcpconnlat-daemon-tmn6r   0/1     CrashLoopBackOff   5 (2m14s ago)   5m13s
﹪ kubectl logs -f tcpconnlat-daemon-646h2
failed to set temporary RLIMIT_MEMLOCK: operation not permitted
```

Let's enable it.

> By default a container is not allowed to access any devices on the host,
> but a "privileged" container is given access to all devices on the host.
> This allows the container nearly all the same access as processes running on the host.
>
> https://kubernetes.io/docs/concepts/policy/pod-security-policy/#privileged

```console
﹪ kubectl apply -f - <<<'
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: tcpconnlat-daemon
spec:
  selector:
    matchLabels:
      app: tcpconnlat
  template:
    metadata:
      labels:
        app: tcpconnlat
    spec:
      containers:
        - name: libbpf-tools
          image: marselester/go-libbpf-tools:latest
          command:
            - /opt/libbpf-tools/tcpconnlat
          securityContext:
            privileged: true
'
```

It works! 👇

```console
﹪ kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
tcpconnlat-daemon-9sgc5   1/1     Running   0          18s
tcpconnlat-daemon-lrvh5   1/1     Running   0          18s
tcpconnlat-daemon-th9k4   1/1     Running   0          18s
﹪ kubectl logs -f tcpconnlat-daemon-9sgc5
PID    COMM         IP SADDR            DADDR            DPORT LAT(ms)
5955   coredns      4  127.0.0.1        127.0.0.1        8080  0.02
703    kubelet      4  172.18.8.101     172.18.8.101     6443  0.04
703    kubelet      4  10.233.64.1      10.233.64.4      8181  0.07
703    kubelet      4  169.254.25.10    169.254.25.10    9254  0.03
703    kubelet      4  10.233.64.1      10.233.64.5      8080  0.03
5537   node-cache   4  169.254.25.10    169.254.25.10    9254  0.06
4204   etcd         4  172.18.8.101     172.18.8.103     2380  0.22
4204   etcd         4  172.18.8.101     172.18.8.102     2380  0.21
```

I also tried to run execsnoop and tcpconnect, but alas they crashed with the corresponding errors.

```console
failed to attach the BPF program to sys_enter_execve tracepoint: trace event syscalls/sys_enter_execve: file does not exist

failed to load BPF programs and maps: field TcpV4ConnectRet: program tcp_v4_connect_ret: load program: permission denied: trace type programs with run-time allocated hash maps are unsafe. Switch to preallocated hash maps.
```
