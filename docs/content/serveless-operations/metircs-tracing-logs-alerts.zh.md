---
title: "metircs tracing logs alerts"
draft: false
weight: 35
---

### 监控无服务系统

* 很多方面需要监控： 日志，计量，告警，跟踪
* 使用fluentd进行日志收集--保存在可以搜索的工具中（e.g. Elastic stack）
* 计量：prometheus包含报警管理模块，可以用来基于计量数据进行报警
* 跟踪：使用jaeger或者其他的开源追踪工具

### fission metrics

* fission 可以自动记录每个函数的执行时间，成功率，错误码，fission负载等

* fission集成了prometheus进行计量指标收集

* 你可以通过grafana构建dashboard，通过prometheus进行告警


