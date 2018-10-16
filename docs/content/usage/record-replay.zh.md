---
title: "Recorder Replay"
draft: false
weight: 35
---

### Record-Replay

用最简单的方式重现bugs，并修复它们。

* Record-Replay是一种用来保存激活函数事件的方式，在随后的测试或调试阶段，再度模拟这些事件。
* 测试：模拟保存的事件，触发对函数的请求，判断新版本函数的表现，是否与老版本一致：回归测试
* 调试：触发保存的旧事件，观察函数的执行过程。

### 使用场景

* 开发：开发可以在测试的过程中，使用记录功能，用来保证可以重现一个错误。
* 运维：可以记录部分生产环境中的流量，保证开发者可以重现问题，调试问题，和验证新版本

### fission

* fission有内建的record-replay功能，可以存储http请求和响应，然后根据需要重现
* 在fission中可以为函数创建一个recorder资源，可以配置记录什么，以及需要保存多久
* 根据需要，如测试一个新版本功能或者调试一个旧的版本，重现一个请求

```
[root@dev-86-208 hello]# fission env create --name nodejs --iamge fission/node-env

[root@dev-86-208 hello]# fission fn create --spec --name hello-m --env nodejs --code hello.js 

[root@dev-86-208 hello]# fission spec apply --wait
1 package created: hello-js-u1vu
1 function created: hello-m

[root@dev-86-208 hello]# fission fn test --name hello-m
hello.world!

[root@dev-86-208 hello]# fission route create --function hello-m --url /hello-m --method GET
trigger 'e9cd950b-9a10-4354-b941-f6196df3a86e' created

[root@dev-86-208 hello]# fission recorder create --name msxu-recorder --function hello-m
recorder 'msxu-recorder' created

[root@dev-86-208 hello]# time -p curl http://10.96.164.184/hello-m
hello.world!
real 0.06
user 0.00
sys 0.00


[root@dev-86-208 hello]# fission records view -v
REQUID                                  REQUEST METHOD FUNCTION RESPONSE STATUS TRIGGER
REQae4eedf9-2e95-4fb1-876b-cb437150f48c GET            hello-m  200 OK          e9cd950b-9a10-4354-b941-f6196df3a86e

[root@dev-86-208 hello]# fission replay --reqUID REQae4eedf9-2e95-4fb1-876b-cb437150f48c
hello.world!
```

