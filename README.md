Prerequisites:
-------------
1.> Should have access to Google Cloud, sign up for free tier.
2.> GCloud SDK should be installed and configured on the local machine (laptop/desktop).


Steps:
------
1.> Configure network in Google cloud
   ```
    $gcloud compute networks create k8s-stackx --subnet-mode custom
    $gcloud compute networks subnets create k8s-subnet --network k8s-stackx --range 10.240.0.0/24
   ```
2.> Create filewall rules
```
#gcloud compute firewall-rules create k8s-stackx-allow-internal --allow tcp,udp,icmp --network k8s-stackx --source-ranges 10.240.0.0/24,10.200.0.0/16
#gcloud compute firewall-rules create k8s-stackx-allow-external --allow tcp:22,tcp:6443,icmp --network k8s-stackx --source-ranges 0.0.0.0/0
#gcloud compute firewall-rules list --filter="network:k8s-stackx"
```
3.> Allocate static IP

```
#gcloud compute addresses create k8s-stackx --region $(gcloud config get-value compute/region)
#gcloud compute addresses list --filter="name=('k8s-stackx')"
```

4.> List down the public static IP and take node of it.

```
[abhinit@centos7 ~]$ gcloud compute addresses list --filter="name=('k8s-stackx')"
NAME        ADDRESS/RANGE  TYPE      PURPOSE  NETWORK  REGION       SUBNET  STATUS
k8s-stackx  34.93.7.80     EXTERNAL                    asia-south1          RESERVED
[abhinit@centos7 ~]$
```

5.> Configure compute instances

```
Configure Master Servers
for i in 0 1 2; do
  gcloud compute instances create c1-master-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-1604-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --private-network-ip 10.240.0.1${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet k8s-subnet \
    --tags k8s-stackx,master
done
```

```
Configure worker nodes
for i in 0 1 2; do
  gcloud compute instances create c1-node-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-1604-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --metadata pod-cidr=10.200.${i}.0/24 \
    --private-network-ip 10.240.0.2${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet k8s-subnet \
    --tags k8s-stackx,node
done
```

6.> List the configure compute instances

```
[abhinit@centos7 ~]$ gcloud compute instances list
NAME         ZONE           MACHINE_TYPE               PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP    STATUS

----Master Nodes----
c1-master-0  asia-south1-c  custom (2 vCPU, 3.75 GiB)               10.240.0.10  34.93.237.66   RUNNING
c1-master-1  asia-south1-c  custom (2 vCPU, 3.75 GiB)               10.240.0.11  34.93.121.152  RUNNING
c1-master-2  asia-south1-c  custom (2 vCPU, 3.75 GiB)               10.240.0.12  34.93.111.149  RUNNING
----Worker nodes----
c1-node-0    asia-south1-c  n1-standard-1                           10.240.0.20  34.93.196.53   RUNNING
c1-node-1    asia-south1-c  n1-standard-1                           10.240.0.21  34.93.51.166   RUNNING
c1-node-2    asia-south1-c  n1-standard-1                           10.240.0.22  34.93.104.131  RUNNING
----Tool server, jenkins, helm, ansible installed on this node----
srv-tool     asia-south1-c  n1-standard-1                           10.240.0.3   34.93.27.193   RUNNING
[abhinit@centos7 ~]$
```

6.> Configure the loadbalancer for kubernetes API service
###Get the static public IP, allocated before
```
KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe k8s-stackx --region $(gcloud config get-value compute/region) --format 'value(address)')
```
###Create health-check for LB
```
gcloud compute http-health-checks create kubernetes --description "Kubernetes Health Check" --host "kubernetes.default.svc.cluster.local" --request-path "/healthz"
```
###Create firewall rules for LB
```
gcloud compute firewall-rules create k8s-stackx-allow-health-check --network k8s-stackx --source-ranges 34.93.0.0/16,0.0.0.0/0 --allow tcp
```
###Create target pools
```
gcloud compute target-pools create kubernetes-target-pool --http-health-check kubernetes
```
###Include Master servers in the target pool
```
gcloud compute target-pools add-instances kubernetes-target-pool --instances c1-master-0,c1-master-1,c1-master-2
```
###Create forwarding rule from LB to master servers
```
gcloud compute forwarding-rules create kubernetes-forwarding-rule --address ${KUBERNETES_PUBLIC_ADDRESS} --ports 6443 --region $(gcloud config get-value compute/region) --target-pool kubernetes-target-pool
```

7.> Install Docker, Kubeadm and kubelet on all the master nodes
```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -


bash -c 'cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF'


apt-get update
apt-get install -y docker.io kubelet kubeadm kubectl
```

8.> Create file "kubeadm-config.yaml" on c1-master-0 node with below content

```
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: stable
controlPlaneEndpoint: "LOAD_BALANCER_DNS:LOAD_BALANCER_PORT"
```

9.> Initialize the control plane
```
kubeadm init --config=kubeadm-config.yaml --upload-certs
```
10.> Add other control plane server and workers and configure kubectl. Below snippet is after the kubeadm initialized on the first master node
```
<snip>
You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 34.93.7.80:6443 --token uyfdaf.7y3a0yt1579dxvgu \
    --discovery-token-ca-cert-hash sha256:73cf3302e0b11e0242ce4ff17587191b946f0019523747ee72a92436cebcc189 \
    --experimental-control-plane --certificate-key bd28c3d6e00d3cc724f9a2bd066e0e1c880e4bdd80b32892061245c753754524

Then you can join any number of worker nodes by running the following on each as root:

    kubeadm join 34.93.7.80:6443 --token uyfdaf.7y3a0yt1579dxvgu --discovery-token-ca-cert-hash sha256:73cf3302e0b11e0242ce4ff17587191b946f0019523747ee72a92436cebcc189
</snip>

<snip>
  To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
</snip>
```

11> Used network plugin "Calico" initially, steps are below, but that failed somehow and didn't have much time so went with weave:-
```
wget https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/rbac-kdd.yaml
wget https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml

kubectl apply -f rbac-kdd.yaml
kubectl apply -f calico.yaml
```
12.> Deleted calico pods and deployed weave
```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```
13.> Prepare Jenkins server on tools server
```
gcloud compute instances create srv-tool --async --boot-disk-size 200GB --can-ip-forward --image-family ubuntu-1604-lts --image-project ubuntu-os-cloud --machine-type n1-standard-1 \
--private-network-ip 10.240.0.3 --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring --subnet k8s-subnet --tags k8s-stackx,tools
```
14.> Install jenkins
```
apt-get install default-jre

wget -q -O - https://pkg.jenkins.io/debian/jenkins-ci.org.key | sudo apt-key add -
echo deb https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list


apt-get update
apt-get install jenkins

systemctl status jenkins
```

15.> Configure jenkins to use kubectl
```
su - jenkins
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
16.> Install and configure Helm
```
curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > get_helm.sh; chmod 700 get_helm.sh; ./get_helm.sh
```
17.> Configure helm
```
mkdir helm
cd helm
```
#Create Tiller service account, content for tiller SA, tiller-serviceaccount.yaml
```
kubectl apply -f service-account.yaml
```
#Create cluster account rolebinding, role-binding.yaml with below content
```
kubectl apply -f role-binding.yaml
```
#Deploy Tiller
```
helm init --service-account tiller --wait
```

18.> Install and configure ansible
```
apt-get install python-pip
pip install ansible
```
#Login as user Jenkins and create directory "ansible"

#Download of clone guestbook repo in ansible directory from "https://github.com/kubernetes/examples/tree/master/guestbook"

19.> Run ansible playbook to test the guestbook deployment, we'll use the same playbook for deploying Guestbook application via Jenkins
```
su - jenkins
cd ansible
ansible-playbook deploy-guestbook.yaml
```
20.> We'll use helm chart from git repo "https://github.com/helm/charts" to deploy Prometheus and Grafana:
```
#Clone the repository locally
git clone https://github.com/kubernetes/charts

cd ~/charts/stable/prometheus
```
#Edit values.yaml as per your needs, in production specify the PV and storageClass

#Deploy prometheus
```
helm install -f values.yaml stable/prometheus --name prometheus --namespace monitoring
```

21.> Now deploy Grafana
```
cd ~/charts/stable/prometheus
```
Edit values.yaml as per your needs, set admin password. In production specify the PV and storageClass
```
helm install -f values.yaml stable/grafana --name grafana --namespace monitoring
```

22.> Configure storageClass in the clsuter.

#Create file gce-pd-sc.yaml with below content:
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gold
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
  replication-type: none
```

#Run kubectl
```
kubectl apply -f gce-pd-sc.yaml
```
#Check storageClass
```
abhinit@c1-master-0:~$ kubectl get sc
NAME   PROVISIONER            AGE
gold   kubernetes.io/gce-pd   5s
abhinit@c1-master-0:~$
```
#Patch the storageClass to be default
```
kubectl patch storageclass slow -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```
23.> We'll need to perform some extra work, so that storage can be provisioned dynamically
```
a.) Create a ServiceAccount with admin access on compute resources.
##Follow documents on https://cloud.google.com/iam/docs/creating-managing-service-accounts#iam-service-accounts-create-console

b.) The Kubelet needs to run with the argument "--cloud-provider=gce".
For this the "KUBELET_KUBECONFIG_ARGS in /etc/systemd/system/kubelet.service.d/10-kubeadm.conf" have to be edited.
The Kubelet can then be restarted with "sudo systemctl restart kubelet"

c.) Edit file "/etc/kubernetes/cloud-config" and add below content
[Global]
project-id = "<google-project-id>"

d.) Kubeadm needs to have GCE configured as its cloud provider. For this create file gce.yaml as below:

#
root@c1-master-0:~# cat gce.yaml
apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
controllerManager:
  extraArgs:
    cloud-provider: gce
    cloud-config: /etc/kubernetes/cloud
  extraVolumes:
  - name: cloud-config
    hostPath: /etc/kubernetes/cloud-config
    mountPath: /etc/kubernetes/cloud
    pathType: FileOrCreate
root@c1-master-0:~#
#

##And run below commmand

sudo kubeadm upgrade apply --config gce.yml
```

24.> Setup elasticsearch, fluentd and kibana
```
a.) Create kube-logging namespaces
$vi kube-logging.yaml
kind: Namespace
apiVersion: v1
metadata:
  name: kube-logging

$kubectl create -f kube-logging.yaml

b.) Create headless service

$kubectl create -f elasticsearch_svc.yaml  

c.) Create StatefulSet
$kubectl create -f elasticsearch_statefulset.yaml

#Monitor the status of StatefulSet
$kubectl rollout status sts/es-cluster --namespace=kube-logging

d.) Rollout kibana dashboard

$kubectl create -f kibana.yaml

#Monitor the status of kibana Rollout
$kubectl rollout status deployment/kibana --namespace=kube-logging

e.) Create fluentd DaemonSet
$kubectl create -f fluentd.yaml


f.) Port forward to access Kibana dashboard
$kubectl port-forward kibana-67f95cc5f4-vv4kd 5601:5601 --namespace=kube-logging      

g.) Open in browser "http://<public_ip_of_node>:5601"
Note- Port 5601 needed to open at VPC

h.) Now add elasticsearch as the Data source in Kibana.
```
