Here's a `README.md` file for setting up a 3-node etcd cluster on Ubuntu machines, along with an HAProxy load balancer.

```markdown
# 3-Node etcd Cluster Installation on Ubuntu

This guide will walk you through setting up a 3-node etcd cluster on Ubuntu machines with an HAProxy load balancer for high availability. The etcd nodes will have the following hostnames and IP addresses:

- `etcd1`: 10.0.1.13
- `etcd2`: 10.0.1.14
- `etcd3`: 10.0.1.15

## Prerequisites

- Three Ubuntu VMs for the etcd nodes (`etcd1`, `etcd2`, and `etcd3`).
- One Ubuntu VM for HAProxy.
- Root access to all VMs.
- Internet access on all VMs to download etcd.

## Step 1: Initial Setup on etcd Nodes

After creating the three VMs, perform the initial setup on all etcd nodes:

### 1. Enable SSH Password Authentication and Set Root Password

Execute the following script on all three etcd nodes to enable SSH password authentication and set the root password:

```bash
#!/bin/bash

# Enable SSH password authentication
echo "[TASK 1] Enable SSH password authentication"
sed -i 's/^PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
echo "PermitRootLogin yes" >> /etc/ssh/sshd_config
systemctl reload sshd

# Set root password
echo "[TASK 2] Set root password"
echo -e "etcdadmin\netcdadmin" | passwd root >/dev/null 2>&1
```

## Step 2: Install etcd on All Nodes

Run the following commands on all three etcd nodes to install etcd:

```bash
ETCD_VER=v3.5.1
wget -q --show-progress "https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz"
tar zxf etcd-v3.5.1-linux-amd64.tar.gz
mv etcd-v3.5.1-linux-amd64/etcd* /usr/local/bin/
rm -rf etcd*
```

## Step 3: Configure and Start etcd Service

For each etcd node, create a systemd service file with the respective configuration. **Replace the IP addresses and node names as necessary**:

### On `etcd1` (10.0.1.13):

```bash
[Unit]
Description=etcd key-value store
Documentation=https://github.com/coreos/etcd
After=network.target

[Service]
ExecStart=/usr/local/bin/etcd \
  --name etcd1 \
  --initial-advertise-peer-urls http://10.0.1.13:2380 \
  --listen-peer-urls http://10.0.1.13:2380 \
  --advertise-client-urls http://10.0.1.13:2379 \
  --listen-client-urls http://10.0.1.13:2379,http://127.0.0.1:2379 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster etcd1=http://10.0.1.13:2380,etcd2=http://10.0.1.14:2380,etcd3=http://10.0.1.15:2380 \
  --initial-cluster-state new
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### On `etcd2` (10.0.1.14):

```bash
[Unit]
Description=etcd key-value store
Documentation=https://github.com/coreos/etcd
After=network.target

[Service]
ExecStart=/usr/local/bin/etcd \
  --name etcd2 \
  --initial-advertise-peer-urls http://10.0.1.14:2380 \
  --listen-peer-urls http://10.0.1.14:2380 \
  --advertise-client-urls http://10.0.1.14:2379 \
  --listen-client-urls http://10.0.1.14:2379,http://127.0.0.1:2379 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster etcd1=http://10.0.1.13:2380,etcd2=http://10.0.1.14:2380,etcd3=http://10.0.1.15:2380 \
  --initial-cluster-state new
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### On `etcd3` (10.0.1.15):

```bash
[Unit]
Description=etcd key-value store
Documentation=https://github.com/coreos/etcd
After=network.target

[Service]
ExecStart=/usr/local/bin/etcd \
  --name etcd3 \
  --initial-advertise-peer-urls http://10.0.1.15:2380 \
  --listen-peer-urls http://10.0.1.15:2380 \
  --advertise-client-urls http://10.0.1.15:2379 \
  --listen-client-urls http://10.0.1.15:2379,http://127.0.0.1:2379 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster etcd1=http://10.0.1.13:2380,etcd2=http://10.0.1.14:2380,etcd3=http://10.0.1.15:2380 \
  --initial-cluster-state new
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### Start and Enable etcd Service

Run the following commands on all three etcd nodes:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now etcd
sudo systemctl start etcd --now
sudo systemctl status etcd
```

## Step 4: Install and Configure HAProxy

On the HAProxy VM, install HAProxy and configure it to load balance etcd and Kubernetes API servers:

### Install HAProxy

```bash
sudo apt update
sudo apt install -y haproxy
```

### Configure HAProxy

Edit the HAProxy configuration file `/etc/haproxy/haproxy.cfg`:

```bash
frontend kubernetes_apiservers
    bind *:6443
    mode tcp
    option tcplog
    default_backend kubernetes_masters

backend kubernetes_masters
    mode tcp
    balance roundrobin
    server master1 <master1-ip>:6443 check
    server master2 <master2-ip>:6443 check
    server master3 <master3-ip>:6443 check

frontend etcd_cluster
    bind *:2379
    mode tcp
    option tcplog
    default_backend etcd_servers

backend etcd_servers
    mode tcp
    balance roundrobin
    server etcd1 10.0.1.13:2379 check
    server etcd2 10.0.1.14:2379 check
    server etcd3 10.0.1.15:2379 check
```

### Restart HAProxy

After saving the configuration, restart HAProxy:

```bash
sudo systemctl restart haproxy
```

## Conclusion

Your 3-node etcd cluster with HAProxy load balancing should now be set up and ready for use. You can verify the etcd cluster status and ensure that the HAProxy is properly routing requests to the etcd nodes.
```

This `README.md` file covers the prerequisites, step-by-step installation, and configuration for setting up a 3-node etcd cluster with HAProxy load balancing. Make sure to replace the placeholder `<master1-ip>`, `<master2-ip>`, and `<master3-ip>` in the HAProxy configuration with your actual master node IPs if needed.
