---
layout: post
title: myActiviti 进阶操作和最佳实践
description: 本章内容向您讲解 myActiviti 高级操作和最佳实践。
category: 工作流
date: 2017-12-03
list_number: false
---

## [进阶操作](#进阶操作)

### [并行网关](#并行网关)
有时候我们需要多个流程分支并行流转，myActiviti 可以通过并行网关实现这样的效果。并行网关的形状是一个菱形中间加一个 “+”，效果是它发散的多条序列流中每一条都可以独立流转；一般一个并行网关发散之后还会有另一个并行网关作为收集，例如下图：

{% asset_img showParallel.png %}

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

值得注意的是，`DataRows` 中出现两个流程分支，这两个分支都可以进行流转，同时 `exeId` 不再与 `procInstId` 相同；并且当流转到收集点时，先到的流程分支会返回（假设A分院处理完毕）：

```json
{
    "DataRows": [
        {
            "actId": "parallelClose",
            "actName": "并行网关结束",
            "actRole": null,
            "document": null,
            "exeId": "40141",
            "isEnd": "0",
            "procInstId": "40130",
            "taskId": null,
            "type": "PARALLEL_GATEWAY"
        }
    ],
    "RetCode": "1",
    "RetVal": "1",
    "extras": [
        {
            "actId": "parallelClose",
            "actName": "并行网关结束",
            "actRole": null,
            "document": null,
            "exeId": "40141",
            "isEnd": "0",
            "procInstId": "40130",
            "taskId": null,
            "type": "PARALLEL_GATEWAY"
        }
    ],
    "isEnd": "0"
}
```
其中 `parallelClose` 是第二个并行网关的 actId，对于这个结果可以通过 `DataRows[0].taskId` 为 null 来确认并发网关并没有结束。

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

### [包容网关](#包容网关)
包容网关是融合了并行网关和互斥网关特性的网关，它可以生成并行分支（并行网关特性），也可以在分支序列流中写判断条件（互斥网关特性），例如我们把上面流程图中的并行网关变为包容网关，如下：

{% asset_img showInclusive.png %}

如果我们在指向 `A分院处理` 的序列流上加条件 `${useA == true}`，在指向 `B分院处理` 的序列流上加条件 `${useB == true}`，则在流转 `请各分院处理` 环节时，如果环节变量 `useA` 为 true 则流转 `A分院处理`，`useB` 为 true 则流转 `B分院处理`，两者都为 true 则都流转。
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
| ${lastTaskId}  | 前一任务的id |
| ${isWithdraw}  | 是否是撤回操作，默认为false，当执行withdraw操作时此值为true |
| $${foo}  | 环节参与变量foo的值 |

最后一个内置参数比较特殊，它的名称中 foo 可以是任何值，这样就可以把您需要的任何由 “form” 引入的变量带到 API 中。

### [调用子流程](#调用子流程)
工作流中在任何一个流程都可以以子流程的方式调用其它的流程，而且实现起来非常简单。

首先，我们建立一个简单的审批流程，这个流程只有申请人提交和项目负责人审批两个环节，大多数流转都开始于这两个环节，它的模型如下：

{% asset_img call_sub_2.png %}

它的全局属性设置见下，注意它的流程ID为 “pm_approval”：

{% asset_img call_sub_3.png %}

将上面这个模型部署成功后，我们再建立一个调用这个流程的流程，它的模型如下：

{% asset_img call_sub_4.png %}

注意上图中 “开始” 环节后的第一个环节，它的类型不是 `用户任务` 而是 `调用子流程`，在左侧树形菜单中选择它的方式如下：

{% asset_img call_sub_1.png %}

这个 `调用子流程` 的设置如下，值得注意的是 `流程参考元素` 要对应您想调用的流程的<b>流程ID</b>，也就是 “pm_approval”：

{% asset_img call_sub_6.png %}

我们可以称这个流程为 “父流程”，它的全局属性设置见下：

{% asset_img call_sub_5.png %}

我们将它部署成功后，对父流程执行 `begin` 操作，之后流程实例动态图如下：

{% asset_img call_sub_7.png %}

而对于 “调用项目经理审批流程” 环节，它的流程实例动态图如下：

{% asset_img call_sub_8.png %}

对含有子流程的流程的api调用和普通流程完全一致，只是在返回时会有一些额外数据 `extras`，如下：

```
{
    "DataRows": [
        {
            "actId": "apply_person",
            "actName": "申请人提交",
            "actRole": null,
            "document": null,
            "exeId": "111486064631214080",
            "isEnd": "0",
            "procInstId": "111486064631214080",
            "taskId": "111486069769236480"
        }
    ],
    "RetCode": "1",
    "RetVal": "1",
    "extras": [
        {
            "actId": "call_pm_approval",
            "actName": "调用项目经理审批流程",
            "actRole": null,
            "document": null,
            "exeId": "111486064610242560",
            "isEnd": "0",
            "procInstId": "111486064606048256",
            "taskId": null,
            "type": "CALL_ACTIVITY_EXECUTE"
        }
    ],
    "isEnd": "0"
}
```

下一步流转使用 `DataRows` 中 `taskId` 的值即可，而 `extras` 中的内容是流转过程中自动经过的元素。例如上图中的元素是 `调用子流程` 这个环节本身，它的 “type” 为 “CALL_ACTIVITY_EXECUTE” 也说明了这一点。

最后，调用子流程是一个可重复过程，即子流程里可以继续调用子流程。

### [调用子流程进阶](#调用子流程进阶)

在前面文章中我们介绍了调用子流程方法，在本篇文章我们介绍子流程调用时监听器和变量传递问题。

在父流程进入子流程时和子流程结束返回父流程时都可以添加监听器，监听器的返回结果会附在对应的元素上，按实际情况有些监听器的返回结果也可能附在 `extras` 里的元素上。

在 `调用子流程` 的属性设置中，还有 `输入参数` 和 `输出参数` 两项内容，前者用于子流程开始时把父流程的变量传递给子流程，后者用于子流程结束时把子流程的变量传递给父流程。

{% asset_img call_sub_11.png %}

点击 `输入参数` 可设定父流程传递给子流程的参数，如下：

{% asset_img call_sub_9.png %}

上图中 `资源` 为父流程中的变量名称，`目标` 为子流程中的变量名称，`资源表达式` 可忽略，变量映射可以是多个。

点击 `输入参数` 可设定子流程传递给父流程的参数，如下：

{% asset_img call_sub_10.png %}

上图中 `资源` 为子流程中的变量名称，`目标` 为父流程中的变量名称，`资源表达式` 可忽略，同样的变量映射可以是多个。

将流程图保存并部署后，就可以在调用时在子流程中使用父流程的变量或在父流程中使用子流程的变量。

### [高级邮件任务](#高级邮件任务)

在前面我们介绍了使用邮件任务发送邮件的方法，实际上邮件任务还可以以模板的方式发送，具体方式为在邮件任务的“文档”中输入模板，例如：

{% asset_img advanced_email.png %}

当“文档”中有内容时，工作流会把其中的内容作为邮件模板，同时会忽略邮件任务中的“邮件文本内容”、“邮件Html内容”以及“邮件接收方”，您在流转这个邮件任务时需要提供环节参与变量来确定邮件接收方和邮件模板中的变量值。例如按如下方式调用（流程的businessId已部署为myEmail）：

```java
./wfservice/begin?businessId=myEmail
    &form={email:"zhang3@test.org",name:"张三"}

```
（注意：如果在浏览器中调用这个服务，需要将 `{` 转义为 `%7B`，将 `}` 转义为 `%7D`。）

这样一来，内容为 “张三您好，欢迎使用<b>工作流平台</b>！” 的邮件就会发送到 “zhang3@test.org” 邮箱中。

环节参与变量中的 `email` 是必需的，它用来指定邮件接收方。可以在 `email` 中指定多个邮箱并用 “;” 或 “,” 分隔，这样就能一次给多个邮箱群发邮件。

### [特殊调用参数](#特殊调用参数)

在用户调用工作流（如 `begin`）时有些参数会起到特殊的做用，这些参数都以 "_" 开头，如下所示：

| 参数名     | 实际值   |
|:--------|:-------|:-------|:-------|
| _softTransaction | 当值为 `true` 时对监听器结果进行事务回滚，为其它值时对监听器结果不进行事务回滚 |
| _content | 当值为 `jsonEnumUseName` 时回传 json 中的 `isEnd` 值为 `NO` 或 `YES`，为其它值时回传 json 中的 `isEnd` 值为 `0` 或 `1`|

例如，当用户启动一个流程，同时开启它 `softTransaction:true` 和 `_content:jsonEnumUseName` 性质时，只需要像下面这样调用：

```
./wfservice/begin?businessId=xxx
                 &_softTransaction=true
                 &_content=jsonEnumUseName
                 &form=...
```

## [最佳实践](#最佳实践)
- 单入口，单出口。虽然工作流支持多个结束环节，但不推荐这样画，建议每个图都有唯一的结束环节。

- 先复制，后变更。当流程图贴近现实场景后，会变得非常复杂，每个流程图的工作量可以达到人天级别，此时编辑流程时防止“手滑”变得至关重要（比如以为元素粘上了实际没有粘上）,尤其是对一些生产环境的流程进行变更时更是如此，所以一定要培养勤备份的习惯。

- 测试环境设计，生产环境导入。一般工作流至少有生产环境和测试环境两套。无论是新增流程还是修改流程，都应该先在测试环境设计、变更、测试，无问题后导出工作流文件，再在生产环境导入工作流文件。切记不要在生产环境进行设计。

- 合理使用监听器可以节省您大量的代码。