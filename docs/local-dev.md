# A very simple way to play with the template

This is a short guide demonstrating how to develop with the hana template on e.g. ansible roles without the need to have a full OpenShift cluster with Openshift Virtualization installed. It uses the kubevirt development environment.

First check out kubevirt

```bash
git clone https://github.com/kubevirt/kubevirt.git
cd kubevirt
```

Second, configure the cluster you want to start:

```bash
export KUBEVIRT_PROVIDER=k8s-1.21
export KUBEVIRT_NUM_NODES=2 # the second node will have cpumanager with static policy enabled
export KUBEVIRT_MEMORY_SIZE=9216M
```

Then start the cluster:

```bash
make cluster-up
```

You can now access the clusteer:

```bash
cluster-up/kubectl.sh get pods
```

You can also export the KUBECONFIG to use the cluster with your local installed tools:

```bash
export KUBECONFIG=$PWD/_ci-configs/k8s-1.21/.kubeconfig
oc get pods --all-namespaces
kubectl get pods --all-namespaces
```

Now install the latest developer build of kubevirt which should always work with the template (https://kubevirt.io/user-guide/operations/installation/#installing-the-daily-developer-builds):

```bash
LATEST=$(curl -L https://storage.googleapis.com/kubevirt-prow/devel/nightly/release/kubevirt/kubevirt/latest)
kubectl apply -f https://storage.googleapis.com/kubevirt-prow/devel/nightly/release/kubevirt/kubevirt/${LATEST}/kubevirt-operator.yaml
kubectl apply -f https://storage.googleapis.com/kubevirt-prow/devel/nightly/release/kubevirt/kubevirt/${LATEST}/kubevirt-cr.yaml
kubectl -n kubevirt wait kv kubevirt --for condition=Available --timeout 15m # speed depens on container pull speed
```

Now render the template locally and deploy it to the cluster:

```bash
oc process --local -f https://raw.githubusercontent.com/rmohr/hana-template/main/hana-template.yaml NAME=testvm | oc apply -f -
```

If the VM does not come up, look at the `describe` output:

```bash
oc describe vmi testvm
```

The SRIOV network defined in the template will not be available in the cluster and must be removed from the definition.

For executing ansible roles inside the guest OS, consider creating a headless service:

```bash
oc create -f https://raw.githubusercontent.com/rmohr/hana-template/main/headless.yaml
```

Then follow the guide in https://github.com/rmohr/jumppod to get easy access
for ansible without having to take care of connection details.

Here a quick-start for deploying the service:

```bash
mkdir -p ~/etc/ssh && ssh-keygen -A -f ~/
kubectl create secret generic host-keys --from-file=${HOME}/etc/ssh
rm -rf ~/etc/ssh
kubectl create -f https://raw.githubusercontent.com/rmohr/jumppod/main/manifests/deployment.yaml
```

Allow your ssh key access and forward a port:

```bash
kubectl create configmap authorized-keys --from-file=${HOME}/.ssh/id_rsa.pub
kubectl port-forward svc/sshd 2222:22 &
```


Define the jumphost in you `.ssh/config` for convenience:

```
Host jumphost
   HostName localhost
   User nonroot
   Port 2222
```

Connect to the VMI via the in-cluster DNS name:


```bash
ssh fedora@testvm.ansible -J jumphost
```
