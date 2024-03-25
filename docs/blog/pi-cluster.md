# Raspberry Pi Kubernetes Cluster

This weeknd I built a Kubernetes cluster on 3 of my Rasphberry Pi 5's.

Here are the specs of the cluster:

* Raspbian OS 12
* MicroK8s
    * Metallb
    * Registry
* Containerd
* Nerdctl
    * Buildkit (dependency)
    * CNI Plugins (dependency)

## Getting Started

Ensure that each of the Pi's has a static IP address.
Ensure that SSH is enabled for them to make it easier to configure them remotely.

Do the following steps on each Pi:

1. Install snapd `sudo apt install snapd`
2. Reboot
3. Install snapd core `sudo snap install core`
4. Install MicroK8S `sudo snap install microk8s --classic`
5. Install containerd `sudo apt install -y containerd`
6. Install iptables `sudo apt install -y iptables`

## Prep MicroK8S

Pick a Pi to be your primary node (the control plane). On this Pi do the following:

1. `microk8s start`
2. `microk8s enable registry`
3. Optional: Add a bash alias to ~/.bash_prfoile `alias k="microk8s.kubectl"`

## Join the other two PI's to the cluster

Run `microk8s.add-node` on the primary node. Copy the comand under that ends with `--worker`.
Run this command on the other two Pi's.

## Allow MicroK8S Registry on Primary

Because the MicroK8S registry is insecure (runs HTTP) there neds to be special configuration to allow it's used.

Do the following steps on the primary node:

1. Get it's routable IP address. This is either `eth0` or `wlan0`.
2. Save the IP address to a variable and run the following script:

```
PRIMARY_IP="{REPLACE WITH REAL IP OF PRIMARY}"
sudo mkdir -p /var/snap/microk8s/current/args/certs.d/$PRIMARY_IP:32000
cat > hosts.toml <<-EOF
server = "http://$PRIMARY_IP:32000"
[host."http://$PRIMARY_IP:32000"]
capabilities = ["pull", "resolve"]
EOF
sudo mv hosts.toml /var/snap/microk8s/current/args/certs.d/$PRIMARY_IP:32000/hosts.toml
sudo chown root:root /var/snap/microk8s/current/args/certs.d/$PRIMARY_IP:32000/hosts.toml
microk8s stop
microk8s start
```

## Allow MicroK8S Registry on the other nodes

Run the same script as above ^^ minus the `microk8s` commands. They don't work on worker nodes.

## Setup MelaLB

In order to access services from your network we need to expose services to the external Interface.

Run the following on the primary node:

`microk8s enable metallb`

You'll be given an option to choose the IP address range. Enter `$PRIMARY_IP-$PRIMARY_IP`.

## Setup Containerd on Primary Node

Containerd needs the CNI plugins to enable networking. Run the following commands on the primary node:

1. wget https://github.com/containernetworking/plugins/releases/download/v1.4.0/cni-plugins-linux-arm64-v1.4.0.tgz
2. sudo mkdir /opt/cni
3. tar zxvf cni-plugins-linux-arm64-v1.4.0.tgz
4. sudo mv bin /opt/cni/

## Setup nerdctl on Primary Node

`nerdctl` is a replacement of the Docker CLI that interacts with containerd instead of the Docker daemon.

1. wget https://github.com/containerd/nerdctl/releases/download/v1.7.5/nerdctl-1.7.5-linux-arm64.tar.gz
2. tar zxvf nerdctl-1.7.5-linux-arm64.tar.gz 
3. mv nerdctl /usr/local/bin

## Setup builkit on Primary Node

In order to build images with `nerdctl` you need `buildkit`. Run the  following commands on the Primary node:

1. wget https://github.com/moby/buildkit/releases/download/v0.13.1/buildkit-v0.13.1.linux-arm64.tar.gz
2. tar -zxvf buildkit-v0.13.1.linux-arm64.tar.gz
3. sudo mv bin/* /usr/local/bin/
4. wget https://raw.githubusercontent.com/moby/buildkit/master/examples/systemd/system/buildkit.socket
5. wget https://raw.githubusercontent.com/moby/buildkit/master/examples/systemd/system/buildkit.service
6. sudo mv buildkit.s* /etc/systemd/system/
7. sudo systemctl enable buildkit.socket
8. sudo systemctl start buildkit

## Build an image

Let's build an image that we can use on our cluster. Run the following comamnds on the Primary node:

```
cat > index.html <<-EOF
<h1>Test Page</h1>
EOF

cat > Dockerfile <<-EOF
FROM nginx
COPY index.html /usr/share/nginx/html
EOF

nerdctl build -t testpage .
```

This creates an nginx image with a custom HTML page that we can see when we're done! 

## Push the image to the registry

Now that we have an image, we need to push it to the registry so we can use it in our cluster.

Run the following comamnds on the Primary node:

PRIMARY_IP="{REPLACE WITH REAL IP OF PRIMARY}"
nerdctl tag testpage $PRIMARY_IP:32000/testpage:registry
nerdctl push $PRIMARY_IP:32000/testpage:registry --insecure-registry

## Creat a deployment and service with the image

Let's use our newly created and pushed image. Run the following comamnds on the Primary node:

```
PRIMARY_IP="{REPLACE WITH REAL IP OF PRIMARY}"
cat > deployment.yaml <<-EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: testpage
  labels:
    app: testpage
spec:
  selector:
    matchLabels:
      app: testpage
  template:
    metadata:
      labels:
        app: testpage
    spec:
      containers:
      - name: nginx
        image: $PRIMARY_IP:32000/testpage:registry
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: testpage
spec:
  selector:
	app.kubernetes.io/name: testpage
  ports:
	- protocol: TCP
  	port: 80
  	targetPort: 80
  type: LoadBalancer
EOF
```

Now you can access the service anywhere on your network by using http://{PRIMARY_IP}.

Replace PRIMARY_IP with the IP of your primary node.




