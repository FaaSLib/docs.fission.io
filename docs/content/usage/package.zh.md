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

