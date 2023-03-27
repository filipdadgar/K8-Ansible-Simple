**This playbook installs and configures Containerd and Kubernetes components on all nodes (both master and worker nodes). It then initializes the Kubernetes master (control plane) with the desired API server settings and sets up Flannel CNI. Finally, it joins the worker nodes to the cluster.**

To run the playbook, execute the following command:

```python
ansible-playbook -i inventory.ini k8s-setup.yaml
```

**After the playbook finishes, you should have a fully functional Kubernetes cluster with containerd as the container runtime and Flannel as the CNI. The API server should be reachable from the 192.168.0.0/22 subnet. Make sure to copy the /etc/kubernetes/admin.conf file from the master node to your local machine to interact with the cluster using kubectl.**

**Note:** The playbook is designed to be idempotent. You can run it multiple times without any issues.

**For a HA setup, modify the inventory file:**

```python
[masters]
192.168.1.1
192.168.1.2
192.168.1.3
192.168.1.4
```

* Update the k8-setup.yaml file with the desired API server settings:

```python
- name: Join additional master nodes to the cluster
  hosts: masters[1:]  # Skip the first master node, as it is already initialized
  become: yes
  tasks:
    - name: Get join command
      become: no
      shell: kubeadm token create --print-join-command --ttl 0
      register: join_command
      delegate_to: 192.168.1.1
      when: "'192.168.1.1' in inventory_hostname"
      run_once: true

    - name: Join the cluster as a master node
      command: "{{ join_command.stdout }} --control-plane"
      args:
        creates: /etc/kubernetes/kubelet.conf
```

* Run the playbook:

```python
ansible-playbook -i inventory.ini k8s-setup.yaml
```

** When you have multiple master nodes, it's recommended to use a load balancer to distribute traffic among the control plane nodes. HAProxy is one such option for load balancing. By setting up HAProxy, you can achieve high availability and fault tolerance for your Kubernetes control plane.

Here is an example for HAProxy:


Install HAProxy on a separate node or one of the existing nodes.
Create an HAProxy configuration file, e.g., haproxy.cfg, with the following content:
    
```python
    global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

defaults
    log global
    mode tcp
    option tcplog
    option dontlognull
    timeout connect 5000
    timeout client 50000
    timeout server 50000

frontend kubernetes-frontend
    bind *:6443
    mode tcp
    option tcplog
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    option tcplog
    option tcp-check
    balance roundrobin
    server master-1 192.168.1.1:6443 check
    server master-2 192.168.1.2:6443 check
    server master-3 192.168.1.3:6443 check
    server master-4 192.168.1.4:6443 check
```

Start HAProxy:

```python
sudo systemctl start haproxy
sudo systemctl enable haproxy
```

Update the k8s-setup.yaml file with the HAProxy IP address:

```python
- name: Join additional master nodes to the cluster
  hosts: masters[1:]  # Skip the first master node, as it is already initialized
  become: yes
  tasks:
    - name: Get join token
      become: no
      shell: kubeadm token create
      register: join_token
      delegate_to: 192.168.1.1
      when: "'192.168.1.1' in inventory_hostname"
      run_once: true

    - name: Join the cluster as a master node
      command: kubeadm join haproxy_ip_or_dns:6443 --token {{ join_token.stdout }} --discovery-token-unsafe-skip-ca-verification --control-plane
      args:
        creates: /etc/kubernetes/kubelet.conf
```

Replace haproxy_ip_or_dns with the IP address or DNS name of the node running HAProxy.
