# 容器配置文件

容器的顶级目录 **必须** 包含一个名为`config.json`的配置文件。
本文档中定义的是典型的结构，但是还有JSON结构[`schema/config-schema.json`](schema/config-schema.json)和Go版本的[`specs-go/config.go`](specs-go/config.go)。

配置文件包含对容器执行标准操作所需的元数据。这包括运行的进程，要注入的环境变量，要使用的沙箱功能等。

下面是配置文件中定义的每个字段的详细描述。

## Specification version

* **`ociVersion`** (string, REQUIRED) **必须** 是符合[SemVer v2.0.0](http://semver.org/spec/v2.0.0.html)格式的，并且指定bundle符合的开放容器运行时规范的版本。开放容器运行时规范遵守语义化版本并保证major版本的向前/后兼容性。例如，如果一个配置兼容本规范的1.1版本，则它必须兼容所有支持1.1和后续版本的运行时（runtime），但是不兼容支持1.0的运行时（runtime）。

### Example

```json
    "ociVersion": "0.1.0"
```

## Root Configuration

**`root`** (object, REQUIRED) 配置容器的rootfs。

* **`path`** (string, REQUIRED) 指定容器的rootfs路径。
  路径可以是绝对路径（以/开头），也可以是相对于bundle的相对路径（不以/开头）。
  For example (Linux), with a bundle at `/to/bundle` and a root filesystem at `/to/bundle/rootfs`, the `path` value can be either `/to/bundle/rootfs` or `rootfs`.
  举个例子（Linux）， bundle的目录是`/to/bundle`，rootfs的目录是`/to/bundle/rootfs`，那么`path`的值可以写成`/to/bundle/rootf`（绝对路径写法）或rootfs（相对路径写法）。
  path内填写的目录 **必须** 存在。
* **`readonly`** (bool, OPTIONAL) 设置成true则容器内的rootfs **必须** 是只读的，默认是false。

### Example

```json
"root": {
    "path": "rootfs",
    "readonly": true
}
```

## Mounts

**`mounts`** (array, OPTIONAL) 配置额外的mount(在[`root`](#root-configuration)之上)。
运行时（runtime）**必须** 按列表顺序mount。
参数与[the Linux mount system call](http://man7.org/linux/man-pages/man2/mount.2.html)类似。

* **`destination`** (string, REQUIRED) 挂载点的目标地址：容器内的路径。
  This value MUST be an absolute path.
  此指**必须**是绝对路径。
  Windows系统下，挂载点的目标地址 **一定不能** 和另外一个挂载点嵌套，例如c:\\foo和c:\\foo\\bar。
* **`type`** (string, REQUIRED) 需要mount的文件系统类型。
  Linux：支持*/proc/filesystems*中列举的文件系统类型（如“minix”，“ext2”，“ext3”，“jfs”，“xfs”，“reiserfs”，”msdos“，”proc“，”nfs“, "iso9660"）。
  Windows：ntfs。
* **`source`** (string, REQUIRED) 设备名，也可以是目录名或虚拟设备。
  Windows：挂载点目标的卷名，\\?\Volume\{GUID}\（Windows下source叫做target）。
* **`options`** (list of strings, OPTIONAL) 文件系统mount参数。
  Linux: [supported][mount.8-filesystem-independent] [options][mount.8-filesystem-specific] are listed in [mount(8)][mount.8].

### Example (Linux)

```json
"mounts": [
    {
        "destination": "/tmp",
        "type": "tmpfs",
        "source": "tmpfs",
        "options": ["nosuid","strictatime","mode=755","size=65536k"]
    },
    {
        "destination": "/data",
        "type": "bind",
        "source": "/volumes/testing",
        "options": ["rbind","rw"]
    }
]
```

### Example (Windows)

```json
"mounts": [
    "myfancymountpoint": {
        "destination": "C:\\Users\\crosbymichael\\My Fancy Mount Point\\",
        "type": "ntfs",
        "source": "\\\\?\\Volume\\{2eca078d-5cbc-43d3-aff8-7e8511f60d0e}\\",
        "options": []
    }
]
```

有关详细信息，请参考[mountvol](http://ss64.com/nt/mountvol.html)和 [SetVolumeMountPoint](https://msdn.microsoft.com/en-us/library/windows/desktop/aa365561(v=vs.85).aspx)（Windows）。


## Process configuration

**`process`** (object, REQUIRED) 配置容器的进程。

* **`terminal`** (bool, optional) 指定是否需要连接该进程的终端，默认是false。
  On Linux, a pseudoterminal pair is allocated for the container process and the pseudoterminal slave is duplicated on the container process's [standard streams][stdin.3].
* **`consoleSize`** (object, OPTIONAL) 指定终端的控制台窗口大小（如果连接了终端），包含以下属性：
  * **`height`** (uint, REQUIRED)
  * **`width`** (uint, REQUIRED)
* **`cwd`** (string, REQUIRED) 为可执行文件设置工作目录。此值 **必须** 是绝对路径。
* **`env`** (array of strings, OPTIONAL) 包含一组在进程执行前设置到其环境中的变量。数组里的元素指定为“KEY=value”格式的字符串。左侧（KEY） **必须** 只包含字母，数字和下划线`_`，如[IEEE Std 1003.1-2001](http://pubs.opengroup.org/onlinepubs/009695399/basedefs/xbd_chap08.html).
* **`args`** (array of strings, REQUIRED) 可执行文件和一组标志（flags）。可执行文件是第一个数组的第一个元素，并且 **必须** 存在于给定的路径里。如果可执行文件的路径不是绝对路径，则通过$PATH查找可执行文件。

对于基于Linux的系统，进程结构支持以下特定字段：

* **`capabilities`** (array of strings, OPTIONAL) capabilities是一个数组，用于指定可以提供给容器内部进程的capabilities。有效值均是字符串，且定义在[the man page](http://man7.org/linux/man-pages/man7/capabilities.7.html)。
* **`rlimits`** (array of rlimits, OPTIONAL) rlimits is an array of rlimits that allows setting resource limits for a process inside the container.
The kernel enforces the `soft` limit for a resource while the `hard` limit acts as a ceiling for that value that could be set by an unprivileged process.
Valid values for the 'type' field are the resources defined in [the man page](http://man7.org/linux/man-pages/man2/setrlimit.2.html).
* **`apparmorProfile`** (string, OPTIONAL) 指定容器使用的apparmor配置文件名字。有关Apparmor的更多信息，请参考[Apparmor documentation](https://wiki.ubuntu.com/AppArmor)。
* **`selinuxLabel`** (string, OPTIONAL) 指定容器内进程运行时使用的SELinux标签。有关SELinux的更多信息，请参考[Selinux documentation](http://selinuxproject.org/page/Main_Page)。
* **`noNewPrivileges`** (bool, OPTIONAL) 将noNewPrivileges设置成true可以阻止容器内的进程申请额外的权限。
[The kernel doc](https://www.kernel.org/doc/Documentation/prctl/no_new_privs.txt)上有更多关于该功能如何使用prctl系统调用实现的信息。

### User

进程的用户是平台特定的结构，允许控制进程以哪个具体的用户运行。

#### Linux and Solaris User

基于Linux和Solaris的系统，用户结构有以下字段：

* **`uid`** (int, REQUIRED) 指定[container namespace][container-namespace]内的UID。
* **`gid`** (int, REQUIRED) 指定[container namespace][container-namespace]内的GID.
* **`additionalGids`** (array of ints, OPTIONAL) 指定添加给进程的额外的GID(在 [container namespace][container-namespace]内)。

_Note: 分别用于uid和gid的symbolic name，例如uname和game，留给更高级去导出（如解析/etcpasswd，NSS等）。_

_Note: Solaris下，uid和gid指定容器内进程的uid和gid，并且不需要和宿主机一致。_

### Example (Linux)

```json
"process": {
    "terminal": true,
    "consoleSize": {
        "height": 25,
        "width": 80
    },
    "user": {
        "uid": 1,
        "gid": 1,
        "additionalGids": [5, 6]
    },
    "env": [
        "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
        "TERM=xterm"
    ],
    "cwd": "/root",
    "args": [
        "sh"
    ],
    "apparmorProfile": "acme_secure_profile",
    "selinuxLabel": "system_u:system_r:svirt_lxc_net_t:s0:c124,c675",
    "noNewPrivileges": true,
    "capabilities": [
        "CAP_AUDIT_WRITE",
        "CAP_KILL",
        "CAP_NET_BIND_SERVICE"
    ],
    "rlimits": [
        {
            "type": "RLIMIT_NOFILE",
            "hard": 1024,
            "soft": 1024
        }
    ]
}
```
### Example (Solaris)

```json
"process": {
    "terminal": true,
    "consoleSize": {
        "height": 25,
        "width": 80
    },
    "user": {
        "uid": 1,
        "gid": 1,
        "additionalGids": [2, 8]
    },
    "env": [
        "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
        "TERM=xterm"
    ],
    "cwd": "/root",
    "args": [
        "/usr/bin/bash"
    ]
}
```

#### Windows User

基于Windows的系统，用户结构有以下的字段：

* **`username`** (string, OPTIONAL) 指定进程的用户名。

### Example (Windows)

```json
"process": {
    "terminal": true,
    "user": {
        "username": "containeradministrator"
    },
    "env": [
        "VARIABLE=1"
    ],
    "cwd": "c:\\foo",
    "args": [
        "someapp.exe",
    ]
}
```


## Hostname

* **`hostname`** (string, OPTIONAL) 配置容器中运行的进程能看到的容器主机名。Linux环境下，你只能在创建了新的[UTS namespace][uts-namespace]后才能设置此值。

### Example

```json
"hostname": "mrsdalloway"
```

## Platform

**`platform`** 指定了配置的目标平台。

* **`os`** (string, REQUIRED) 指定这个镜像对应的操作系统系列。
  如果运行时（runtime）不支持配置的 **`os`** **必须** 报错。
  Bundles SHOULD use, and runtimes SHOULD understand, **`os`** entries listed in the Go Language document for [`$GOOS`][go-environment].
  如果某个操作系统没有列在`$GOOS`的文档里，那么它 **应该** 被提交到这个规范里做标准化。
* **`arch`** (string, REQUIRED) 指定了镜像内二进制文件编译用到的指令集。
  如果运行时（runtime）不支持配置的 **`arch`** **必须** 报错。
  Values for **`arch`** SHOULD use, and runtimes SHOULD understand, **`arch`** entries listed in the Go Language document for [`$GOARCH`][go-environment].
  如果某个架构没有列在`$GOARCH`文档里，那么它 **应该** 被提交到这个规范里做标准化。

### Example

```json
"platform": {
    "os": "linux",
    "arch": "amd64"
}
```

## Platform-specific configuration

[**`platform.os`**](#platform) 用于检查进一步的平台相关的配置。

* **`linux`** (object, OPTIONAL) [Linux-specific configuration](config-linux.md).
  仅当 **`platform.os`** 是`linux`时才推荐设置。
* **`solaris`** (object, OPTIONAL) [Solaris-specific configuration](config-solaris.md).
  仅当 **`platform.os`** 是`solaris`时才推荐设置。
* **`windows`** (object, optional) [Windows-specific configuration](config-windows.md).
  仅当 **`platform.os`** 是`windows`时才推荐设置。

### Example (Linux)

```json
{
    "platform": {
        "os": "linux",
        "arch": "amd64"
    },
    "linux": {
        "namespaces": [
          {
            "type": "pid"
          }
        ]
    }
}
```

## Hooks

**`hooks`** (object, OPTIONAL) 用来配置容器生命周期事件的回调。
Liftcycle hooks允许对容器运行时（runtime）的许多（逻辑）点自定义事件。
目前有`Prestart`，`Poststart`和`Poststop`。

* [`Prestart`](#prestart) 容器进程运行前被调用的一组钩子。
* [`Poststart`](#poststart) 容器进程运行后立即被调用的一组钩子。
* [`Poststop`](#poststop) 容器进程退出后被调用的一组钩子。

钩子允许在容器的各种生命周期事件之前/之后运行代码。钩子 **必须** 按照列表顺序（声明顺序）被调用。容器的状态会通过stdin传入到钩子内，这样钩子（程序）就可以拿到它们工作需要的信息。

钩子的路径是绝对路径，并且可以在[runtime namespace][runtime-namespace]中执行。

### Prestart

pre-start钩子在容器进程产生后，用户提供的命令执行之前被调用。Linux下，他们在容器的namespaces创建以后被调用，因此提供了自定义容器的机会。举个例子， 在此钩子里可以配置network namespace。

如果钩子返回了非零的退出码（参考shell中的$?），则一个包含了退出码和stderr的错误被返回给调用者，同时容器被回收。

### Poststart

post-start钩子在用户进程运行启动后被调用。举个例子，这个钩子可以通知用户真正产生的进程（的PID或其他balabala）。

如果钩子返回了非零的退出码（参考shell中的$?），记录下错误并继续执行剩下的钩子。


### Poststop

post-stop钩子在容器进程停止后被调用。可以在此钩子中执行清理或调试工作。

如果钩子返回了非零的退出码（参考shell中的$?），记录下错误并继续执行剩下的钩子。


### Example

```json
    "hooks": {
        "prestart": [
            {
                "path": "/usr/bin/fix-mounts",
                "args": ["fix-mounts", "arg1", "arg2"],
                "env":  [ "key1=value1"]
            },
            {
                "path": "/usr/bin/setup-network"
            }
        ],
        "poststart": [
            {
                "path": "/usr/bin/notify-start",
                "timeout": 5
            }
        ],
        "poststop": [
            {
                "path": "/usr/sbin/cleanup.sh",
                "args": ["cleanup.sh", "-f"]
            }
        ]
    }
```

`path` is REQUIRED for a hook.
`args` and `env` are OPTIONAL.
`timeout` is the number of seconds before aborting the hook.
The semantics are the same as `Path`, `Args` and `Env` in [golang Cmd](https://golang.org/pkg/os/exec/#Cmd).

## Annotations

**`annotations`** (object, OPTIONAL) 包含了容器的任意元数据。
这些元数据信息 **可能** 是结构化的或非结构化的。
注解 **必须** 是键值对映射，而且键值 **必须** 是字符串。
值 **必须** 存在，但 **可能** 是空字符串（Java对照HashMap理解）。
键在映射中 **必须** 是唯一的，最好的办法就是使用命名空间。
键 **应该** 以反向域名的方式命名 —— 比如 `com.example.myKey`。
`org.opencontainers`命名空间下的键是保留关键字，并且 **一定不能** 被后续的规范使用。
如果没有注解，此值 **可能** 不存在或者是空映射。
正在读取/处理配置文件的（runtime）实现 **一定不能** 在遇到一个未知属性时报错。

```json
"annotations": {
    "com.example.gpu-cores": "2"
}
```

## Extensibility
正在读取/处理配置文件的（runtime）实现 **一定不能** 在遇到一个未知属性时报错，而是 **必须** 忽略这些未知的属性。

## Configuration Schema Example

这是一个可供参考的完整的`config.json`例子。

```json
{
    "ociVersion": "0.5.0-dev",
    "platform": {
        "os": "linux",
        "arch": "amd64"
    },
    "process": {
        "terminal": true,
        "user": {
            "uid": 1,
            "gid": 1,
            "additionalGids": [
                5,
                6
            ]
        },
        "args": [
            "sh"
        ],
        "env": [
            "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
            "TERM=xterm"
        ],
        "cwd": "/",
        "capabilities": [
            "CAP_AUDIT_WRITE",
            "CAP_KILL",
            "CAP_NET_BIND_SERVICE"
        ],
        "rlimits": [
            {
                "type": "RLIMIT_CORE",
                "hard": 1024,
                "soft": 1024
            },
            {
                "type": "RLIMIT_NOFILE",
                "hard": 1024,
                "soft": 1024
            }
        ],
        "apparmorProfile": "acme_secure_profile",
        "selinuxLabel": "system_u:system_r:svirt_lxc_net_t:s0:c124,c675",
        "noNewPrivileges": true
    },
    "root": {
        "path": "rootfs",
        "readonly": true
    },
    "hostname": "slartibartfast",
    "mounts": [
        {
            "destination": "/proc",
            "type": "proc",
            "source": "proc"
        },
        {
            "destination": "/dev",
            "type": "tmpfs",
            "source": "tmpfs",
            "options": [
                "nosuid",
                "strictatime",
                "mode=755",
                "size=65536k"
            ]
        },
        {
            "destination": "/dev/pts",
            "type": "devpts",
            "source": "devpts",
            "options": [
                "nosuid",
                "noexec",
                "newinstance",
                "ptmxmode=0666",
                "mode=0620",
                "gid=5"
            ]
        },
        {
            "destination": "/dev/shm",
            "type": "tmpfs",
            "source": "shm",
            "options": [
                "nosuid",
                "noexec",
                "nodev",
                "mode=1777",
                "size=65536k"
            ]
        },
        {
            "destination": "/dev/mqueue",
            "type": "mqueue",
            "source": "mqueue",
            "options": [
                "nosuid",
                "noexec",
                "nodev"
            ]
        },
        {
            "destination": "/sys",
            "type": "sysfs",
            "source": "sysfs",
            "options": [
                "nosuid",
                "noexec",
                "nodev"
            ]
        },
        {
            "destination": "/sys/fs/cgroup",
            "type": "cgroup",
            "source": "cgroup",
            "options": [
                "nosuid",
                "noexec",
                "nodev",
                "relatime",
                "ro"
            ]
        }
    ],
    "hooks": {
        "prestart": [
            {
                "path": "/usr/bin/fix-mounts",
                "args": [
                    "fix-mounts",
                    "arg1",
                    "arg2"
                ],
                "env": [
                    "key1=value1"
                ]
            },
            {
                "path": "/usr/bin/setup-network"
            }
        ],
        "poststart": [
            {
                "path": "/usr/bin/notify-start",
                "timeout": 5
            }
        ],
        "poststop": [
            {
                "path": "/usr/sbin/cleanup.sh",
                "args": [
                    "cleanup.sh",
                    "-f"
                ]
            }
        ]
    },
    "linux": {
        "devices": [
            {
                "path": "/dev/fuse",
                "type": "c",
                "major": 10,
                "minor": 229,
                "fileMode": 438,
                "uid": 0,
                "gid": 0
            },
            {
                "path": "/dev/sda",
                "type": "b",
                "major": 8,
                "minor": 0,
                "fileMode": 432,
                "uid": 0,
                "gid": 0
            }
        ],
        "uidMappings": [
            {
                "hostID": 1000,
                "containerID": 0,
                "size": 32000
            }
        ],
        "gidMappings": [
            {
                "hostID": 1000,
                "containerID": 0,
                "size": 32000
            }
        ],
        "sysctl": {
            "net.ipv4.ip_forward": "1",
            "net.core.somaxconn": "256"
        },
        "cgroupsPath": "/myRuntime/myContainer",
        "resources": {
            "network": {
                "classID": 1048577,
                "priorities": [
                    {
                        "name": "eth0",
                        "priority": 500
                    },
                    {
                        "name": "eth1",
                        "priority": 1000
                    }
                ]
            },
            "pids": {
                "limit": 32771
            },
            "hugepageLimits": [
                {
                    "pageSize": "2MB",
                    "limit": 9223372036854772000
                }
            ],
            "oomScoreAdj": 100,
            "memory": {
                "limit": 536870912,
                "reservation": 536870912,
                "swap": 536870912,
                "kernel": 0,
                "kernelTCP": 0,
                "swappiness": 0
            },
            "cpu": {
                "shares": 1024,
                "quota": 1000000,
                "period": 500000,
                "realtimeRuntime": 950000,
                "realtimePeriod": 1000000,
                "cpus": "2-3",
                "mems": "0-7"
            },
            "disableOOMKiller": false,
            "devices": [
                {
                    "allow": false,
                    "access": "rwm"
                },
                {
                    "allow": true,
                    "type": "c",
                    "major": 10,
                    "minor": 229,
                    "access": "rw"
                },
                {
                    "allow": true,
                    "type": "b",
                    "major": 8,
                    "minor": 0,
                    "access": "r"
                }
            ],
            "blockIO": {
                "blkioWeight": 10,
                "blkioLeafWeight": 10,
                "blkioWeightDevice": [
                    {
                        "major": 8,
                        "minor": 0,
                        "weight": 500,
                        "leafWeight": 300
                    },
                    {
                        "major": 8,
                        "minor": 16,
                        "weight": 500
                    }
                ],
                "blkioThrottleReadBpsDevice": [
                    {
                        "major": 8,
                        "minor": 0,
                        "rate": 600
                    }
                ],
                "blkioThrottleWriteIOPSDevice": [
                    {
                        "major": 8,
                        "minor": 16,
                        "rate": 300
                    }
                ]
            }
        },
        "rootfsPropagation": "slave",
        "seccomp": {
            "defaultAction": "SCMP_ACT_ALLOW",
            "architectures": [
                "SCMP_ARCH_X86"
            ],
            "syscalls": [
                {
                    "name": "getcwd",
                    "action": "SCMP_ACT_ERRNO"
                }
            ]
        },
        "namespaces": [
            {
                "type": "pid"
            },
            {
                "type": "network"
            },
            {
                "type": "ipc"
            },
            {
                "type": "uts"
            },
            {
                "type": "mount"
            },
            {
                "type": "user"
            },
            {
                "type": "cgroup"
            }
        ],
        "maskedPaths": [
            "/proc/kcore",
            "/proc/latency_stats",
            "/proc/timer_stats",
            "/proc/sched_debug"
        ],
        "readonlyPaths": [
            "/proc/asound",
            "/proc/bus",
            "/proc/fs",
            "/proc/irq",
            "/proc/sys",
            "/proc/sysrq-trigger"
        ],
        "mountLabel": "system_u:object_r:svirt_sandbox_file_t:s0:c715,c811"
    },
    "annotations": {
        "com.example.key1": "value1",
        "com.example.key2": "value2"
    }
}
```

[container-namespace]: glossary.md#container-namespace
[go-environment]: https://golang.org/doc/install/source#environment
[runtime-namespace]: glossary.md#runtime-namespace
[uts-namespace]: http://man7.org/linux/man-pages/man7/namespaces.7.html
[mount.8-filesystem-independent]: http://man7.org/linux/man-pages/man8/mount.8.html#FILESYSTEM-INDEPENDENT_MOUNT%20OPTIONS
[mount.8-filesystem-specific]: http://man7.org/linux/man-pages/man8/mount.8.html#FILESYSTEM-SPECIFIC_MOUNT%20OPTIONS
[mount.8]: http://man7.org/linux/man-pages/man8/mount.8.html
[stdin.3]: http://man7.org/linux/man-pages/man3/stdin.3.html
