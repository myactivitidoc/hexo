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

### [开始（start）](#开始（start）)
之前我们介绍了用于开始流程的 `justStart` 和用于流转的 `flow`，但如果用户不想在第一个环节停留，我们还提供了 `start` 方法用于开始流程并通过第一个环节。这个方法可以看作是先 `justStart` 再 `flow`，它的 api 格式如下：

```java
./wfservice/start?businessId=business2
                &dealRole=projectManager
                &dealPerson=ZhangSan
                &formData={form_data:{endApply:false, nextOU:"M"}}
```

其中，`businessId`（要启动的业务 Id）、`dealRole`（参与者角色）和  `dealPerson`（参与者）是必填项，`formData`（参与流转的流程变量）是选填项，但是如果待流转的环节含有条件序列流，在发起流转请求时必须包含可以执行条件判断的所有变量。它的返回和 `flow` 操作的返回是相同的。

### [移动（move）](#移动（move）)
`move` 是我们推荐使用的流转方式，这种方式采用 `taskId` 作为流转关键参数，如前面所提 `taskId` 的特点是每次流转后都会改变，使用它作为关键参数可以实现明确指定当前环节（正在流转的环节）和避免多次抖动提交造成过度流转（流转到预期之外的环节）的需求。起草环节的流转 api 格式如下：

```java
./wfservice/move?taskId=40016
                &assignee=ZhangSan
                &formData={form_data:{endApply:false, nextOU:"M"}}
```

其中，`taskId`（任务 Id）是必填项，`assignee`（环节参与者）和 `formData`（参与流转的流程变量）是选填项，同样如果待流转的环节含有条件序列流，在发起 /move 请求时必须包含可以执行条件判断的所有变量（封装于 `formData` 下的 `form_data` 中）。

### [开始并移动（startMove）](#开始并流转（startMove）)

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
撤回是任务的来源人将任务取回的方法，它的 api 格式如下：

```java
./wfservice/withdraw?taskId=${此任务前一环节的任务变量ID}
```

其中 `taskId` 是必填项，撤回成功后会显示前一环节详细信息。撤回是一种异步方法，即不处于激活状态的环节发起的操作，所以撤回方需要提供当时保存的 `taskId`。如我们之前所说，`taskId` 是业务系统在每次流转返回后都需要自己保存的数据，即为异步方法调用之用。

并且，撤回的成功还取决于两个条件：任务未被下一个参与者完成，也未被下一个参与者领取，这是符合业务流转的逻辑的。

最后，撤回不具有回溯性，即如果 A 发任务给 B 然后 B 发任务给 C，如果 B 撤回后，A 无法把任务再撤回到自己这里。

## [软结束与消息事件](#软结束与消息事件)

### [软结束标志](#软结束标志)

在环节详细信息中，`isEnd` 标识此环节是不是当前流程的最后一个环节，这个规则既适用于主流程也是用于流程分支。但是，某些情况下用户的业务系统希望在一个非最后环节获得 `isEnd` 等于 1（结束）的效果，例如想让流程在这里“看上去”结束，在将来合适的时候再唤醒，就可以使用“软结束”来达到这个效果。

myActiviti 实现软结束的方式很简单，只需要将某个环节的输入区的 `文档` 设为 “end” 即可，如下：

{% asset_img soft_end.png %}

这样，当流转到这个环节时返回的 `isEnd` 就是 1（但其它信息不会改变），您的业务系统可以通过这一点做特别的事情。

### [消息触发事件（message）](#消息触发事件（message）)

通过消息触发任务是 BPMN 标准中一个异步触发任务的手段，这种方式在使用上非常灵活，myActiviti 对此也增加了微服务调用支持。

在流程图中画消息触发任务的方法很简单，将流程设计器左边树形菜单的 `边界事件-消息边界事件` 拖到一个用户任务上，就形成了消息触发任务。我们仍以立项流程图为例，现在将“结束环节”环节改为“软结束环节”，并在“软结束环节”上加一个消息触发事件来指向真正的结束环节，如下：

{% asset_img message_end.png %}

接下来我们配置这个消息触发事件的工作区。首先我们在流程图的全局工作区中增加一个消息名称，如下：

{% asset_img message_config.png %}

这里消息取名为 “finish”，实际可任意取名。然后消息触发事件配置如下（除 `消息名称` 外其它可任意设置）：

{% asset_img message_end_input2.png %}

这样，当业务系统流转到“软结束环节”时就会认为流程已经结束，而事实上流程却可以通过撤回“起死回生”，而这个流程想要真正结束的话，可以通过“触发消息”来实现。触发消息的 api 格式如下：

```java
./wfservice/message?exeId=${目标环节的流程实例的执行变量ID}
                   &message=${触发的消息名称}
```

具体到立项流程，如果 `message` 为 “finish”，则流程实例真正结束，返回如下：

```json
{
    "DataRows": null,
    "RetCode": "1",
    "RetVal": "1",
    "isEnd": "1"
}
```
## [总结](#总结)
- 本章中介绍的所有 api 都支持 get 和 post 两种调用方式。

- 本章介绍的方法参数格式都已经确定，下一章将介绍调用方式没有完全确定的进阶操作（并行网关、自由流等）