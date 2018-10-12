---
title: "Packaging source code"
draft: false
weight: 35
---

### Creating a Source Package

在创建 package 之前，你需要创建一个 env 。

```bash
$ fission env create \
    --name pythonsrc \
    --image fission/python-env:latest \
    --builder fission/python-builder:latest \
    --mincpu 40 \
    --maxcpu 80 \
    --minmemory 64 \
    --maxmemory 128 \
    --poolsize 2
environment 'pytonsrc' created
```

 让我们船舰一个简单的 python function。他依赖 `pyyaml` module。我们需要设置 `requirements.txt` 和一个简单的构建命令。目录结构和内容如下。

```
sourcepkg/
├── __init__.py
├── build.sh
├── requirements.txt
└── user.py
```
And the file contents:
```
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

$zip -jr demo-src-pkg.zip sourcepkg/
  adding: __init__.py (stored 0%)
  adding: build.sh (deflated 24%)
  adding: requirements.txt (stored 0%)
  adding: user.py (deflated 25%)
```

用上面生成的压缩包创建 package

```bash
$ fission package create \
    --sourcearchive demo-src-pkg.zip \
    --env pythonsrc \
    --buildcmd "./build.sh"
Package 'demo-src-pkg-zip-8lwt' created
```

当我们在使用 source package 的时候，我们提供一个构建命令 build command。一旦我们创建 package，构建流程就会开启。可以通过 `fission package info` 来看到日志。

```
$ fission pkg info --name demo-src-pkg-zip-8lwt
Name:        demo-src-pkg-zip-8lwt
Environment: pythonsrc
Status:      succeeded
Build Logs:
Collecting pyyaml (from -r /packages/demo-src-pkg-zip-8lwt-v57qil/requirements.txt (line 1))
  Using cached PyYAML-3.12.tar.gz
Installing collected packages: pyyaml
  Running setup.py install for pyyaml: started
    Running setup.py install for pyyaml: finished with status 'done'
Successfully installed pyyaml-3.12
```

用上边的 package 可以创建 function。当一个 package 关联到 source package, environment, build command后，创建 function 时忽略新的参数。

这个时候唯一能增加的是 entrypoint

```shell
$ fission fn create \
    --name srcpy \
    --pkg demo-src-pkg-zip-8lwt \
    --entrypoint "user.main"
function 'srcpy' created

# Run the function
$ fission fn test --name scrpy
a: 1
b: {c: 3, d: 4}
```

### Creating a Deployment Package

创建 package 之前需要先创建一个带 builder image 的 environment 。

```shell
$ fission env create \
    --name pythondeploy \
    --image fission/python-env:latest \
    --builder fission/python-builder:latest \
    --mincpu 40 \
    --maxcpu 80 \
    --minmemory 64 \
    --maxmemory 128 \
    --poolsize 2
environment 'pythonsrc' created
```

我们将用一个 python 例子来创建 deployment archive

```python
$ cat testDir/hello.py
def main():
    return "Hello, world!"

$zip -jr demo-deploy-pkg.zip testDir/
```

用之前创建的 archive 和 environment，你可以创建一个 package。

```shell
$ fission package create 
    --deployarchive demo-deploy-pkg.zip \
    --env pythondeploy
Package 'demo-deploy-pkg-zip-whzl' created
```

因为是一个 deployment archive。所以没有必要构建他。所以构建日志是空的。

```
$ fission package info --name demo-deploy-pkg-zip-whzl
Name:        demo-deploy-pkg-zip-xlaw
Environment: pythondeploy2
Status:      succeeded
Build Logs:
```

最后你可以创建一个 function 然后测试他。

```shell
$ fission fn create \
    --name deploypy \
    --pkg demo-deploy-pkg-zip-whzl \
    --entrypoint "hello.main"
$ fission fn test --name deploypy
Hello, world!
```

这个例子展示了在不需要每次都从源码构建时，如何使用 package。更好的实践见[Specifications](../developer-workflow/).
