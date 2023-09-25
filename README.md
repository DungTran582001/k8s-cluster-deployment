# k8s-cluster-deployment

![alt](overview.png)
## Single node:
1. Install container runtime (Docker)

- Set up the repository:
```yml
sudo apt-get update
-------------------------------------------------------------------------------------------------------
sudo apt-get install ca-certificates curl gnupg
```
```yml
sudo install -m 0755 -d /etc/apt/keyrings
-------------------------------------------------------------------------------------------------------
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

```yml
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

```yml
sudo apt-get update
-------------------------------------------------------------------------------------------------------
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

> Add current user in docker group by the command: `sudo usermod -aG docker $USER`. The user needs to Log out and log back into the Ubuntu server by command: `sudo reboot` so that group membership is re-evaluated. After that the user will be able to run Docker commands without using root or sudo.
2. Install kubeadm, kubelet, kubectl in each node:
- Disable swap space in `each node`:
```yml
# Disable swap
sudo swapoff -a

# Disable swap every time node starts in /etc/fstab
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

```
- Install tools:
```yml
sudo apt-get update
-------------------------------------------------------------------------------------------------------
sudo apt-get install -y apt-transport-https ca-certificates curl
```
```yml
sudo mkdir -m 0755 -p /etc/apt/keyrings
-------------------------------------------------------------------------------------------------------
sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

```

```yml
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

```yml
sudo apt-get update
-------------------------------------------------------------------------------------------------------
sudo apt-get install -y kubelet kubeadm kubectl
-------------------------------------------------------------------------------------------------------
sudo apt-mark hold kubelet kubeadm kubectl
```
3. Init control plane:
- Init master node:
```yml
kubeadm init --apiserver-advertise-address=<IP_node_master> --pod-network-cidr=192.168.0.0/16
```
> Should save the command `kubectl join <IP_node_master>:6443 --token...` appeared in output after running the above command. It's responsible for helping other nodes to join the cluster.
```yml
mkdir -p $HOME/.kube
-------------------------------------------------------------------------------------------------------
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
-------------------------------------------------------------------------------------------------------
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
- Install plugin network to help nodes in cluster can communicate together. (Choose plugin `Calico`)
```yml
# Install the Tigera Calico operator and custom resource definitions.
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/tigera-operator.yaml
-------------------------------------------------------------------------------------------------------
# Install Calico by creating the necessary custom resource. For more information on configuration options available in this manifest.
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/custom-resources.yaml
-------------------------------------------------------------------------------------------------------
# Confirm that all of the pods are running with the following command.
watch kubectl get pods -n calico-system
-------------------------------------------------------------------------------------------------------
# Remove the taints on the control plane so that you can schedule pods on it.
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
kubectl taint nodes --all node-role.kubernetes.io/master-
# Expected output: node/<your-hostname> untainted
```
- Run command to check the result:
```yml
kubectl cluster-info
-------------------------------------------------------------------------------------------------------
kubectl get nodes -o wide
```
[Go to Calico quickstart website here!](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart)

## Multi nodes:
- Can use the command saved in step `init control plane/init master node` above.

```yml
sudo kubeadm join --token <token> <control-plane-host>:<control-plane-port> --discovery-token-ca-cert-hash sha256:<hash>
```

- If you forget the command `kubeadm join`, follow the step below to get token and hash code:
```yml
# Get token
kubeadm token list
-------------------------------------------------------------------------------------------------------
# Get hash code
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
openssl dgst -sha256 -hex | sed 's/^.* //'
```

- If the token expires (after 24 hours), let create new token:
```yml
kubeadm token create
```
