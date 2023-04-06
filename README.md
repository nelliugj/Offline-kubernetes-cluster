# Offline-kubernetes-cluster
Setting up a kubernetes cluster on an offline server using containerd, calico, linkerd and helm charts.
# KUBEADM OFFLINE SETUP:

## Containerd, Kubernetes, Calico, Linkerd, Helm, Nginx

This project describes the procedure to deploy a fully operational kubernetes cluster on an offline server.
> Note: For this scenario, 2 ubuntu 22.04 LTS servers were used, one connected to the internet to pull images and files, 
> and a second one without an active internet connection but able to communicate with the online server for pulling images locally.
> offline machine: 10.0.0.116
> online machine: 10.0.0.121

## Prerequisites for installing Kubernetes

The following requirements are needed for a successful Kubernetes deployment:

* swap off
* ip_tables, br_netfilter, overlay modules added to /etc/modules-load.d/k8s.conf file
* ufw firewall disabled
* apparmor disabled
* net.bridge.bridge-nf-call-ip6tables = 1, net.bridge.bridge-nf-call-iptables = 1, net.ipv4.ip_forward = 1 added to sysctl.d/k8s.conf file


## Containerd and Docker Installation: 

If you can’t use Docker’s apt repository to install Docker Engine, you can download the deb file for your release and install it manually. You need to download a new file each time you want to upgrade Docker Engine.

### Online machine:

Go to [Docker Install Ubuntu](https://download.docker.com/linux/ubuntu/dists/)

Select your Ubuntu version from the list, go to pool/stable/ and select the applicable architecture (amd64, armhf, arm64, or s390x).

Download the following deb files for the Docker Engine, CLI, containerd, and Docker Compose packages:

* containerd.io_<version>_<arch>.deb
* docker-ce_<version>_<arch>.deb
* docker-ce-cli_<version>_<arch>.deb
* docker-buildx-plugin_<version>_<arch>.deb
* docker-ce-rootless-extras_<version>_<arch>.deb
* docker-compose-plugin_<version>_<arch>.deb
* docker-scan-plugin_<version>_<arch>.deb

These are the specific versions used for this test:

* containerd.io_1.6.19-1_amd64.deb
* docker-ce_23.0.1-1_ubuntu.22.04_jammy_amd64.deb
* docker-ce-cli_23.0.1-1_ubuntu.22.04_jammy_amd64.deb
* docker-buildx-plugin_0.10.2-1_ubuntu.22.04_jammy_amd64.deb
* docker-ce-rootless-extras_23.0.1-1_ubuntu.22.04_jammy_amd64.deb
* docker-compose-plugin_2.16.0-1_ubuntu.22.04_jammy_amd64.deb
* docker-scan-plugin_0.23.0_ubuntu-jammy_amd64.deb


Copy the package files to offline machine, for example using scp:
```shell
scp *.deb admin@10.0.0.116:/home/admin/docker-deb/
```

### Offline machine:

Install containerd .deb package. Update the path in the following example to where you saved the package files.
```shell
sudo dpkg -i ./containerd.io_1.6.19-1_amd64.deb 
```

then configure containerd to use SystemdCgroup,
```shell
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

Containerd is now configured and ready, the rest of the Docker deb packages can be installed if required, for deploying other docker containers external to the kubernetes cluster:

Install docker components on offline machine:
```shell
sudo dpkg -i  ./docker-ce_<version>_<arch>.deb \
  ./docker-ce-cli_<version>_<arch>.deb \
  ./docker-buildx-plugin_<version>_<arch>.deb \
  ./docker-compose-plugin_<version>_<arch>.deb
sudo usermod -aG docker $USER
newgrp docker
sudo systemctl start docker
sudo systemctl enable docker
```

## Kubernetes Installation:

### Online machine:

Add kubernetes repository:
```shell
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
sudo apt update
```
Download kubernetes deb packages to online machine:
```shell
apt-get download kubeadm kubelet kubectl kubernetes-cni cri-tools conntrack ebtables socat
```
copy the .deb files to offline machine.

### Offline machine

Install kubernetes deb packages on offline machine:
```shell
sudo dpkg -i conntrack_1%3a1.4.6-2build2_amd64.deb
sudo dpkg -i cri-tools_1.26.0-00_amd64.deb
sudo dpkg -i ebtables_2.0.11-4build2_amd64.deb
sudo dpkg -i socat_1.7.4.1-3ubuntu4_amd64.deb
sudo dpkg -i kubernetes-cni_1.2.0-00.deb
sudo dpkg -i kubelet_1.26.3-00_amd64.deb
sudo dpkg -i kubectl_1.26.3-00_amd64.deb
sudo dpkg -i kubeadm_1.26.3-00_amd64.deb
```

### Online machine:
Get a list of images required for kubadmn installation:
```shell
kubeadm config images list
```

example output:

* registry.k8s.io/kube-apiserver:v1.26.3
* registry.k8s.io/kube-controller-manager:v1.26.3
* registry.k8s.io/kube-scheduler:v1.26.3
* registry.k8s.io/kube-proxy:v1.26.3
* registry.k8s.io/pause:3.9
* registry.k8s.io/etcd:3.5.6-0
* registry.k8s.io/coredns/coredns:v1.9.3

Pull each one of these images on online machine and save them as tar files, example:
```shell
$docker pull registry.k8s.io/kube-apiserver:v1.26.3
$docker save registry.k8s.io/kube-apiserver:v1.26.3 > kube-apiserver_v1.26.3.tar
.
.
.
```
Additionally, pull and save as tar the following image, too. It is required for kubelet to initialize properly
```shell
docker pull registry.k8s.io/pause:3.6
```

Copy the 8 tar files to the offline machine.

### Offline machine:

Import the images into containerd k8s.io namespace, using the following command for each of the images:

```shell
sudo ctr -n k8s.io images import etcd_3.5.6-0.tar
```
Import for the other 7 tar files (apiserver,coredns,scheduler,controller,pause_3.9 and pause_3.6)

Confirm they are successfully loaded:
```shell
sudo ctr -n k8s.io images ls
```

### Initialize control plane, define the apiserver address, and kubernetes version to be used, in the example below we are using the latest available kubernetes version at the time this docuement was created:
```shell
sudo kubeadm init --apiserver-advertise-address=192.168.56.103 --kubernetes-version=v1.26.3
```
Allow non-root user access to kubectl:
```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Untaint node to be able to run pods on it:
```shell
kubectl taint node --all node-role.kubernetes.io/control-plane-
kubectl taing node --all node-role.kubernetes.io/master-
```

## Calico Installation:

### Online machine:
Download calico's manifest, v3.25.0 is the latest version available when this document was created:
```shell
curl https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml > calico.yaml
```
Open the yaml file, find cni image version, pull it, and save it, in this example v3.25.0 is used:
```shell
docker pull docker.io/calico/cni:v3.25.0
docker save  docker.io/calico/cni:v3.25.0 > cni_v3.25.0.tar
```
Do the same for calico kube-controllers and node images:
```shell
docker pull docker.io/calico/kube-controllers:v3.25.0
docker pull docker.io/calico/node:v3.25.0
docker save docker.io/calico/kube-controllers:v3.25.0 > kube-controllers_v3.25.0.tar
docker save docker.io/calico/node:v3.25.0 > node_v3.25.0.tar
```
Copy these 3 tar files to offline machine (cni, kube-controllers and node), and the manifest too: calico.yaml

### Offline machine:

Import calico images to containerd k8s.io namespace:

```shell
sudo ctr -n k8s.io images import kube-controllers_v3.25.0.tar node_v3.25.0.tar
sudo ctr -n k8s.io images import node_v3.25.0.tar
sudo ctr -n k8s.io images ls | grep calico
```

Install calico:
```shell
kubectl apply -f calico.yaml
```

## Linkerd v2.12.4

### Online machine:

Pull linkerd images:
```shell
docker pull cr.l5d.io/linkerd/controller:stable-2.12.4
docker pull cr.l5d.io/linkerd/metrics-api:stable-2.12.4
docker pull cr.l5d.io/linkerd/policy-controller:stable-2.12.4
docker pull cr.l5d.io/linkerd/proxy-init:v2.0.0
docker pull cr.l5d.io/linkerd/proxy:stable-2.12.4
docker pull cr.l5d.io/linkerd/tap:stable-2.12.4
docker pull cr.l5d.io/linkerd/web:stable-2.12.4
docker pull docker.io/prom/prometheus:v2.30.3
```
Save each of the 8 images to a tar file:
```shell
docker save cr.l5d.io/linkerd/controller:stable-2.12.4 > controller_stable-2.14.2.tar
docker save cr.l5d.io/linkerd/metrics-api:stable-2.12.4 > metrics-api_stable-2.12.4.tar
.
.
```

Copy the 8 tar files to offline machine ....
Then, load each of these 8 images to containerd k8s.io namespace:
```shell
$ sudo ctr -n k8s.io images import controller_stable-2.14.2.tar
.
.
.
```

### Online machine:
download CLI binary file from this URL, (...don't forget to validate sha256 checksum)

https://github.com/linkerd/linkerd2/releases/download/stable-2.12.4/linkerd2-cli-stable-2.12.4-linux-amd64

download Linkerd installer:
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install > linkerd_cli_installer

Copy binary and installer script to offline machine.

Linkerd installer script was edited to work without checksum validations, urls, saved it as:
linkerd_cli_installer_offline_JG.  changes shown below:

### linkerd installer script for offline server:
> some lines removed from original file, 
> IMPORTANT, dstfile must be set to linkerd2-cli-stable-2.12.4-linux-amd64 file, for example:
> dstfile="/home/sysadmin/containerd-k8s-offline/linkerd/linkerd2-cli-stable-2.12.4-linux-amd64"

```shell
#!/bin/sh

set -eu

LINKERD2_VERSION=${LINKERD2_VERSION:-stable-2.12.4}
INSTALLROOT=${INSTALLROOT:-"${HOME}/.linkerd2"}

happyexit() {
  echo ""
  echo "Add the linkerd CLI to your path with:"
  echo ""
  echo "  export PATH=\$PATH:${INSTALLROOT}/bin"
  echo ""
  echo "Now run:"
  echo ""
  echo "  linkerd check --pre                     # validate that Linkerd can be installed"
  echo "  linkerd install --crds | kubectl apply -f - # install the Linkerd CRDs"
  echo "  linkerd install | kubectl apply -f -    # install the control plane into the 'linkerd' namespace"
  echo "  linkerd check                           # validate everything worked!"
  echo ""
  echo "You can also obtain observability features by installing the viz extension:"
  echo ""
  echo "  linkerd viz install | kubectl apply -f -  # install the viz extension into the 'linkerd-viz' namespace"
  echo "  linkerd viz check                         # validate the extension works!"
  echo "  linkerd viz dashboard                     # launch the dashboard"
  echo ""
  echo "Looking for more? Visit https://linkerd.io/2/tasks"
  echo ""
  exit 0
}

OS=$(uname -s)
arch=$(uname -m)
cli_arch=""
case $OS in
  CYGWIN* | MINGW64*)
    OS=windows.exe
    ;;
  Darwin)
    case $arch in
      x86_64)
        cli_arch=""
        ;;
      arm64)
        cli_arch=$arch
        ;;
      *)
        echo "There is no linkerd $OS support for $arch. Please open an issue with your platform details."
        exit 1
        ;;
    esac
    ;;
  Linux)
    case $arch in
      x86_64)
        cli_arch=amd64
        ;;
      armv8*)
        cli_arch=arm64
        ;;
      aarch64*)
        cli_arch=arm64
        ;;
      armv*)
        cli_arch=arm
        ;;
      amd64|arm64)
        cli_arch=$arch
        ;;
      *)
        echo "There is no linkerd $OS support for $arch. Please open an issue with your platform details."
        exit 1
        ;;
    esac
    ;;
  *)
    echo "There is no linkerd support for $OS/$arch. Please open an issue with your platform details."
    exit 1
    ;;
esac

dstfile="/home/sysadmin/containerd-k8s-offline/linkerd/linkerd2-cli-stable-2.12.4-linux-amd64"

(
  mkdir -p "${INSTALLROOT}/bin"
  chmod +x "${dstfile}"
  rm -f "${INSTALLROOT}/bin/linkerd"
  ln -s "${dstfile}" "${INSTALLROOT}/bin/linkerd"
)

# rm -r "$tmpdir"
echo "Linkerd ${LINKERD2_VERSION} was successfully installed 🎉"
echo ""
happyexit
```


With those changes in place, add executable permissions to the linkerd installer:
```shell
chmod +x linkerd_cli_installer_offline_JG
```
And then, install linkerd_cli:
```shell
./linkerd_cli_installer_offline_JG
```

finally, follow Linkerd instructions to complete installation/configuration:
```shell
linkerd check --pre                     
linkerd install --crds | kubectl apply -f - 
linkerd install --set proxyInit.runAsRoot=true | kubectl apply -f -    
linkerd check                           
linkerd viz install | kubectl apply -f - 
linkerd viz check                         
```

Linkerd is now installed and ready, since there is no internet connection, it is normal to see up to 7 linkerd heartbeat pods showing errors.
These pods are created to reach an https linkerd server to validate if a new version of Linkerd is available, but it does not affect the overall operation of Linkerd and the cluster.

## Helm 

### Online machine:

Download Helm binary file from github,

```shell
curl -O https://get.helm.sh/helm-v3.11.2-linux-amd64.tar.gz
```
Helm will need to be installed on online machine to fetch nginx helm charts:
de-compress tar file:
```shell
tar -zxvf helm-v3.11.2-linux-amd64.tar.gz
``` 
move helm binary to /usr/local/bin
```shell
sudo mv linux-amd64/helm /usr/local/bin/helm
```
Helm is now installed on online machine.

Copy tar file to offline machine, and repeat the same process to install helm on offline machine:

### Offline machine:
de-compress tar file:
```shell
tar -zxvf helm-v3.11.2-linux-amd64.tar.gz
```
move helm binary to /usr/local/bin
```shell
sudo mv linux-amd64/helm /usr/local/bin/helm
```

## Docker Private Registry setup 

A private docker registry needs to be configured to store and install NGINX and other applications via helm charts 
In this sample configuration, the private registry was setup on the online machine, and it was assigned the http://my-docker-registry.com:5000 endpoint.

[Deploy a registry server](https://docs.docker.com/registry/deploying/)

### Online machine:
Make sure docker is installed on online machine.
Deploy a registry server:
The following command creates a registry container listening on port 5000:
```shell
docker run -d -p 5000:5000 --restart=always --name registry registry:2
```

in this sample configuration an insecure/plain HTTP registry was configured by following docker's instructions.

[Run an insecure registry](https://docs.docker.com/registry/insecure/)

Step 1, the following lines must be added to the /etc/docker/daemon.json file, if it does not exist it must be created:
```shell
 {
  "insecure-registries" : ["http://my-docker-registry.com:5000"]
 }
```
Step 2, save the file and restart docker:
```shell
sudo systemctl restart docker
```

Step 3, edit the /etc/hosts file to include an entry for the private docker registry, for example:
```shell
10.0.0.121  my-docker-registry.com
```

### Offline machine:

Steps 1, 2, and 3 must be performed on the offline machine, too.
After this the offline machine should be able to access the private registry, this can be confirmed with the following command:

curl my-docker-registry.com:5000/v2/_catalog


## Setup Containerd to use the Private Docker Registry

Containerd by default uses docker.io as its main registry. To allow access to a private docker registry, the private registry must be
defined as a registry mirror and its http url must be configured as its endpoint:

The example below shows the changes made to the /etc/containerd/config.toml: (line 154 was added)

```shell
.
.
.
.
144     [plugins."io.containerd.grpc.v1.cri".registry]
145       config_path = ""
146
147       [plugins."io.containerd.grpc.v1.cri".registry.auths]
148
149       [plugins."io.containerd.grpc.v1.cri".registry.configs]
150
151       [plugins."io.containerd.grpc.v1.cri".registry.headers]
152
153       [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
154         [plugins."io.containerd.grpc.v1.cri".registry.mirrors."my-docker-registry.com:5000"] endpoint = ["http://my-docker-registry.com:5000", ]
.
.
.
```

Save the file and restart containerd:
```shell
sudo systemctl restart containerd
```

## Nginx - Helm

Online machine:
add nginx repo to helm:
```shell
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```
pull/fetch ingress helm charts decompressed:
```shell
helm fetch ingress-nginx/ingress-nginx --untar
```
Copy the decompressed helm charts folder "ingress-nginx" to the offline machine.

Pull docker images for Nginx on online machine, too:
```shell
docker pull registry.k8s.io/ingress-nginx/controller:v1.7.0
docker pull registry.k8s.io/ingress-nginx/kube-webhook-certgen:v20230312-helm-chart-4.5.2-28-g66a760794
```

Tag them, and push them to the private registry (in the example below, the original versioning was kept):
```shell
docker tag registry.k8s.io/ingress-nginx/controller:v1.7.0 my-docker-registry.com:5000/controller:v1.7.0
docker push my-docker-registry.com:5000/controller:v1.7.0
docker tag registry.k8s.io/ingress-nginx/kube-webhook-certgen:v20230312-helm-chart-4.5.2-28-g66a760794 my-docker-registry.com:5000/kube-webhook-certgen:v20230312-helm-chart-4.5.2-28-g66a76079
docker push my-docker-registry.com:5000/kube-webhook-certgen:v20230312-helm-chart-4.5.2-28-g66a76079
```

Delete the images from the online server registry, to save resources, both originals and tagged ones: 
```shell
$ docker image rm registry.k8s.io/ingress-nginx/kube-webhook-certgen:v20230312-helm-chart-4.5.2-28-g66a760794
$ docker image rm my-docker-registry.com:5000/kube-webhook-certgen:v20230312-helm-chart-4.5.2-28-g66a76079
$ docker image rm registry.k8s.io/ingress-nginx/controller:v1.7.0
$ docker image rm my-docker-registry.com:5000/controller:v1.7.0
```

confirm the images are now in the private registry:
```shell
curl my-docker-registry.com:5000/v2/_catalog
```
the output should look something like this, which means the controller and kube-webhook-certgen images are now in the private registry.:
```shell
{"repositories":["controller","kube-webhook-certgen"]}
```

Confirm the tags added to these images using the following command:
```shell
curl my-docker-registry.com:5000/v2/controller/tags/list
```
output:
```shell
{"name":"controller","tags":["v1.7.0"]}
```
Same thing for the kube-webhook-certgen:
```shell
$ curl my-docker-registry.com:5000/v2/kube-webhook-certgen/tags/list
```
output:
```shell
{"name":"kube-webhook-certgen","tags":["v20230312-helm-chart-4.5.2-28-g66a760794"]}
```

Edit yaml files to include the private docker registry configured in the previous steps instead of the default registry: 
From the de-compressed folder that was copied to offline machine "ingress-nginx" two files must be edited for Nginx to work:

* ingress-nginx/values.yaml
* ingress-nginx/templates/_helpers.tpl

Edit the values.yaml file to make sure the right image, repository, tag and digest are defined for both the controller, and the kube-webhook-certgen containers. Also, set "HostNetwork = true"
_helpers.tpl must be edited to avoid validating the ingress-nginx controller's minimum release version, which requires access to internet.


The following are examples of the values.yaml file and _helpers.tpl as they were edited to initialize the ingress-nginx controller

#### values.yaml:
```shell
controller:
  name: controller
  image:
    ## Keep false as default for now!
    chroot: false
    ##registry: registry.k8s.io
    ##image: ingress-nginx/controller
    ## for backwards compatibility consider setting the full image url via the repository value below
    ## use *either* current default registry/image or repository format or installing chart by providing the values.yaml will fail
    repository: "my-docker-registry.com:5000/controller"
    tag: "v.1.7.0"
    #digest: sha256:7612338342a1e7b8090bef78f2a04fffcadd548ccaabe8a47bf7758ff549a5f7
    digest: sha256:733b1e00bd6aeada7d29ccbc2eb3ff18b1e0e2ae0c6ffe5c5b435ffaef0fc9dc
    digestChroot: sha256:e84ef3b44c8efeefd8b0aa08770a886bfea1f04c53b61b4ba9a7204e9f1a7edc
    pullPolicy: IfNotPresent
    # www-data -> uid 101
    runAsUser: 101

.
.
.
.

   patchWebhookJob:
      securityContext:
        allowPrivilegeEscalation: false
      resources: {}
    patch:
      enabled: true
      image:
        ##registry: registry.k8s.io
        ##image: ingress-nginx/kube-webhook-certgen
        ## for backwards compatibility consider setting the full image url via the repository value below
        ## use *either* current default registry/image or repository format or installing chart by providing the values.yaml will fail
        repository: "my-docker-registry.com:5000/kube-webhook-certgen"
        tag: "v20230312-helm-chart-4.5.2-28-g66a760794"
        #tag: "latest"
        #digest: sha256:01d181618f270f2a96c04006f33b2699ad3ccb02da48d0f89b22abce084b292f
        digest: sha256:22bf4e148e63d2aabb047281aa60f1a5c1ddfce73361907353e660330aaf441a
        pullPolicy: IfNotPresent
.
.
.
.


  hostNetwork: true
  ## Use host ports 80 and 443
  ## Disabled by default
```


####_helpers.tpl, the following lines were commented out:
```shell
#{{/*
#Check the ingress controller version tag is at most three versions behind the last release
#*/}}
#{{- define "isControllerTagValid" -}}
#{{- if not (semverCompare ">=0.27.0-0" .Values.controller.image.tag) -}}
#{{- fail "Controller container image tag should be 0.27.0 or higher" -}}
#{{- end -}}
#{{- end -}}
```

Create an ingress-nginx namespace:
```shell
kubectl create ns ingress-nginx
```

Then install the ingres-nginx helm charts from a directory above the ingress-nginx folder that was copied from online machine:

```shell
helm install ingress-nginx ingress-nginx -n ingress-nginx -f ingress-nginx/values.yaml
```
The nginx controller should be up and running now.

At this point,the cluster is fully functional, and other applications can be installed via helm by following the same procedure as it was followed for nginx:

* Images must be added to the private repository
* value.yaml file, (and perhaps other files, depending on the application) must be configured to use the private registry.
* helm charts must be installed from the local helm charts folder, instead of the helm repo.








