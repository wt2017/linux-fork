# Linux 内核启动流程与 rootfs 加载机制分析

本文档详细分析 Linux 内核启动流程，特别是加载 rootfs（根文件系统）前后的关键细节。基于对 Linux 内核代码（版本 b954e94a71d80adf9d52d66a88b11e0e20cf26ec）的分析。

## 1. 内核启动整体流程概述

### 1.1 启动入口点
Linux 内核的启动入口点是 `start_kernel()` 函数（定义于 `init/main.c` 第1017行）。这是内核从引导加载程序接管控制后的第一个 C 语言函数。

### 1.2 启动阶段划分
内核启动可以分为以下几个主要阶段：
1. **早期初始化阶段**：处理器、内存、中断等基础设置
2. **子系统初始化阶段**：各种内核子系统的初始化
3. **设备驱动初始化阶段**：设备驱动的探测和初始化
4. **文件系统挂载阶段**：根文件系统的识别和挂载
5. **用户空间初始化阶段**：启动第一个用户进程（init）

## 2. 启动参数处理

### 2.1 命令行参数解析
内核启动参数处理主要涉及以下函数：
- `parse_early_param()`：解析早期参数
- `parse_args()`：解析常规参数
- `unknown_bootoption()`：处理未知启动选项

### 2.2 关键启动参数
与 rootfs 加载相关的关键参数：

| 参数 | 作用 | 处理函数 |
|------|------|----------|
| `root=` | 指定根设备 | `root_dev_setup()` |
| `ro`/`rw` | 只读/读写挂载 | `readonly()`/`readwrite()` |
| `rootwait` | 等待根设备就绪 | `rootwait_setup()` |
| `init=` | 指定 init 程序 | `init_setup()` |
| `rdinit=` | 指定 initramfs 中的 init | `rdinit_setup()` |
| `rootfstype=` | 指定根文件系统类型 | 在 `mount_root()` 中处理 |

## 3. 文件系统初始化

### 3.1 VFS 缓存初始化
- `vfs_caches_init_early()`：早期 VFS 缓存初始化
- `vfs_caches_init()`：完整的 VFS 缓存初始化

### 3.2 文件系统类型注册
内核在启动过程中会注册各种文件系统类型，包括：
- 内置文件系统（ext4, btrfs, xfs 等）
- 伪文件系统（proc, sysfs, tmpfs 等）
- 网络文件系统（NFS, CIFS 等）

## 4. rootfs 加载的关键路径

### 4.1 整体调用链
```
start_kernel()
    └── rest_init()
        └── kernel_init()
            └── kernel_init_freeable()
                ├── do_basic_setup()
                ├── wait_for_initramfs()
                ├── console_on_rootfs()
                └── prepare_namespace()  # 关键：准备命名空间并挂载根文件系统
```

### 4.2 prepare_namespace() 函数
`prepare_namespace()` 是 rootfs 加载的核心函数，主要职责：
1. 识别根设备（通过 `ROOT_DEV` 全局变量）
2. 处理 initramfs/initrd
3. 挂载根文件系统
4. 切换到新的根文件系统

### 4.3 根设备识别流程
```c
// do_mounts.c 中的关键流程
mount_root()
    ├── 解析 root= 参数指定的设备
    ├── 尝试各种文件系统类型
    ├── 调用 mount_root_generic() 实际挂载
    └── 设置当前根目录
```

### 4.4 支持的文件系统类型
内核支持多种根文件系统类型：
1. **本地块设备**：ext4, btrfs, xfs, etc.
2. **网络文件系统**：NFS（需要 CONFIG_ROOT_NFS）
3. **SMB/CIFS**：需要 CONFIG_CIFS_ROOT
4. **伪文件系统**：ramfs, tmpfs（用于 initramfs）

## 5. initramfs/initrd 处理机制

### 5.1 initramfs 与 initrd 的区别
- **initramfs**：基于 cpio 格式的初始 RAM 文件系统，内置在内核镜像中
- **initrd**：独立的初始 RAM 磁盘镜像，需要额外加载

### 5.2 initramfs 加载流程
```c
// initramfs.c 中的关键函数
populate_rootfs()
    ├── 解压内置 initramfs
    ├── 处理外部 initramfs（如果有）
    └── 创建初始根文件系统结构
```

### 5.3 initrd 加载流程
```c
// do_mounts_initrd.c 中的关键函数
initrd_load()
    ├── 从指定位置加载 initrd 镜像
    ├── 解压到内存中
    └── 切换到 initrd 作为临时根文件系统
```

### 5.4 从 initramfs 切换到真实根文件系统
切换过程发生在 `prepare_namespace()` 函数中：
1. 检查是否有 initramfs/initrd
2. 如果有，先在其中执行初始化
3. 通过 `pivot_root()` 或 `chroot()` 切换到真实根文件系统
4. 清理 initramfs/initrd 占用的内存

## 6. 关键数据结构和全局变量

### 6.1 重要全局变量
```c
// do_mounts.c
dev_t ROOT_DEV;                    // 根设备号
int root_mountflags = MS_RDONLY | MS_SILENT; // 根文件系统挂载标志
static char saved_root_name[64];   // 保存的根设备名称
static int root_wait;              // 根设备等待超时

// main.c
char *execute_command;             // init= 参数指定的命令
char *ramdisk_execute_command;     // rdinit= 参数指定的命令
bool ramdisk_execute_command_set;  // rdinit 是否已设置
```

### 6.2 关键数据结构
```c
// 根设备挂载信息
struct root_device {
    dev_t dev;
    char *name;
    int flags;
};

// 文件系统类型信息
struct file_system_type {
    const char *name;
    int fs_flags;
    struct dentry *(*mount)(struct file_system_type *, int,
                           const char *, void *);
    // ... 其他字段
};
```

## 7. 用户空间初始化

### 7.1 第一个用户进程的启动
在 `kernel_init()` 函数中，内核尝试按以下顺序启动 init 进程：
1. `ramdisk_execute_command`（rdinit= 参数）
2. `execute_command`（init= 参数）
3. `CONFIG_DEFAULT_INIT`（编译时配置的默认 init）
4. `/sbin/init`
5. `/etc/init`
6. `/bin/init`
7. `/bin/sh`

### 7.2 控制台初始化
`console_on_rootfs()` 函数负责打开 `/dev/console` 作为标准输入、输出和错误输出。

### 7.3 系统状态转换
```c
enum system_states {
    SYSTEM_BOOTING,      // 系统启动中
    SYSTEM_SCHEDULING,   // 调度器已初始化
    SYSTEM_FREEING_INITMEM, // 正在释放初始化内存
    SYSTEM_RUNNING       // 系统正常运行
};
```

## 8. PID0 创建 PID1 的详细过程

Linux 内核启动过程中，PID0（内核空闲线程）创建 PID1（init 进程）的过程是系统从内核空间过渡到用户空间的关键步骤。

### 8.1 整体调用链
```
start_kernel()
    └── rest_init()                    # PID0 执行
        ├── user_mode_thread()         # 创建 init 进程
        │   └── kernel_clone()         # 实际创建进程
        │       └── copy_process()     # 复制进程描述符
        └── kernel_thread()            # 创建 kthreadd 进程
```

### 8.2 rest_init() 函数分析
`rest_init()` 函数（定义于 `init/main.c` 第716行）是内核启动的第二阶段入口：

```c
static noinline void __ref __noreturn rest_init(void)
{
    struct task_struct *tsk;
    int pid;

    rcu_scheduler_starting();
    
    // 1. 创建 init 进程（PID 1）
    pid = user_mode_thread(kernel_init, NULL, CLONE_FS);
    
    // 2. 将 init 进程绑定到启动 CPU
    rcu_read_lock();
    tsk = find_task_by_pid_ns(pid, &init_pid_ns);
    tsk->flags |= PF_NO_SETAFFINITY;
    set_cpus_allowed_ptr(tsk, cpumask_of(smp_processor_id()));
    rcu_read_unlock();

    numa_default_policy();
    
    // 3. 创建 kthreadd 进程（PID 2）
    pid = kernel_thread(kthreadd, NULL, NULL, CLONE_FS | CLONE_FILES);
    rcu_read_lock();
    kthreadd_task = find_task_by_pid_ns(pid, &init_pid_ns);
    rcu_read_unlock();

    system_state = SYSTEM_SCHEDULING;
    complete(&kthreadd_done);

    // 4. PID0 进入空闲循环
    schedule_preempt_disabled();
    cpu_startup_entry(CPUHP_ONLINE);
}
```

### 8.3 user_mode_thread() 函数
`user_mode_thread()` 函数（定义于 `kernel/fork.c` 第2790行）是创建用户模式线程的包装函数：

```c
pid_t user_mode_thread(int (*fn)(void *), void *arg, unsigned long flags)
{
    struct kernel_clone_args args = {
        .flags        = ((flags | CLONE_VM | CLONE_UNTRACED) & ~CSIGNAL),
        .exit_signal  = (flags & CSIGNAL),
        .fn           = fn,
        .fn_arg       = arg,
    };
    
    return kernel_clone(&args);
}
```

**关键标志说明**：
- `CLONE_VM`：共享地址空间
- `CLONE_UNTRACED`：不被跟踪（避免 ptrace 干扰）
- `CLONE_FS`：共享文件系统信息

### 8.4 kernel_clone() 函数
`kernel_clone()` 函数（定义于 `kernel/fork.c` 第2672行）是进程创建的核心函数：

```c
pid_t kernel_clone(struct kernel_clone_args *args)
{
    u64 clone_flags = args->flags;
    struct completion vfork;
    struct pid *pid;
    struct task_struct *p;
    int trace = 0;
    pid_t nr;

    // 1. 复制进程描述符
    p = copy_process(NULL, trace, NUMA_NO_NODE, args);
    
    if (IS_ERR(p))
        return PTR_ERR(p);

    // 2. 获取 PID
    pid = get_task_pid(p, PIDTYPE_PID);
    nr = pid_vnr(pid);

    // 3. 唤醒新进程
    wake_up_new_task(p);

    put_pid(pid);
    return nr;
}
```

### 8.5 copy_process() 的关键操作
`copy_process()` 函数执行以下关键操作：
1. **分配 task_struct**：为新进程分配进程描述符
2. **复制父进程资源**：复制或共享地址空间、文件描述符、信号处理等
3. **设置进程上下文**：初始化栈、寄存器状态等
4. **分配 PID**：从 PID 命名空间中分配新的进程 ID

### 8.6 进程创建的关键区别

| 进程类型 | 创建函数 | 关键标志 | 执行上下文 |
|----------|----------|----------|------------|
| **init 进程** | `user_mode_thread()` | `CLONE_VM \| CLONE_FS \| CLONE_UNTRACED` | 用户空间 |
| **kthreadd** | `kernel_thread()` | `CLONE_FS \| CLONE_FILES` | 内核空间 |
| **普通进程** | `sys_clone()` | 用户指定 | 用户空间 |

### 8.7 进程状态转换
```
PID0 (内核空闲线程)
    ├── 创建 PID1 (init 进程)
    │   ├── 复制进程上下文
    │   ├── 设置用户空间执行环境
    │   └── 通过 execve() 加载 init 程序
    └── 创建 PID2 (kthreadd 进程)
        └── 管理内核线程
```

### 8.8 关键数据结构

```c
// 进程创建参数
struct kernel_clone_args {
    u64 flags;          // 克隆标志
    int __user *pidfd;  // PID 文件描述符
    int __user *child_tid; // 子进程 TID
    int __user *parent_tid; // 父进程 TID
    int exit_signal;    // 退出信号
    unsigned long stack; // 用户栈地址
    unsigned long stack_size; // 栈大小
    pid_t *set_tid;     // PID 设置
    size_t set_tid_size; // PID 设置大小
    int cgroup;         // Cgroup ID
    struct cgroup *cgrp; // Cgroup 指针
    int (*fn)(void *);  // 执行函数
    void *fn_arg;       // 函数参数
    struct file *file;  // 文件描述符
};
```

### 8.9 进程创建的系统调用关系

Linux 内核中进程创建的系统调用形成了一个层次化的调用关系，最终都汇聚到 `kernel_clone()` 函数：

```
用户空间系统调用：
    fork()   clone()   vfork()  pthread_create()
       |        |         |            |
       └────────┼─────────┼────────────┘
                ▼         ▼
         kernel_clone()   <---   kernel_thread()
                ▲                      |
                | (内核 API)            |
                └──────────────────────┘
                      |
                      ▼
             创建新的 task_struct
                 (新进程/线程)

独立路径：
    execve()  → 替换当前进程的映像 (不创建新进程)
```

#### 8.9.1 系统调用映射关系

| 用户空间系统调用 | 内核入口函数 | 关键标志 | 描述 |
|------------------|--------------|----------|------|
| `fork()` | `sys_fork()` | `SIGCHLD` | 创建子进程，完全复制父进程 |
| `vfork()` | `sys_vfork()` | `CLONE_VFORK \| CLONE_VM \| SIGCHLD` | 创建子进程，共享地址空间，父进程阻塞直到子进程退出 |
| `clone()` | `sys_clone()` | 用户指定 | 灵活的进程/线程创建 |
| `pthread_create()` | `sys_clone()` | `CLONE_VM \| CLONE_FS \| CLONE_FILES \| CLONE_SIGHAND \| CLONE_THREAD \| CLONE_SYSVSEM \| CLONE_SETTLS \| CLONE_PARENT_SETTID \| CLONE_CHILD_CLEARTID` | POSIX 线程创建 |

#### 8.9.2 内核内部创建函数

| 内核函数 | 调用者 | 用途 | 关键区别 |
|----------|--------|------|----------|
| `kernel_thread()` | 内核代码 | 创建内核线程 | 在内核空间执行，不返回用户空间 |
| `user_mode_thread()` | `rest_init()` | 创建用户空间线程 | 创建后通过 `execve()` 加载用户程序 |
| `kthread_create()` | 内核模块 | 创建可延迟启动的内核线程 | 使用工作队列延迟启动 |

#### 8.9.3 关键标志说明

| 标志 | 含义 | 影响 |
|------|------|------|
| `CLONE_VM` | 共享地址空间 | 创建线程而非进程 |
| `CLONE_FS` | 共享文件系统信息 | 共享当前目录、根目录等 |
| `CLONE_FILES` | 共享文件描述符表 | 共享打开的文件 |
| `CLONE_SIGHAND` | 共享信号处理程序 | 共享信号处理函数 |
| `CLONE_THREAD` | 在同一线程组中 | 共享 TGID（线程组 ID） |
| `CLONE_VFORK` | vfork() 语义 | 父进程阻塞直到子进程 execve() 或退出 |
| `CLONE_UNTRACED` | 不被跟踪 | 避免 ptrace 干扰 |

#### 8.9.4 execve() 的特殊性

`execve()` 系统调用与其他进程创建系统调用有本质区别：
- **不创建新进程**：替换当前进程的地址空间
- **进程 ID 不变**：保持相同的 PID
- **资源处理**：
  - 关闭标记了 `FD_CLOEXEC` 的文件描述符
  - 保留其他资源（信号掩码、进程组等）
  - 重置信号处理程序为默认值

#### 8.9.5 进程创建的性能考虑

1. **写时复制（Copy-on-Write）**：
   - 页表、内存页等资源初始时共享
   - 只有在写入时才进行实际复制
   - 大幅减少 fork() 的开销

2. **线程 vs 进程**：
   - 线程创建（`CLONE_VM`）比进程创建快
   - 线程共享地址空间，减少内存开销
   - 进程有独立地址空间，提供更好的隔离性

3. **vfork() 的优化**：
   - 完全共享地址空间
   - 父进程阻塞，避免竞争条件
   - 适用于立即执行 execve() 的场景

## 9. 调试和故障排除

### 9.1 常见问题

#### 9.1.1 根文件系统挂载失败
可能原因：
1. 根设备不存在或未就绪
2. 文件系统类型不匹配
3. 内核缺少对应的文件系统驱动
4. 设备节点未创建

调试方法：
- 检查内核日志中的错误信息
- 确认 `root=` 参数格式正确
- 使用 `rootdelay=` 参数增加等待时间

#### 9.1.2 initramfs 相关问题
可能原因：
1. initramfs 镜像损坏
2. initramfs 中缺少必要的驱动或工具
3. 切换根文件系统失败

调试方法：
- 在 initramfs 中添加调试 shell
- 检查 initramfs 的构建过程
- 查看内核解压 initramfs 的日志

### 9.2 调试技巧

#### 9.2.1 启用详细日志
在内核命令行中添加：
```
loglevel=7 debug earlyprintk
```

#### 9.2.2 检查启动参数
```
cat /proc/cmdline
```

#### 9.2.3 查看根设备信息
```
cat /proc/mounts | grep " / "
```

## 10. 代码文件索引

### 10.1 关键文件
| 文件路径 | 主要功能 |
|----------|----------|
| `init/main.c` | 内核启动主流程 |
| `init/do_mounts.c` | 根文件系统挂载 |
| `init/do_mounts.h` | 挂载相关函数声明 |
| `init/initramfs.c` | initramfs 处理 |
| `init/do_mounts_initrd.c` | initrd 处理 |
| `init/do_mounts_rd.c` | RAM 磁盘支持 |
| `kernel/fork.c` | 进程创建相关函数 |

### 10.2 关键函数
| 函数名 | 所在文件 | 功能描述 |
|--------|----------|----------|
| `start_kernel()` | main.c | 内核启动入口点 |
| `rest_init()` | main.c | 创建内核线程，进入第二阶段 |
| `kernel_init()` | main.c | 内核初始化主函数 |
| `prepare_namespace()` | do_mounts.c | 准备命名空间并挂载根文件系统 |
| `mount_root()` | do_mounts.c | 挂载根文件系统 |
| `initrd_load()` | do_mounts_initrd.c | 加载 initrd |
| `populate_rootfs()` | initramfs.c | 填充 initramfs |
| `user_mode_thread()` | fork.c | 创建用户模式线程 |
| `kernel_clone()` | fork.c | 进程创建核心函数 |
| `copy_process()` | fork.c | 复制进程描述符 |

## 11. 总结

Linux 内核启动流程是一个复杂但高度模块化的过程。rootfs 加载是启动过程中的关键环节，涉及多个子系统的协同工作：

1. **参数解析**：正确解析启动参数是成功挂载 rootfs 的前提
2. **设备识别**：准确识别根设备是挂载的基础
3. **文件系统支持**：内核需要包含对应的文件系统驱动
4. **initramfs/initrd**：提供了灵活的早期用户空间环境
5. **用户空间切换**：平滑过渡到完整的用户空间环境
6. **进程创建**：PID0 通过 `rest_init()` -> `user_mode_thread()` -> `kernel_clone()` 调用链创建 PID1（init 进程）

理解这些机制对于内核调试、系统定制和嵌入式开发都具有重要意义。

---
*文档基于 Linux 内核版本：b954e94a71d80adf9d52d66a88b11e0e20cf26ec*
*分析时间：2026年4月19日*