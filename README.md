# K8S-QNRG

K8S was already installed on the devices. however I needed to create a cluster and connect the devices with one another. for this purpose I used kubeadm.

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
http://pwittrock.github.io/docs/setup/independent/create-cluster-kubeadm/
https://www.digitalocean.com/community/tutorials/how-to-create-a-kubernetes-cluster-using-kubeadm-on-ubuntu-18-04


After initializing kubeadm I got this:

```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.19.55.10:6443 --token qag3j1.6aqekoofrwuqmks7 \
    --discovery-token-ca-cert-hash sha256:21c73f2eabef352e94a10fa72e7912de3962089feaeb6b91262f4c4ec9cddfeb `

```
which needs to be executed on the worker nodes.
I got an error saying port 10250 is in use. after inspection I had to kill the process (tcp6) which was using it using:

`netstat -lnp | grep 1025`

I later found out that K3S was running on Rodney and Elizabeth and I had to do ./etc/local/bin/k3s-**-uninstall.sh to remove K3s.

Another incompatibility which avoided workers to connect to the master was that Docker belonged to a cgroup other than the systemd. It was solved using:

```
sudo mkdir /etc/docker
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```
