---
title: "Accessing Secrets in Functions"
draft: false
weight: 36h
---

函数可以获取Kubernetes的
[Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
和
[ConfigMaps](https://kubernetes.io/docs/concepts/storage/volumes/#configmap).

为这些对象 API keys, authentication tokens, 等，创建secrets

为任何不需要使用secret的别的配置，创建config maps。

### Create A Secret or a ConfigMap

你可以通过 Kubernetes CLI 创建一个 Secret 或者 ConfigMap :

```bash
$ kubectl -n default create secret generic my-secret --from-literal=TEST_KEY="TESTVALUE"

$ kubectl -n default create configmap my-configmap --from-literal=TEST_KEY="TESTVALUE"
```

或者， 使用 `kubectl create -f <filename.yaml>` 从yaml文件创建：

```yaml
apiVersion: v1
kind: Secret
metadata:
  namespace: default
  name: my-secret
data:
  TEST_KEY: VEVTVFZBTFVF # value after base64 encode
type: Opaque

---
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: default
  name: my-configmap
data:
  TEST_KEY: TESTVALUE
```

### 获取 Secrets 和 ConfigMaps

Secrets 和 configmaps 的获取方式类似，  每个 secret 或者
configmap 是一对键值对。Fission 把这些设置成你可以从函数中读取的文件。

```bash
# Secret path
/secrets/<namespace>/<name>/<key>

# ConfigMap path
/configs/<namespace>/<name>/<key>
```

之前的例子， 路径是：

```bash
# secret my-secret
/secrets/default/my-secret/TEST_KEY

# confimap my-configmap
/configs/default/my-configmap/TEST_KEY
```

现在，我们来创建一个简单的python function (leaker.py) ，用来返回 Secret `my-secret` 和 ConfigMap `my-configmap`的值。

```python
# leaker.py

def main():
    path = "/configs/default/my-configmap/TEST_KEY"
    f = open(path, "r")
    config = f.read()

    path = "/secrets/default/my-secret/TEST_KEY"
    f = open(path, "r")
    secret = f.read()

    msg = "ConfigMap: %s\nSecret: %s" % (config, secret)

    return msg, 200
```

创建一个 environment 和一个 function:

```bash
# create python env
$ fission env create --name python --image fission/python-env

# create function named "leaker"
$ fission fn create --name leaker --env python --code leaker.py --secret my-secret --configmap my-configmap
```

运行函数，输出应该如下：

```bash
$ fission function test --name leaker
ConfigMap: TESTVALUE
Secret: TESTVALUE
```

{{% notice note %}}
如果 Secret 或者 ConfigMap 的值更新了，函数有时可能获取不到更新后的值，可能获取到的是缓存的旧值。
{{% /notice %}}

