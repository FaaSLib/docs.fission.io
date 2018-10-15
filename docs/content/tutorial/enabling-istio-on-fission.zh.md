---
title: "Enabling Istio on Fission"
draft: false
weight: 62
---

## Test Environment

* Google kubernetes Engine: 1.9.2-gke.1

## Set Up

### Create k8s v1.9+ cluster

``` bash
$ export ZONE=<zone name>
$ gcloud container clusters create istio-demo-1 \
    --machine-type=n1-standard-2 \
    --num-nodes=1 \
    --no-enable-legacy-authorization \
    --zone=$ZONE \
    --cluster-version=1.9.2-gke.1
```

### 获取集群管理权限

将管理员权限赋予 `system:serviceaccount:kube-system:default` 和当前用户 User.

```bash
$ kubectl create clusterrolebinding \
    --user system:serviceaccount:kube-system:default kube-system-cluster-admin \
    --clusterrole cluster-admin

$ kubectl create clusterrolebinding cluster-admin-binding \
    --clusterrole=cluster-admin \
    --user=$(gcloud config get-value core/account)
```

### 设置 Istio 环境

For Istio 0.5.1 you can follow the installation tutorial below. Also, you can follow the latest installation guides on Istio official site: [Quick Start](https://istio.io/docs/setup/kubernetes/quick-start.html) and [Sidecar Injection](https://istio.io/docs/setup/kubernetes/sidecar-injection.html).

Download Istio 0.5.1

``` bash
$ export ISTIO_VERSION=0.5.1
$ curl -L https://git.io/getLatestIstio | sh -
$ cd istio-0.5.1
```

应用 istio 相关 YAML 文件

```bash
$ kubectl apply -f install/kubernetes/istio.yaml
```

自动 sidecar injection

```bash
$ kubectl api-versions | grep admissionregistration
admissionregistration.k8s.io/v1beta1
```

安装 webhook

下载需要的文件 istio release 0.5.1

```bash
$ wget https://raw.githubusercontent.com/istio/istio/master/install/kubernetes/webhook-create-signed-cert.sh -P install/kubernetes/
$ wget https://raw.githubusercontent.com/istio/istio/master/install/kubernetes/webhook-patch-ca-bundle.sh -P install/kubernetes/
$ chmod +x install/kubernetes/webhook-create-signed-cert.sh install/kubernetes/webhook-patch-ca-bundle.sh
```

Install the sidecar injection configmap.

``` bash
$ ./install/kubernetes/webhook-create-signed-cert.sh \
    --service istio-sidecar-injector \
    --namespace istio-system \
    --secret sidecar-injector-certs

$ kubectl apply -f install/kubernetes/istio-sidecar-injector-configmap-release.yaml
```

Install the sidecar injector

``` bash
$ cat install/kubernetes/istio-sidecar-injector.yaml | \
     ./install/kubernetes/webhook-patch-ca-bundle.sh > \
     install/kubernetes/istio-sidecar-injector-with-ca-bundle.yaml

$ kubectl apply -f install/kubernetes/istio-sidecar-injector-with-ca-bundle.yaml

# Check sidecar injector status
$ kubectl -n istio-system get deployment -listio=sidecar-injector
NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
istio-sidecar-injector   1         1         1            1           26s
```

### 安装 fission

Set default namespace for helm installation, here we use `fission` as example namespace.
``` bash
$ export FISSION_NAMESPACE=fission
```

Create namespace & add label for Istio sidecar injection.

``` bash
$ kubectl create namespace $FISSION_NAMESPACE
$ kubectl label namespace $FISSION_NAMESPACE istio-injection=enabled
$ kubectl config set-context $(kubectl config current-context) --namespace=$FISSION_NAMESPACE
```

Follow the [installation guide](../../installation/) to install fission with flag `enableIstio` true.

``` bash
$ helm install --namespace $FISSION_NAMESPACE --set enableIstio=true --name istio-demo <chart-fission-all-url>
```

### Create a function

设置环境

``` bash
$ export FISSION_URL=http://$(kubectl --namespace fission get svc controller -o=jsonpath='{..ip}')
$ export FISSION_ROUTER=$(kubectl --namespace fission get svc router -o=jsonpath='{..ip}')
```


Let's create a simple function with Node.js.

``` js
# hello.js
module.exports = async function(context) {
    console.log(context.request.headers);
    return {
        status: 200,
        body: "Hello, World!\n"
    };
}
```

Create environment

``` bash
$ fission env create --name nodejs --image fission/node-env:latest
```

Create function

``` bash
$ fission fn create --name h1 --env nodejs --code hello.js --method GET
```

Create route

``` bash
$ fission route create --method GET --url /h1 --function h1
```

Access function

``` bash
$ curl http://$FISSION_ROUTER/h1
Hello, World!
```

### Install Istio Add-ons

* Prometheus

...