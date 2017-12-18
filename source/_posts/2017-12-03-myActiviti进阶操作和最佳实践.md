---
layout: post
title: myActiviti 进阶操作和最佳实践
description: 本章内容向您讲解 myActiviti 高级操作和最佳实践。
category: 工作流
date: 2017-12-03
list_number: false
---

## [进阶操作](#进阶操作)
本节中的方法仍在开发之中，最终调用格式未完全确定，您可以先睹为快。

### [并行调用](#并行调用)
有时候我们需要多个流程分支并行流转，myActiviti 可以通过并行网关实现这样的效果。并行网关的形状是一个菱形中间加一个 “+”，效果是它发散的多条序列流中每一条都可以独立流转；一般一个并行网关发散之后还会有另一个并行网关作为收集，例如下图：

{% asset_img parallel.png %}

在这个流程图中，当“请各分院处理”环节流转过后，返回值会如下所示：

```json
{
    "DataRows": [
        {
            "actId": "ADispose",
            "actName": "A分院处理",
            "actRole": [
                "interfaceA"
            ],
            "exeId": "40141",
            "isEnd": "0",
            "taskId": "40144"
        },
        {
            "actId": "BDispose",
            "actName": "B分院处理",
            "actRole": [
                "interfaceB"
            ],
            "exeId": "40142",
            "isEnd": "0",
            "taskId": "40147"
        }
    ],
    "RetCode": "1",
    "RetVal": "1",
    "isEnd": "0"
}
```

值得注意的是，`DataRows` 中出现两个流程分支，这两个分支都可以进行流转；并且当流转到收集点时，先到的流程分支会返回：

```json
{
    "DataRows": [],
    "RetCode": "1",
    "RetVal": "1",
    "isEnd": "0"
}
```

后到的分支会返回：

```json
{
    "DataRows": [
        {
            "actId": "parallel_end",
            "actName": "总院查看处理结果",
            "actRole": [
                "interface"
            ],
            "exeId": "40130",
            "isEnd": "0",
            "taskId": "40152"
        }
    ],
    "RetCode": "1",
    "RetVal": "1",
    "isEnd": "0"
}
```

即两条分支收成了一条，以上两条分支无论谁先谁后都是如此。

### [自由流](#自由流)

自由流是 BPMN 2.0 新增流转方式之一，它的含义是选择一组特定的环节，组内环节每个都可以无障碍的流转到另一个。在 myActiviti 中，自由流如下图所示：

{% asset_img free.png %}

在自由流内部流转时，采用 `jump` 而非 `flow` 的方式，如：

```java
./wfservice/jump?exeId=40013
                &dealRole=projectManager
                &dealPerson=ZhangSan
                &formData={form_data:{endApply:false, nextOU:"M"}}
```
（自由流功能尚在开发中，参数格式并没有完全确定）

自由流跳转的返回的内容与正常流转返回内容相同。

## [最佳实践](#最佳实践)
- 单入口，单出口。虽然工作流支持多个结束环节，但不推荐这样画，建议每个图都有唯一的结束环节。

- 先复制，后变更。当流程图贴近现实场景后，会变得非常复杂，每个流程图的工作量可以达到人天级别，此时编辑流程时防止“手滑”变得至关重要（比如以为元素粘上了实际没有粘上）,尤其是对一些生产环境的流程进行变更时更是如此，所以一定要培养勤备份的习惯。

- 测试环境设计，生产环境导入。一般工作流至少有生产环境和测试环境两套。无论是新增流程还是修改流程，都应该先在测试环境设计、变更、测试，无问题后导出工作流文件，再在生产环境导入工作流文件。切记不要在生产环境进行设计。