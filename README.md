# Kubernetes

After installing (refering install_docker.sh, install_kubernetes_tools.sh), we will bootstrap the cluster on the Kube master node. Then, we will join each of the two worker nodes to the cluster, forming an actual multi-node Kubernetes cluster.

Here are the commands used:

On the Kube master node, initialize the cluster:

`sudo kubeadm init --pod-network-cidr=10.244.0.0/16`

That command may take a few minutes to complete. When it is done, set up the local kubeconfig:

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Verify that the cluster is responsive and that Kubectl is working:

`kubectl version`

You should get Server Version as well as Client Version. It should look something like this:

```
Client Version: version.Info{Major:"1", Minor:"12", GitVersion:"v1.12.2", GitCommit:"17c77c7898218073f14c8d573582e8d2313dc740", GitTreeState:"clean", BuildDate:"2018-10-24T06:54:59Z", GoVersion:"go1.10.4", Compiler:"gc", Platform:"linux/amd64"}

Server Version: version.Info{Major:"1", Minor:"12", GitVersion:"v1.12.2", GitCommit:"17c77c7898218073f14c8d573582e8d2313dc740", GitTreeState:"clean", BuildDate:"2018-10-24T06:43:59Z", GoVersion:"go1.10.4", Compiler:"gc", Platform:"linux/amd64"}
```

The `kubeadm init` command should output a `kubeadm join` command containing a token and hash. Copy that command and run it with sudo on both worker nodes. It should look something like this:

`sudo kubeadm join $some_ip:6443 --token $some_token --discovery-token-ca-cert-hash $some_hash`

Verify that all nodes have successfully joined the cluster:

`kubectl get nodes`

Once the Kubernetes cluster is set up, we still need to configure cluster networking in order to make the cluster fully functional. We went through the process of configuring a cluster network using Flannel. 

You can find more information on Flannel at the official site: https://coreos.com/flannel/docs/latest/.

Here are the commands used:

On all three nodes, run the following:

`echo "net.bridge.bridge-nf-call-iptables=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p`

Install Flannel in the cluster by running this only on the Master node:

```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/bc79dd1505b0c8681ece4de4c0d86c5cd2643275/Documentation/kube-flannel.yml
```

Verify that all the nodes now have a STATUS of Ready:

`kubectl get nodes`

You should see all three of your servers listed, and all should have a STATUS of Ready. Get more information about a specific node:

`kubectl describe node $node_name`

Note: It may take a few moments for all nodes to enter the Ready status, so if they are not all Ready, wait a few moments and try again.

It is also a good idea to verify that the Flannel pods are up and running. Run this command to get a list of system pods:

`kubectl get pods -n kube-system`

You should have three pods with flannel in the name, and all three should have a status of Running.

In order to run and manage containers with Kubernetes, you will need to use pods.

Here are the commands used:

Create a simple pod running an nginx container:

```
cat << EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
name: nginx
spec:
containers:
- name: nginx
image: nginx
EOF
```

Get a list of pods and verify that your new nginx pod is in the Running state:

`kubectl get pods`

Other useful commands: `kubectl get pods -n kube-system`

Get more information about your nginx pod:

`kubectl describe pod nginx`

Delete the pod:

`kubectl delete pod nginx`


# Networking in Kubernetes
The network functionality by contacting one pod from another pod over the virtual network. Create a deployment with two nginx pods:

```
cat << EOF | kubectl create -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
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
        image: nginx:1.15.4
        ports:
        - containerPort: 80
EOF
```

Create a busybox pod to use for testing:

```
cat << EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: busybox
spec:
  containers:
  - name: busybox
    image: radial/busyboxplus:curl
    args:
    - sleep
    - "1000"
EOF
```

Get the IP addresses of your pods:

`kubectl get pods -o wide`

Get the IP address of one of the nginx pods, then contact that nginx pod from the busybox pod using the nginx pod's IP address:

`kubectl exec busybox -- curl $nginx_pod_ip`

# Deployments 

Deployments are an important tool if you want to take full advantage of the automation capabilities provided by Kubernetes.
Here are the commands used:

Create a deployment:

```
cat <<EOF | kubectl create -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
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
        image: nginx:1.15.4
        ports:
        - containerPort: 80
EOF
```

Get a list of deployments:

`kubectl get deployments`

Get more information about a deployment:

`kubectl describe deployment nginx-deployment`

Get a list of pods:

`kubectl get pods`

You should see two pods created by the deployment. Delete one and check out the pods again to see how the state is maintained by deployments. 

# Services

While deployments provide a great way to automate the management of your pods, you need a way to easily communicate with the dynamic set of replicas managed by a deployment. This is where services come in. 

Here are the commands used:

Create a NodePort service on top of your nginx pods:

```
cat << EOF | kubectl create -f -
kind: Service
apiVersion: v1
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30080
  type: NodePort
EOF
```

Get a list of services in the cluster.

`kubectl get svc` or `kubectl get service`

You should see your service called nginx-service. Since this is a NodePort service, you should be able to access it using port 30080 on any of your cluster's servers. You can test this with the command:

`curl localhost:30080`

You should get an HTML response from nginx!

