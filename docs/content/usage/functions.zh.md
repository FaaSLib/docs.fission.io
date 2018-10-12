---
title: "Functions"
draft: false
weight: 42
---

### 创建 function

在创建 function 之前，需要先创建 environment。

让我们写一个简单的 NodeJS 代码片段，用来输出 "Hello, world!"：

```js
module.exports = asysnc function(context) {
    return {
        status: 200,
        body: "Hello, world!\n"
    };
}
```

让我们在集群中创建这个 function。我们只是创建/注册funciton，并没有运行他。

```bash
$ fission fn create \
    --name hello \
    --code hello.js \
    --env node
```

接下来，创建一个专门用于上面 function 的 router，或者叫 HTTP trigger。

```bash
$ fission route create \
    --function hello \
    --url /hello \
trigger '5327e9a7-6d87-4533-a4fb-c67f55b1e492' created
```

当你访问 function 的 URL 时，将会得到如下结果。

```bash
$ curl http://${FISSION_ROUTER}/hello
Hello, world!
```

你可以创建带有 "newdeploy" 类型 excutor 的 function。设置 minimum, maximum。

```bash
$ fission fn create \
    --name hello \
    --code hello.js \
    --env node \
    --minscale 1 \
    --maxscale 5 \
    --executortype newdeploy
```

### 查看和更新 function 源代码

可以通过如下命令查看 function 的源代码

```js
$ fission fn get --name hello
module.exports = async function(context) {
    return {
        status: 200, 
        body: "Hello, world!\n"
    };
}
```

让我们更新一下 function 的输出，用 "Hello Fission" 代替 "Hello world"。用下面命令更新 function。

```bash
$ fission fn update \
    --name hello \
    --code ../hello.js \
package 'hello-js-ku9s' update
function 'hello' updated
```

让我们验证一下 function 的新相应。

```bash
$ curl http://${FISSION_ROUTER}/hello
Hello, Fission!
```

### Test and debug function

你可以用 test 命令来直接测试 function 。

```bash
$ fission fn test --name hello
Hello, Fission!
```

如果产生错误，那么 func 的执行日志将会打印出来。

```bash
$ fission fn test --name hello
Error calling function hello: 500 Internal server error (fission)

> fission-nodejs-runtime@0.1.0 start /usr/src/app
> node server.js

Codepath defaulting to  /userfunc/user
Port defaulting to 8888
user code load error: SyntaxError: Unexpected token function
::ffff:10.8.1.181 - - [16/Feb/2018:08:44:33 +0000] "POST /specialize HTTP/1.1" 500 2 "-" "Go-http-client/1.1"

```

你也可以用命令直接查看日志。

```bash
$ fission fn logs --name hello
[2018-02-16 08:41:43 +0000 UTC] 2018/02/16 08:41:43 fetcher received fetch request and started downloading: {1 {hello-js-rqew  default    0 0001-01-01 00:00:00 +0000 UTC <nil> <nil> map[] map[] [] nil [] }   user [] []}
[2018-02-16 08:41:43 +0000 UTC] 2018/02/16 08:41:43 Successfully placed at /userfunc/user
[2018-02-16 08:41:43 +0000 UTC] 2018/02/16 08:41:43 Checking secrets/cfgmaps
[2018-02-16 08:41:43 +0000 UTC] 2018/02/16 08:41:43 Completed fetch request
[2018-02-16 08:41:43 +0000 UTC] 2018/02/16 08:41:43 elapsed time in fetch request = 89.844653ms
[2018-02-16 08:41:43 +0000 UTC] user code loaded in 0sec 4.235593ms
[2018-02-16 08:41:43 +0000 UTC] ::ffff:10.8.1.181 - - [16/Feb/2018:08:41:43 +0000] "POST /specialize HTTP/1.1" 202 - "-" "Go-http-client/1.1"
[2018-02-16 08:41:43 +0000 UTC] ::ffff:10.8.1.182 - - [16/Feb/2018:08:41:43 +0000] "GET / HTTP/1.1" 200 16 "-" "curl/7.54.0"
```

### Fission 构建和编译产物 artifacts

大多数现实中的 function 会需要不止一个源文件。很容简单的提供源文件，以及让 Fission 负责构建工作。Fission 原生支持从源代码构建，和用编译产物创建 function。

你可以创建一个 source/deployment package 附着在 a function 。或者创建一个 package 供多个 function 使用。

#### Building functions from source

让我们用 python 举个例子。这个例子中依赖了 python pyyaml module。我们可以在 requirements.txt 中定义这个依赖。然后用一个简单的命令来创建他。目录结构如下。

```
sourcepkg/
├── __init__.py
├── build.sh
├── requirements.txt
└── user.py
```

文件内容如下。

```bash
$ cat user.py 

import sys
import yaml

document = """
  a: 1
  b:
    c: 3
    d: 4
"""

def main():
    return yaml.dump(yaml.load(document))

$ cat requirements.txt 
pyyaml

$ cat build.sh 
#!/bin/sh
pip3 install -r ${SRC_PKG}/requirements.txt -t ${SRC_PKG} && cp -r ${SRC_PKG} ${DEPLOY_PKG}
```

你需要提前创建一个 env，它包含了 evironment image 和 python-builder image。

```bash 
$ fission env create \ 
    --name python \
    --image fission/python-env:latest \
    --builder fission/python-builder:latest \
    --mincpu 40 \
    --maxcpu 80 \
    --minmemory 64 \
    --maxmemory 128 \
    --poolsize 2
```

然后我们打包源代码

```bash
$ zip -jr demo-src-pkg.zip sourcepkg/
  adding: __init__.py (stored 0%)
  adding: build.sh (deflated 24%)
  adding: requirements.txt (stored 0%)
  adding: user.py (deflated 25%)

$ fission fn create \
    --name hellopy \
    --env python \
    --src demo-src-pkg.zip  \
    --entrypoint "user.main" \
    --buildcmd "./build.sh"
function 'hellopy' created

$ fission route create \
    --function hellopy \
    --url /hellopy
```

一旦你创建 function。构建流程就开始启动。你可以检查构建 log。

```bash
$ kubectl -n fission-builder logs -f py3-4214348-59555d9bd8-ks7m4 builder
2018/02/16 11:44:21 Builder received request: {demo-src-pkg-zip-ninf-djtswo ./build.sh}
2018/02/16 11:44:21 Starting build...

=== Build Logs ===command=./build.sh
env=[PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin HOSTNAME=py3-4214348-59555d9bd8-ks7m4 PYTHON_4212095_PORT_8000_TCP_PROTO=tcp PY3_4214348_SERVICE_HOST=10.11.250.161 KUBERNETES_PORT=tcp://10.11.240.1:443 PYTHON_4212095_PORT=tcp://10.11.244.134:8000 PYTHON_4212095_PORT_8000_TCP=tcp://10.11.244.134:8000 PYTHON_4212095_PORT_8001_TCP_PROTO=tcp PYTHON_4212095_PORT_8001_TCP_ADDR=10.11.244.134 PY3_4214348_SERVICE_PORT=8000 PY3_4214348_SERVICE_PORT_BUILDER_PORT=8001 PY3_4214348_PORT_8001_TCP=tcp://10.11.250.161:8001 KUBERNETES_PORT_443_TCP_PORT=443 KUBERNETES_PORT_443_TCP_ADDR=10.11.240.1 PY3_4214348_SERVICE_PORT_FETCHER_PORT=8000 PY3_4214348_PORT_8000_TCP=tcp://10.11.250.161:8000 PY3_4214348_PORT_8001_TCP_PORT=8001 PYTHON_4212095_SERVICE_PORT_FETCHER_PORT=8000 PYTHON_4212095_PORT_8000_TCP_ADDR=10.11.244.134 KUBERNETES_SERVICE_HOST=10.11.240.1 PY3_4214348_PORT=tcp://10.11.250.161:8000 PYTHON_4212095_SERVICE_PORT_BUILDER_PORT=8001 PYTHON_4212095_PORT_8001_TCP=tcp://10.11.244.134:8001 PY3_4214348_PORT_8000_TCP_PROTO=tcp PY3_4214348_PORT_8000_TCP_PORT=8000 KUBERNETES_SERVICE_PORT_HTTPS=443 KUBERNETES_PORT_443_TCP=tcp://10.11.240.1:443 PYTHON_4212095_PORT_8001_TCP_PORT=8001 PY3_4214348_PORT_8000_TCP_ADDR=10.11.250.161 PY3_4214348_PORT_8001_TCP_PROTO=tcp KUBERNETES_SERVICE_PORT=443 PYTHON_4212095_SERVICE_PORT=8000 PYTHON_4212095_PORT_8000_TCP_PORT=8000 PY3_4214348_PORT_8001_TCP_ADDR=10.11.250.161 KUBERNETES_PORT_443_TCP_PROTO=tcp PYTHON_4212095_SERVICE_HOST=10.11.244.134 HOME=/root SRC_PKG=/packages/demo-src-pkg-zip-ninf-djtswo DEPLOY_PKG=/packages/demo-src-pkg-zip-ninf-djtswo-c40gfu]
Collecting pyyaml (from -r /packages/demo-src-pkg-zip-ninf-djtswo/requirements.txt (line 1))
  Downloading PyYAML-3.12.tar.gz (253kB)
Installing collected packages: pyyaml
  Running setup.py install for pyyaml: started
    Running setup.py install for pyyaml: finished with status 'done'
Successfully installed pyyaml-3.12
==================
2018/02/16 11:44:24 elapsed time in build request = 3.460498847s
```

一旦创建成功。就可以像 function 的 URL 发起请求。

```bash
$curl http://$FISSION_ROUTER/hellopy
a: 1
b: {c: 3, d: 4}
```

如果你正配合源代码使用 Fission 。请确保你正在使用推荐的开发流程(developer-workflow)

#### Using compiled artifacts with Fission

在有些使用场景，你会将预编译好的 package 部署到 fission 中。在这个例子中我们使用 python 。也可以使用任何其他可编译的 package。

我们用一个 pyhone 文件目录。然后转入一个 deployment package

```bash
$ cat testDir/hello.py
def main():
    return "Hello, world!"

$zip -jr demo-deploy-pkg.zip testDir/

```

用该 deployment package 创建 function 和 router 然后测试一下。

```bash
$ fission fn create \
    --name hellopy \
    --env python \
    --deploy demo-deploy-pkg.zip \
    --entrypoint "hello.main"
function 'hellopy' created

$ fission route create \
    --function hellopy \
    --url /hellopy

$ curl http://$FISSION_ROUTER/hellopy
Hello, world!
```

### 查看 function 信息

```bash
$ fission fn getmeta --name hello
NAME  UID                                  ENV
hello 34234b50-12f5-11e8-85c9-42010aa00010 node

$ fission fn list
NAME   UID                                  ENV  EXECUTORTYPE MINSCALE MAXSCALE TARGETCPU
hello  34234b50-12f5-11e8-85c9-42010aa00010 node poolmgr      0        1        80
hello2 e37a46e3-12f4-11e8-85c9-42010aa00010 node newdeploy    1        5        80
```
