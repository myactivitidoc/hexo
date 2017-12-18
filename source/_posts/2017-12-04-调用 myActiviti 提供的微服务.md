---
layout: post
title: 调用 myActiviti 提供的微服务
description: 本章内容向您讲解如何调用 myActiviti 提供的微服务。
category: 工作流
date: 2017-12-04
list_number: false
---

## [开始、流转与仅仅开始](#开始、流转与仅仅开始)

### [仅仅开始（justStart）](#仅仅开始（justStart）)
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

### [流转（flow）](#流转（flow）)
多数情况下，业务系统会接着调用 `flow`（流转）操作，仍以上一章的立项流程为例，起草环节的流转 api 格式如下：

```java
./wfservice/flow?exeId=40013
                &dealRole=projectManager
                &dealPerson=ZhangSan
                &formData={form_data:{endApply:false, nextOU:"M"}}
```

其中，`exeId`（执行变量 Id）、`dealRole`（参与者角色）和  `dealPerson`（参与者）是必填项，`formData`（参与流转的流程变量）是选填项，但是如果待流转的环节含有条件序列流，在发起流转请求时必须包含可以执行条件判断的所有变量。比如我们在上一章设计的流程图中起草环节有可能进行如下判断：


| 序列流指向     | 序列流条件   |
|:--------|:-------|
| `总院TMO审核`  | ${endApply == false && nextOU == "M"} |
| `A分院TMO审核` | ${endApply == false && nextOU == "A"} |
| `B分院TMO审核` | ${endApply == false && nextOU == "B"} |
| `结束环节`     | ${endApply == true} |

因此在发起这个环节的流转请求时我们或者给出 `endApply` 为 false 和 `nextOU` 的值；或者给出 `endApply` 为 true（此时可以不提供 `nextOU` 的值）。具体赋值的格式如下：

```java
formData={form_data:{endApply:false, nextOU:"M"}}`
```

这个方法的返回结果形如：

```json
{
    "DataRows": [
        {
            "actId": "tmoCheak",
            "actName": "总院TMO审核",
            "actRole": [
                "tmo"
            ],
            "exeId": "40013",
            "isEnd": "0",
            "taskId": "40028"
        }
    ],
    "RetCode": "1",
    "RetVal": "1",
    "isEnd": "0"
}
```

内容格式和上一小节 `justStart` 返回的内容格式相同，值得注意的是，这次返回的是流程图中起草的后一个环节。

### [开始（start）](#开始（start）)
之前我们介绍了用于开始流程的 `justStart` 和用于流转的 `flow`，但如果用户不想在第一个环节停留，我们还提供了 `start` 方法用于开始流程并通过第一个环节。这个方法可以看作是先 `justStart` 再 `flow`，它的 api 格式如下：

```java
./wfservice/start?businessId=business2
                &dealRole=projectManager
                &dealPerson=ZhangSan
                &formData={form_data:{endApply:false, nextOU:"M"}}
```

其中，`businessId`（要启动的业务 Id）、`dealRole`（参与者角色）和  `dealPerson`（参与者）是必填项，`formData`（参与流转的流程变量）是选填项，但是如果待流转的环节含有条件序列流，在发起流转请求时必须包含可以执行条件判断的所有变量。它的返回和 `flow` 操作的返回是相同的。

## [挂起、恢复与终止](#挂起、恢复与终止)

### [挂起（suspend）](#挂起（suspend）)
挂起是一种暂停某一流程实例流转的方法，它的 api 格式如下：

```java
./wfservice/suspend?exeId=${希望挂起的流程实例的执行变量ID}
```

其中 `exeId` 是必填项。挂起后返回结果 json 中只需要看两个属性：

| 属性     | 说明   |
|:--------|:-------|
| `RetCode`  | 可能的返回值“0/1/2”，0 代表工作流发生问题，1 代表正常返回且业务逻辑正常，2 代表正常返回但业务逻辑异常|
| `RetVal` | 对返回情况的进一步描述，当 RetCode 为 0 或 2 时这里会给出详细信息 |

当挂起成功后，如果再流转这个流程实例，返回 json 如下：

```json
{
    "DataRows": null,
    "RetCode": "2",
    "RetVal": "Cannot claim a suspended task",
    "isEnd": null
}
```

也就是流转并没有成功。如果想让已挂起的流程实例再度流转，需要使用恢复方法。

### [恢复（activate）](#恢复（activate）)
操作是将已挂起流程实例恢复正常的方法，它的 api 格式如下：

```java
./wfservice/activate?exeId=${希望恢复的流程实例的执行变量ID}
```

其中 `exeId` 是必填项。恢复后返回结果 json 中只需要看两个属性：

| 属性     | 说明   |
|:--------|:-------|
| `RetCode`  | 可能的返回值“0/1/2”，0 代表工作流发生问题，1 代表正常返回且业务逻辑正常，2 代表正常返回但业务逻辑异常|
| `RetVal` | 对返回情况的进一步描述，当 RetCode 为 0 或 2 时这里会给出详细信息 |

当恢复成功后，这个流程实例可以再次正常流转。

### [终止（terminate）](#终止（terminate）)
终止是将已启动的流程实例强制结束的方法，它的 api 格式如下：

```java
./wfservice/terminate?exeId=${希望终止的流程实例的执行变量ID}
```

其中 `exeId` 是必填项。恢复后返回结果 json 中只需要看两个属性：

| 属性     | 说明   |
|:--------|:-------|
| `RetCode`  | 可能的返回值“0/1/2”，0 代表工作流发生问题，1 代表正常返回且业务逻辑正常，2 代表正常返回但业务逻辑异常|
| `RetVal` | 对返回情况的进一步描述，当 RetCode 为 0 或 2 时这里会给出详细信息 |

当终止成功后，此流程实例无法再被使用。

## [签收、取消签收与撤回](#签收、取消签收与撤回)

### [签收（receipt）](#签收（receipt）)
签收是声明用户获取到任务的一个方法，它的 api 格式如下：

```java
./wfservice/receipt?exeId=${签收的流程实例的执行变量ID}
                   &dealPerson=${参与者}
```

其中，`exeId` 和 `dealPerson` 是必填项。签收成功后会显示该环节详细信息，同时这个任务无法再被其他人签收（除非执行在我们下一小节会介绍的取消签收方法）。

签收的意义在于防止任务被撤回，未签收的任务可以被它的来源人撤回，但已签收的任务不能被撤回。最后，如果业务系统不使用签收操作也无妨，在流转时会自动执行签收操作。

### [取消签收（unreceipt）](#取消签收（unreceipt）)
取消签收是已签收任务的用户放弃签收的方法，它的 api 格式如下：

```java
./wfservice/unreceipt?exeId=${取消签收的流程实例的执行变量ID}
                   &dealPerson=${参与者}
```

其中，`exeId` 和 `dealPerson` 是必填项。取消签收成功后会显示该环节详细信息，同时这个任务进入无人认领状态，可以被其他用户进行签收，也可被它的来源人撤回。

### [撤回（withdraw）](#撤回（withdraw）)

## [软结束与消息事件](#软结束与消息事件)

### [软结束标志](#软结束标志)

### [消息触发事件（message）](#消息触发事件（message）)

## [总结](#总结)