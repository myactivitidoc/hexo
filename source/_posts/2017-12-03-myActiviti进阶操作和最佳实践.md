---
layout: post
title: myActiviti 进阶操作和最佳实践
description: 本章内容向您讲解 myActiviti 高级操作和最佳实践。
category: 工作流
date: 2017-12-03
list_number: false
---

## [进阶操作](#进阶操作)

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
            "procInstId": "40130",
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
            "procInstId": "40130",
            "taskId": "40147"
        }
    ],
    "RetCode": "1",
    "RetVal": "1",
    "isEnd": "0"
}
```

值得注意的是，`DataRows` 中出现两个流程分支，这两个分支都可以进行流转，同时 `exeId` 不再与 `procInstId` 相同；并且当流转到收集点时，先到的流程分支会返回：

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
            "procInstId": "40130",
            "taskId": "40152"
        }
    ],
    "RetCode": "1",
    "RetVal": "1",
    "isEnd": "0"
}
```

即两条分支收成了一条，以上两条分支无论谁先谁后都是如此。

### [监听器](#监听器)
监听器是 myActiviti 的一种高级特性，它可以极大简化调用工作流 api 解决实际问题的代码，同时极大的提高您软件的生产力。

当您使用工作流解决实际问题时，您进行每一次流转需要的并不仅仅是流程数据，还会用到人员数据、组织机构数据、角色数据、特定业务数据……等等。这些数据有一个统一特点，就是它们和流程环节有密切的相关性，如果您仅仅调用工作流 api，就可以在得到流程数据的同时获得当前环节相关的人员数据、组织机构数据等，将大大提高开发效率，这就是监听器的由来。

在 myActiviti 中使用监听器的方法很简单，在流程图中相关环节上点击“执行监听器”或“任务监听器”即可（两者可选择的生命周期不同），这里以“执行监听器”为例：

{% asset_img modeler_listener_2.png %}

之后进入“执行监听器”设置界面，点击如下图所示的“+”号建立监听器：

{% asset_img modeler_listener2_8.png %}

之后，增加一个事件为 <b>start</b>，委托表达式为 `commonStartListener` 的监听器。
（这里我们选择监听 start 事件，它得到的数据会在返回结果中以 `startListener` 属性显示）

{% asset_img modeler_listener6_3.png %}

因为 `commonStartListener` 监听器的要求，我们需要设置一个名为 “type” 的变量，它可以是 <b>系统保留值 “form_data”</b> 之外的 <b>任何值</b>，这里我们将它设为 “act_person”:

{% asset_img modeler_listener7_2.png %}

之后，同样因为 `commonStartListener` 监听器的要求，我们还需要为此监听器设置一个名为 “path” 的变量，为此点选左下方的“+”号，输入名称 “path” 和字符串值 “microservice/serviceauth/p_getusersbyroleid”（一个微服务地址），如下图：

{% asset_img modeler_listener8_2.png %}

最后点选 `保存`，这个监听器就生效了。以后再用 api 流转这个环节时，在 `listener` 中加入 “act_person” 内容和参数，例如：

```java
listener={act_person:{arg_roleid:"046ab6429ce611e7ad99008cfa042288"}}
```

如上所示，相关监听器就会自动调用微服务：

```java
microservice/serviceauth/p_getusersbyroleid?arg_roleid=046ab6429ce611e7ad99008cfa042288
```
（前面的 ip 部分可以不写，myActiviti 会根据当前环境自动处理）

调用后得到的数据会在返回结果中以 “startListener.act_person” 属性显示，如下：

```
{
    "DataRows": [
        {
            "actId": "tmoCheak",
            "actName": "总院TMO审核",
            "startListener": {
                "act_person": {
                    "RetVal": "1",
                    "RetCode": "1",
                    "RowCount": "6",
                    "DataRows": [{
                        ......
                    }]
                }
            },
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

其中，“startListener” 是封装所有 start 事件监听器返回值的固定属性，只在该事件监听器有返回值时才会出现。而 “act_person” 属性名是由监听器的 “type” 值决定的，因此您可以通过在一个环节中添加多个 `commonStartListener` 监听器的方式来调用多个微服务（按定义的顺序执行），只要它们的 “type” 值不同即可。

“执行监听器” 中除 start 事件之外还有其它的事件可监控，并且 “执行监听器” 之外还有 “任务监听器”，它们的对应关系如下：

| 监听器     | 事件   | 回传封装属性   | 委托表达式   |
|:--------|:-------|:-------|:-------|
| 执行监听器  | `start` | startListener | commonStartListener |
| 执行监听器  | `take` | 无 | 当前无法触发 |
| 执行监听器  | `end` | endListener | commonEndListener |
| 任务监听器  | `create` | createListener | commonCreateListener |
| 任务监听器  | `assignment` | assignmentListener | commonAssignmentListener |
| 任务监听器  | `complete` | completeListener | commonCompleteListener |

在执行监听器指向的 API 时，有时我们需要传入工作流本身的变量（如 taskId、actName 等），可这些变量在流转前用户无法获得，因此我们增加了一些内置参数来表示这些变量，如：

```java
listener={act_person:{arg_roleid:"046ab6429ce611e7ad99008cfa042288", arg_task_id:"${taskId}", arg_act_name:"${actName}"}}
```

这样在调用 API 时 arg_task_id 的值为当前 `taskId`，arg_act_name 的值为当前的 `actName`。

全部内置参数如下：

| 参数名     | 实际值   |
|:--------|:-------|:-------|:-------|
| ${actId}  | 当前的环节id |
| ${actName}  | 当前的环节名称 |
| ${exeId}  | 当前的执行变量id |
| ${procDefId}  | 当前的流程定义id |
| ${procInstId}  | 当前的流程实例id |
| ${taskId}  | 当前的任务id |
| ${taskName}  | 当前的任务名称 |
| $${foo}  | 环节参与变量foo的值 |

最后一个内置参数比较特殊，它的名称中 foo 可以是任何值，这样就可以把您需要的任何由 “form” 引入的变量带到 API 中。

## [最佳实践](#最佳实践)
- 单入口，单出口。虽然工作流支持多个结束环节，但不推荐这样画，建议每个图都有唯一的结束环节。

- 先复制，后变更。当流程图贴近现实场景后，会变得非常复杂，每个流程图的工作量可以达到人天级别，此时编辑流程时防止“手滑”变得至关重要（比如以为元素粘上了实际没有粘上）,尤其是对一些生产环境的流程进行变更时更是如此，所以一定要培养勤备份的习惯。

- 测试环境设计，生产环境导入。一般工作流至少有生产环境和测试环境两套。无论是新增流程还是修改流程，都应该先在测试环境设计、变更、测试，无问题后导出工作流文件，再在生产环境导入工作流文件。切记不要在生产环境进行设计。

- 合理使用监听器可以节省您大量的代码。