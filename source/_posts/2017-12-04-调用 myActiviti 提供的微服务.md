---
layout: post
title: 调用 myActiviti 提供的微服务
description: 本章内容向您讲解如何调用 myActiviti 提供的微服务。
category: 工作流
date: 2017-12-04
list_number: false
---

## [开始、流转与仅仅开始](#开始、流转与仅仅开始)
在上一章中，我们使用 `justStart` 方法调用了流程，这个方法的中文名称是 “仅仅开始”，它的 api 格式如下：

```java
./wfservice/justStart?businessId=${您要启动的业务ID}
```

其中 `businessId` 是必填项，它描述您想启动哪个业务的流程（myActiviti 中每个流程都对应唯一且不重复的业务）。这个方法的返回结果形如：

```json
{
    "DataRows": [
        {
            "actId": "drafter",
            "actName": "起草",
            "actRole": [
                "projectManager"
            ],
            "exeId": "40013",
            "isEnd": "0",
            "taskId": "40016"
        }
    ],
    "RetCode": "1",
    "RetVal": "1",
    "isEnd": "0"
}
```

说明如下：

| 属性     | 说明   |
|:--------|:-------|
| `RetCode`  | 可能的返回值“0/1/2”，0 代表工作流发生问题，1 代表正常返回且业务逻辑正常，2 代表正常返回但业务逻辑异常|
| `RetVal` | 对返回情况的进一步描述，当 RetCode 为 0 或 2 时这里会给出详细信息 |
| `isEnd` | 可能的返回值“0/1”，0 代表主流程未结束，1代表流程已结束。出现在 `DataRows` 中则表示分支流程的结束情况 |
| `DataRows`     | 展示当前所有分支流程信息的数组 |
| `actId`     | 只出现在 `DataRows` 中，显示流程图中此环节 ID |
| `actName`     | 只出现在 `DataRows` 中，显示流程图中此环节名称 |
| `actRole`     | 只出现在 `DataRows` 中，显示流程图中此可以处理此环节的角色（数组） |
| `exeId`     | 只出现在 `DataRows` 中，显示此次流程实例执行变量的 ID |
| `taskId`     | 只出现在 `DataRows` 中，显示此次流程实例任务变量的 ID |

- 对 myActiviti 来说，主流程中的分支流程（`DataRows` 中的内容）永远存在，只不过普通情况下分支流程只有一个，而并行处理情况下分支流程会有多个。

- `exeId` 是业务系统在返回后需要用到的数据，它是业务系统下一次调用“流转”等操作的必要输入参数。

- `taskId` 是业务系统在返回后需要保存的数据，它是业务系统调用“撤回”等异步操作（即不处于激活状态的环节发起的操作）时必要的输入参数。

- `exeId` 和 `taskId` 的区别是，大多数情况下一个流程实例的 `exeId` 是不变的，但 `taskId` 永远都在改变。

以上就是 `justStart` 操作的全部内容，业务系统调用并获得返回数据后可以按自己的需要使用这些信息。

多数情况下，业务系统会接着调用 `flow`（流转）操作，仍以上一章的立项流程为例，起草环节的流转 api 格式如下：

```java
./wfservice/flow?exeId=40013
                &dealRole=projectManager
                &dealPerson=ZhangSan
                &formData={form_data:{endApply:false, nextOU:"M"}}
```


## [挂起、恢复与终止](#挂起、恢复与终止)

## [签收、取消签收与撤回](#签收、取消签收与撤回)

## [软结束与消息事件](#软结束与消息事件)