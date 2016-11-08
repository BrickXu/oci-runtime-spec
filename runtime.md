# 容器的作用域

除了访问控制问题，使用运行时（runtime）创建容器的实体必须能够针对同一容器使用本规范中定义的操作。其他实体使用相同或其他实例的运行时（runtime）是否可以看到该容器超出本规范的范围。

## State

容器的状态至少 **必须** 包含以下的属性：

* **`ociVersion`**: (string) 创建容器时用到的规范的版本号
* **`id`**: (string) 容器ID， **必须** 是本地唯一的，但不要求跨主机唯一。
* **`status`**: (string) 容器的运行时状态。 **可能** 是以下值中的一个:
  * `created`: 容器已经被创建，但是用户代码未被执行。
  * `running`: 容器被创建，且用户代码正在运行。
  * `stopped`: 容器被创建，用户代码执行完毕，但不在继续运行。

新的状态可以由运行时（runtime）自定义，但是它们 **必须** 用来表示未在上面定义的新运行时（runtime）状态。

* **`pid`**: (int) 主机可见的容器进程ID。
* **`bundlePath`**: (string) 容器bundle目录的绝对路径。提供给consumers查找容器的配置和rootfs。
* **`annotations`**: (map) 包含了容器相关的注解（我是有些懵逼的，没找到好的翻译）列表。如果没提供任何的注解，则 **可能** 不存在或是个空map。

当使用JSON序列化时，其格式 **必须** 遵守以下样式：

```
{
    "ociVersion": "0.2.0",
    "id": "oci-container1",
    "status": "running",
    "pid": 4422,
    "bundlePath": "/containers/redis",
    "annotations": {
        "myKey": "myValue"
    }
}
```

有关检索容器的状态信息，请看[query state](#query-state)。

## Lifecycle

生命周期描述了从容器被创建到停止退出之间的事件的时间线。

* 兼容OCI的运行时（runtime）[`create`](runtime.md#create)命令需要配合bundle文件地址和唯一ID调用。
* **必须** 通过[`config.json`](config.md)中的配置创建容器的运行时（runtime）环境，如果运行时（runtime）无法创建环境， **必须** 报错。虽然 **必须** 创建[`config.json`](config.md)中请求的资源，但用户代码不能在此时运行。此阶段后对[`config.json`](config.md)的任何更改 **一定不能** 影响到容器。
* Once the container is created additional actions MAY be performed based on the features the runtime chooses to support. However, some actions might only be available based on the current state of the container (e.g. only available while it is started).
* 运行时（runtime）的[`start`](runtime.md#start)命令需要配合容器的唯一ID调用。运行时（runtime） **必须** 通过指定的[`进程`](config.md#process-configuration)执行用户代码。
* 容器的进程停止， **可能** 是由于进程出错，退出，宕机或调用了运行时（runtime）的[`kill`](runtime.md#kill)命令。
* 运行时（runtime）的[`delete`](runtime.md#delete)命令需要配合容器的唯一ID调用。 必须回滚在create阶段期间执行的步骤销毁容器。

## Errors

（本规范列举的）操作报错的情况下， 本规范不强制要求具体实现（比如ruc）如何/是否返回或暴露错误给用户。除非另有说明，否则报错后 **必须** 重置环境的状态到操作没被执行过一样——除了像日志这种比较小的修改（你想改也改不回去）。

## Operations

除非底层的操作系统无法支持，否则兼容OCI的运行时（runtimes） **必须** 支持下列的操作。

这些操作不指定任何命令行API，并且这些参数是用于一般操作的入参。


### Query State

`state <container-id>`

此操作在不提供容器ID时 **必须** 报错。尝试查询一个不存在的容器 **必须** 报错。此操作 **必须** 返回同[`状态`](#state)一节指定的容器的状态（信息）。

### Create


`create <container-id> <path-to-bundle>`

此操作在不提供bundle文件的路径和容器ID时 **必须** 报错。如果提供了一个在runtime（管理的）范围内不唯一，或者无效的ID，其底层实现 **必须** 报错，且容易 **一定不能** 被创建。利用[`config.json`](config.md)文件里的数据，此操作 **必须** 创建一个新的容器，这意味着该容器相关的资源 **必须** 被创建，但是，用户指定的代码在此阶段 **一定不能** 被执行。如果运行时（runtime）不能依照如[`config.md`](config.md)指定的容器，则 **必须** 报错，且新容器 **一定不能** 被创建。

成功完成此操作后，此容器的`status`属性值 **必须** 是`created`。

运行时（runtime）在创建容器前（[步骤2](#lifecycle)），**可以** 根据此规范一般或相对于当前系统的能力验证[`config.json`](config.md)。
运行时（runtime）的调用者如果感兴趣pre-create校验，可以在此操作（create）前调用[bundle-validation tools](implementations.md#testing--tools)。

成功执行完成此操作后，容器 **必须** 被创建好。

再此操作执行成功之后，任何针对[`config.json`](config.md)的修改都不会影响到（创建出来的）容器。

### Start

`start <container-id>`

此操作在不提供容器ID时 **必须** 报错。尝试启动一个不存在的容器 **必须** 报错。尝试启动一个已经运行的容器 **必须** 没有任何效果，且 **必须** 报错。此操作 **必须** 由[`进程`](config.md#process-configuration)运行用户代码。

成功执行完成此操作后，此容器的[`status`](#state)属性值 **必须** 是`running`。

### Kill


`kill <container-id> <signal>`

此操作在不提供容器ID时 **必须** 报错。尝试向一个非运行的容器发送信号量， **必须** 没有任何效果，并且 **必须** 报错。此操作 **必须** 向容器内的进程发送指定的信号量。

当容器内的进程停止了，不管是`kill`操作还是其他原因导致的，这个容器的[`status`](#state)属性值 **必须** 是`stopped`。

### Delete

`delete <container-id>`

此操作在不提供容器ID时 **必须** 报错。尝试删除一个不存在容器 **必须** 报错。尝试删除一个其进程仍然运行的容器 **必须** 报错。删除容器 **必须** 释放其[`创建`](#create)时申请的资源。注意，非容器创建的， 但是相关的资源， **一定不能** 被删除。一旦容器被删除，它的ID **可能** 分配给后续的容器使用。

## 钩子（Hooks）

在本规范中提到的许多操作，都允许在每个操作之前或之后采取附加动作的“钩子”。更多“钩子”的信息请请参考[运行时配置](./config.md#hooks)。

## 引用

https://github.com/opencontainers/runtime-spec/blob/master/runtime.md
