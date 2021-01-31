---
layout: post
title: 调用 myActiviti 提供的微服务
description: 本章内容向您讲解如何调用 myActiviti 提供的微服务。
category: 工作流
date: 2017-12-04
list_number: false
---

## [开始与流转](#开始与流转)

### [开始（begin）](#开始（begin）)
`begin` 方法用于开始流程，它的 api 格式如下：

```java
./wfservice/begin?businessId=${您要启动的业务ID}
                     &form={${所有环节参与变量}}
                     &listener={${所有监听器参与变量}}
```

其中 `businessId` 是必填项，它描述您想启动哪个业务的流程（myActiviti 中每个流程都对应唯一且不重复的业务），`form` 和 `listener` 为选填项。一个典型的 begin 方法如下（注意：如果在浏览器中调用这个服务，需要将 `{` 转义为 `%7B`，将 `}` 转义为 `%7D`。）：

```java
./wfservice/begin?businessId=business2
                 &form={myVariable1:'mv'}
                 &listener={myListener1:{myListener1Variable1:'mlv'}}
```
这个方法的返回结果形如：

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
            "procInstId": "40013",
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
| `procInstId`  | 只出现在 `DataRows` 中，显示此次流程实例的 ID |
| `exeId`     | 只出现在 `DataRows` 中，显示此次流程实例的执行变量的 ID |
| `taskId`     | 只出现在 `DataRows` 中，显示此次流程实例的任务变量的 ID |

- 对 myActiviti 来说，主流程中的分支流程（`DataRows` 中的内容）永远存在，只不过普通情况下分支流程只有一个，而并行处理情况下分支流程会有多个。

- `taskId` 是业务系统在返回后需要用到的数据，它是业务系统下一次调用“流转”等操作的必要输入参数。

- `exeId` 是业务系统在返回后用于参考的数据，大多数情况下一个流程实例中 exeId 不会改变，除非出现并行网关。

- `procInstId` 是一个流程实例中确定不会改变的数据，它可以唯一标识这个流程实例，大多数情况下 `exeId` 与它相同，但出现并行分支时 `exeId` 会不同于它。

- `procInstId`、`exeId` 和 `taskId` 的区别是，一个流程实例的 `procInstId` 永远不变，大多数情况下 `exeId` 也不会变，但 `taskId` 永远都在改变。

- 关于监听器的更多信息请参阅下一章

以上就是 `begin` 操作的全部内容，业务系统调用并获得返回数据后可以按自己的需要使用这些信息。

### [流转（complete）](#流转（complete）)
`complete` 是我们推荐使用的流转方式，这种方式采用 `taskId` 作为流转关键参数，如前面所提 `taskId` 的特点是每次流转后都会改变，使用它作为关键参数可以实现明确指定当前环节（正在流转的环节）和避免多次抖动提交造成过度流转（流转到预期之外的环节）的需求。起草环节的流转 api 格式如下（注意：如果在浏览器中调用这个服务，需要将 `{` 转义为 `%7B`，将 `}` 转义为 `%7D`。）：

```java
./wfservice/complete?taskId=40016
                &assignee=ZhangSan
                &form={${所有环节参与变量}}
                &listener={${所有监听器参与变量}}
```

其中，`taskId`（任务 Id）是必填项，`assignee`（环节参与者）、`form`（环节参与变量）和 `listener`（监听器参与变量）是选填项，同样如果待流转的环节含有条件序列流，在发起 /complete 请求时必须包含可以执行条件判断的所有变量（封装于 `form` 中）。比如我们在上一章设计的流程图中起草环节有可能进行如下判断：

| 序列流指向     | 序列流条件   |
|:--------|:-------|
| `总院TMO审核`  | ${endApply == false && nextOU == "M"} |
| `A分院TMO审核` | ${endApply == false && nextOU == "A"} |
| `B分院TMO审核` | ${endApply == false && nextOU == "B"} |
| `结束环节`     | ${endApply == true} |

因此在发起这个环节的流转请求时我们或者给出 `endApply` 为 false 和 `nextOU` 的值；或者给出 `endApply` 为 true（此时可以不提供 `nextOU` 的值）。具体赋值的格式如下：

```java
form={endApply:false, nextOU:"M"}
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
            "procInstId": "40013",
            "taskId": "40028"
        }
    ],
    "RetCode": "1",
    "RetVal": "1",
    "isEnd": "0"
}
```

内容格式和上一小节 `begin` 返回的内容格式相同，值得注意的是，这次返回的是流程图中起草的后一个环节。

"form" 中传入的变量称为环节参与变量，这些变量都是局部变量即仅在当前环节有效，当前环节结束后就无法访问。如果您需要一个长期有效的变量，可以在环节参与变量名前加“$”,如：

```java
form={$endApply:false, $nextOU:"M"}
```

这样环节参与变量“endApply”和“nextOU”就会在流程实例生命周期中一直有效，它的意义在“监听器”一章可以看到。

## [挂起、恢复与终止](#挂起、恢复与终止)

### [挂起（suspend）](#挂起（suspend）)
挂起是一种暂停某一流程实例流转的方法，它的 api 格式如下：

```java
./wfservice/suspend?procInstId=${希望挂起的流程实例的ID}
```

其中 `procInstId` 是必填项。挂起后返回结果 json 中只需要看两个属性：

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
./wfservice/activate?procInstId=${希望恢复的流程实例的ID}
```

其中 `procInstId` 是必填项。恢复后返回结果 json 中只需要看两个属性：

| 属性     | 说明   |
|:--------|:-------|
| `RetCode`  | 可能的返回值“0/1/2”，0 代表工作流发生问题，1 代表正常返回且业务逻辑正常，2 代表正常返回但业务逻辑异常|
| `RetVal` | 对返回情况的进一步描述，当 RetCode 为 0 或 2 时这里会给出详细信息 |

当恢复成功后，这个流程实例可以再次正常流转。

### [终止（terminate）](#终止（terminate）)
终止是将已启动的流程实例强制结束的方法，它的 api 格式如下：

```java
./wfservice/terminate?procInstId=${希望终止的流程实例的ID}
```

其中 `procInstId` 是必填项。恢复后返回结果 json 中只需要看两个属性：

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
撤回是任务的来源人将任务取回的方法，它的 api 格式如下：

```java
./wfservice/withdraw?taskId=${此任务前一环节的任务变量ID}
```

其中 `taskId` 是必填项，值为当前最新环节的 `taskId`，撤回成功后会显示前一环节详细信息。

注意，您只能从当前最新环节进行撤回。

## [历史遗留](#历史遗留)
以下 API 曾被开发，但经时间证明它们并非最佳实践因而不再推荐使用。这些 API 一直可用，但不会再增加特性，在此列出并建议使用了它们的用户尽快替换。

### [流转（flow）](#流转（flow）)
`flow` 是曾经的用于流转的 API ，使用 `exeId` 做为流转关键参数，它的格式如下：

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
formData={form_data:{endApply:false, nextOU:"M"}}
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
            "procInstId": "40013",
            "taskId": "40028"
        }
    ],
    "RetCode": "1",
    "RetVal": "1",
    "isEnd": "0"
}
```

内容格式和上一小节 `justStart` 返回的内容格式相同，值得注意的是，这次返回的是流程图中起草的后一个环节。

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
            "procInstId": "40013",
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
| `procInstId`  | 只出现在 `DataRows` 中，显示此次流程实例的 ID |
| `exeId`     | 只出现在 `DataRows` 中，显示此次流程实例的执行变量的 ID |
| `taskId`     | 只出现在 `DataRows` 中，显示此次流程实例的任务变量的 ID |

- 对 myActiviti 来说，主流程中的分支流程（`DataRows` 中的内容）永远存在，只不过普通情况下分支流程只有一个，而并行处理情况下分支流程会有多个。

- `exeId` 是业务系统在返回后需要用到的数据，它是业务系统下一次调用“流转”等操作的必要输入参数。

- `taskId` 是业务系统在返回后需要保存的数据，它是业务系统调用“撤回”等异步操作（即不处于激活状态的环节发起的操作）时必要的输入参数。

- `procInstId` 是一个流程实例中确定不会改变的数据，它可以唯一标识这个流程实例，大多数情况下 `exeId` 与它相同，但出现并行分支时 `exeId` 会不同于它。

- `procInstId`、`exeId` 和 `taskId` 的区别是，一个流程实例的 `procInstId` 永远不变，大多数情况下 `exeId` 也不会变，但 `taskId` 永远都在改变。

以上就是 `justStart` 操作的全部内容，业务系统调用并获得返回数据后可以按自己的需要使用这些信息。

### [开始（start）](#开始（start）)
`start` 方法是曾经的用于开始流程并通过第一个环节的 API，可以看作是先 `justStart` 再 `flow`，它的 api 格式如下：

```java
./wfservice/start?businessId=business2
                &dealRole=projectManager
                &dealPerson=ZhangSan
                &formData={form_data:{endApply:false, nextOU:"M"}}
```

其中，`businessId`（要启动的业务 Id）、`dealRole`（参与者角色）和  `dealPerson`（参与者）是必填项，`formData`（参与流转的流程变量）是选填项，但是如果待流转的环节含有条件序列流，在发起流转请求时必须包含可以执行条件判断的所有变量。它的返回和 `flow` 操作的返回是相同的。

### [移动（move）](#移动（move）)
`move` 采用 `taskId` 作为流转关键参数，如前面所提 `taskId` 的特点是每次流转后都会改变，使用它作为关键参数可以实现明确指定当前环节（正在流转的环节）和避免多次抖动提交造成过度流转（流转到预期之外的环节）的需求。起草环节的流转 api 格式如下：

```java
./wfservice/move?taskId=40016
                &assignee=ZhangSan
                &formData={form_data:{endApply:false, nextOU:"M"}}
```

其中，`taskId`（任务 Id）是必填项，`assignee`（环节参与者）和 `formData`（参与流转的流程变量）是选填项，同样如果待流转的环节含有条件序列流，在发起 /move 请求时必须包含可以执行条件判断的所有变量（封装于 `formData` 下的 `form_data` 中）。比如我们在上一章设计的流程图中起草环节有可能进行如下判断：

| 序列流指向     | 序列流条件   |
|:--------|:-------|
| `总院TMO审核`  | ${endApply == false && nextOU == "M"} |
| `A分院TMO审核` | ${endApply == false && nextOU == "A"} |
| `B分院TMO审核` | ${endApply == false && nextOU == "B"} |
| `结束环节`     | ${endApply == true} |

因此在发起这个环节的流转请求时我们或者给出 `endApply` 为 false 和 `nextOU` 的值；或者给出 `endApply` 为 true（此时可以不提供 `nextOU` 的值）。具体赋值的格式如下：

```java
formData={form_data:{endApply:false, nextOU:"M"}}
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
            "procInstId": "40013",
            "taskId": "40028"
        }
    ],
    "RetCode": "1",
    "RetVal": "1",
    "isEnd": "0"
}
```

内容格式和上一小节 `justStart` 返回的内容格式相同，值得注意的是，这次返回的是流程图中起草的后一个环节。

"form_data" 中传入的变量称为环节参与变量，这些变量都是局部变量即仅在当前环节有效，当前环节结束后就无法访问。如果您需要一个长期有效的变量，可以在环节参与变量名前加“$”,如：

```java
formData={form_data:{$endApply:false, $nextOU:"M"}}
```

这样环节参与变量“endApply”和“nextOU”就会在流程实例生命周期中一直有效，它的意义在“监听器”一章可以看到。

### [开始并移动（startMove）](#开始并流转（startMove）)
之前我们介绍了用于开始流程的 `justStart` 和用于流转的 `move`，但如果用户不想在第一个环节停留，我们还提供了 `startMove` 方法用于开始流程并通过第一个环节。这个方法可以看作是先 `justStart` 再 `move`，它的 api 格式如下：

```java
./wfservice/startMove?businessId=business2
                &assignee=ZhangSan
                &formData={form_data:{endApply:false, nextOU:"M"}}
```

其中，`businessId`（要启动的业务 Id）是必填项，`assignee`（环节参与者）和 `formData`（参与流转的流程变量）是选填项，但是如果待流转的环节含有条件序列流，在发起流转请求时必须包含可以执行条件判断的所有变量。它的返回和 `move` 操作的返回是相同的。

## [总结](#总结)
- 本章中介绍的所有 api 都支持 get 和 post 两种调用方式。

- 目前 `move`、`start` 和 `startMove` 已不推荐使用，建议用户替换为 `complete` 和 `begin`。

- 下一章将介绍进阶操作（并行网关、监听器等）