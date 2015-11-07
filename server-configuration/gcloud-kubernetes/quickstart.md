# CoreOS, Kubernetes and Gcloud
# Quick Start Guide

## Provision 4 CoreOS nodes:

### CREATE FOUR NEW INSTANCES

```bash
for i in {0..3}; do
  gcloud compute instances create node${i} \
  --image-project coreos-cloud \
  --image coreos-stable-717-3-0-v20150710 \
  --boot-disk-size 200GB \
  --machine-type n1-standard-1 \
  --can-ip-forward \
  --scopes compute-rw
done
```

Verify Nodes

```bash
gcloud compute instances list
```

Create and Enable Routes

```bash
gcloud compute routes create default-route-10-200-1-0-24 --destination-range 10.200.1.0/24 --next-hop-instance node1 && gcloud compute routes create default-route-10-200-2-0-24 --destination-range 10.200.2.0/24 --next-hop-instance node2 && gcloud compute routes create default-route-10-200-3-0-24 --destination-range 10.200.3.0/24 --next-hop-instance node3
```

```bash
gcloud compute ssh node1 --command "sudo iptables -t nat -A POSTROUTING ! -d 10.0.0.0/8 -o ens4v1 -j MASQUERADE"
```

```bash
gcloud compute ssh node2  --command "sudo iptables -t nat -A POSTROUTING ! -d 10.0.0.0/8 -o ens4v1 -j MASQUERADE"
```

```bash
gcloud compute ssh node3 --command "sudo iptables -t nat -A POSTROUTING ! -d 10.0.0.0/8 -o ens4v1 -j MASQUERADE"
```

Verify Services

```bash
gcloud compute routes list
```


### Node 0

Download and Start Services

```bash
sudo curl https://kuar.io/etcd.service -o /etc/systemd/system/etcd.service && sudo systemctl daemon-reload && sudo systemctl enable etcd && sudo systemctl start etcd && sudo systemctl status etcd && sleep .5 && sudo curl https://kuar.io/kube-apiserver.service -o /etc/systemd/system/kube-apiserver.service && sudo systemctl daemon-reload && sudo systemctl enable kube-apiserver && sudo systemctl start kube-apiserver && sudo systemctl status kube-apiserver && sleep .5 && sudo curl https://kuar.io/kube-controller-manager.service -o /etc/systemd/system/kube-controller-manager.service && sleep .5 && sudo systemctl daemon-reload && sudo systemctl enable kube-controller-manager && sudo systemctl start kube-controller-manager && sudo systemctl status kube-controller-manager && sudo curl https://kuar.io/kube-scheduler.service -o /etc/systemd/system/kube-scheduler.service && sleep .5 && sudo curl -o /opt/bin/kubectl https://kuar.io/linux/kubectl && sleep .5 && sudo chmod +x /opt/bin/kubectl && sudo systemctl daemon-reload && sudo systemctl enable kube-scheduler && sudo systemctl start kube-scheduler && sudo systemctl status kube-scheduler
```


Check Setup:

```bash
kubectl get cs
kubectl get pods
kubectl get nodes
kubectl get services
```

### node1

Download Services:

```bash
sudo curl https://kuar.io/docker.service -o /etc/systemd/system/docker.service && sudo curl https://kuar.io/kubelet.service -o /etc/systemd/system/kubelet.service && sudo curl https://kuar.io/kube-proxy.service -o /etc/systemd/system/kube-proxy.service
```

Configure Services:

```bash
# Configure Docker
sudo vim /etc/systemd/system/docker.service
# --bip=10.200.1.1/24 \

# Configure Kublet
sudo vim /etc/systemd/system/kubelet.service
# --api-servers=http://node0.c.PROJECT_NAME.internal:8080 \

# Configure Service Proxy
sudo vim /etc/systemd/system/kube-proxy.service
# --master=http://node0.c.PROJECT_NAME.internal:8080
```

Start Services:

```bash
sudo systemctl daemon-reload && sudo systemctl enable docker && sudo systemctl start docker && sudo systemctl daemon-reload && sudo systemctl enable kubelet && sudo systemctl start kubelet && sudo systemctl status kubelet && sudo systemctl daemon-reload && sudo systemctl enable kube-proxy && sudo systemctl start kube-proxy && sudo systemctl status kube-proxy
```

Verify Services:

```bash
ifconfig;
docker version;
docker run -t -i --rm busybox /bin/sh;
ip -f inet addr show eth0;
```

----

### node2

Download Services:

```bash
sudo curl https://kuar.io/docker.service -o /etc/systemd/system/docker.service && sudo curl https://kuar.io/kubelet.service -o /etc/systemd/system/kubelet.service && sudo curl https://kuar.io/kube-proxy.service -o /etc/systemd/system/kube-proxy.service
```

Configure Services

```bash
# Configure Docker
sudo vim /etc/systemd/system/docker.service
# --bip=10.200.2.1/24 \

# Configure Kublet
sudo vim /etc/systemd/system/kubelet.service
# --api-servers=http://node0.c.PROJECT_NAME.internal:8080 \

# Configure Service Proxy
sudo vim /etc/systemd/system/kube-proxy.service
# --master=http://node0.c.PROJECT_NAME.internal:8080
```

Start Services:

```bash
sudo systemctl daemon-reload && sudo systemctl enable docker && sudo systemctl start docker && sudo systemctl daemon-reload && sudo systemctl enable kubelet && sudo systemctl start kubelet && sudo systemctl status kubelet && sudo systemctl daemon-reload && sudo systemctl enable kube-proxy && sudo systemctl start kube-proxy && sudo systemctl status kube-proxy
```

----

### node3

Download Services:

```bash
# Download Docker
sudo curl https://kuar.io/docker.service -o /etc/systemd/system/docker.service
# Download Kublet unit file
sudo curl https://kuar.io/kubelet.service -o /etc/systemd/system/kubelet.service
```

Configure Services

```bash
# Configure Docker
sudo vim /etc/systemd/system/docker.service
# --bip=10.200.3.1/24 \

# Configure Kublet
sudo vim /etc/systemd/system/kubelet.service
# --api-servers=http://node0.c.PROJECT_NAME.internal:8080 \

# Configure Service Proxy
sudo vim /etc/systemd/system/kube-proxy.service
# --master=http://node0.c.PROJECT_NAME.internal:8080
```

Start Services:

```bash
sudo systemctl daemon-reload && sudo systemctl enable docker && sudo systemctl start docker && sudo systemctl daemon-reload && sudo systemctl enable kubelet && sudo systemctl start kubelet && sudo systemctl status kubelet && sudo systemctl daemon-reload && sudo systemctl enable kube-proxy && sudo systemctl start kube-proxy && sudo systemctl status kube-proxy
```

----

## Verify Configuration

### Node0

```bash
ifconfig;
docker version;
ping -c 3 10.200.1.2;
ping -c 3 google.com;
/opt/bin/kubectl get nodes
```

### Node1

```bash
sudo iptables -vL -n -t nat
```

### Node2

```bash
sudo iptables -vL -n -t nat
```

### Node3

```bash
sudo iptables -vL -n -t nat
```

----
## App Containers

```bash
kubectl run redis \
  --labels="app=redis,track=stable" \
  --image=dockerfile/redis
  
  kubectl run elasticsearch \
  --labels="app=elasticsearch,,track=stable" \
  --image=dockerfile/elasticsearch
```
