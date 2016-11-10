# Glossary

## Bundle

一个提前写好的[目录结构](bundle.md)，可以被分发，提供给运行时创建[container](#container)并启动内部的进程。

## Configuration

[bundle](#bundle)中的[`config.json`](config.md)文件定义了预期的容器和容器进程。

## Container

执行具有可配置的隔离性和资源限制的进程的环境。
例如，命名空间，资源限制和挂载信息都是容器环境的一部分。

## Container namespace

On Linux, a leaf in the [namespace][namespaces.7] hierarchy in which the [configured process](config.md#process-configuration) executes.

## JSON

所有配置的[JSON][] **必须** 经过 [UTF-8][] 编码。
JSON对象 **一定不能** 有重复的名字。
JSON对象中的条目顺序不重要。

## Runtime

本规范的一个（编程）实现。
它从[bundle](#bundle)中读取[配置文件](#configuration)，使用其信息创建[container](#container)，在容器内启动进程，以及执行[生命周期动作](runtime.md)。

## Runtime namespace

On Linux, a leaf in the [namespace][namespaces.7] hierarchy from which the [runtime](#runtime) process is executed.
New container namespaces will be created as children of the runtime namespaces.

[JSON]: https://tools.ietf.org/html/rfc7159
[UTF-8]: http://www.unicode.org/versions/Unicode8.0.0/ch03.pdf
[namespaces.7]: http://man7.org/linux/man-pages/man7/namespaces.7.html
