## VFS 核心定位与设计思想

虚拟文件系统（Virtual Filesystem, VFS）是 Linux 内核在用户空间系统调用与具体存储介质之间构建的抽象层。其核心价值是践行 Unix **“一切皆文件”** 的哲学，为应用程序屏蔽底层文件系统（Ext4、XFS、NFS、procfs、sysfs 等）与硬件设备的物理差异，提供统一的 POSIX API（`open`、`read`、`write`、`ioctl` 等）。



### 多态实现的本质：C 语言实现的“面向对象”

VFS 并不是通过统一的逻辑“兼容”所有文件系统，而是通过类似 C++ 虚函数表（`vtable`）的机制实现运行时多态：



- **抽象基类**：VFS 定义了一组通用的数据结构规范（`super_block`、`inode`、`dentry`、`file`）。
- **虚函数表**：每个结构体内嵌专属的函数指针表（`super_operations`、`inode_operations`、`file_operations` 等）。
- **具体子类派生**：具体文件系统或底层驱动（如 Ext4 或字符设备驱动）在初始化时，将自己的具体实现函数地址填充到对应的函数指针表中。当用户调用 `read(fd, ...)` 时，内核沿着 `file->f_op->read` 动态分发至底层代码执行。

## VFS 四大核心对象全景

```
+-------------------------------------------------------------------------+
|                              super_block                                |
| (表示挂载的文件系统实例，记录块大小、挂载参数，持有 s_op: super_operations) |
+------------------------------------+------------------------------------+
                                     | 1
                                     |
                                     | N
+------------------------------------+------------------------------------+
|                                 dentry                                  |
| (目录项缓存，维护路径层级拓扑、文件名 d_name，持有 d_op: dentry_operations)   |
+------------------------------------+------------------------------------+
                                     | N
                                     |
                                     | 1
+------------------------------------+------------------------------------+
|                                 inode                                   |
| (物理文件元数据：大小、权限、物理块、i_mapping，持有 i_op: inode_operations) |
+------------------------------------+------------------------------------+
                                     ^
                                     | 指向
+------------------------------------+------------------------------------+
|                              struct file                                |
| (进程打开文件的动态上下文：读写偏移 f_pos、状态标志，持有 f_op: file_operations) |
+-------------------------------------------------------------------------+
```

| **核心对象**            | **对应内核结构体**   | **存储介质**                           | **生命周期**                                            | **核心职责与关键字段**                                       | **关联操作函数表**                                           |
| ----------------------- | -------------------- | -------------------------------------- | ------------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **超级块 (Superblock)** | `struct super_block` | 磁盘有对应，挂载后加载至内存           | 随文件系统 `mount` 创建，`umount` 销毁                  | 记录文件系统整体元数据。关键字段：`s_blocksize`（块大小）、`s_type`（文件系统类型）、`s_root`（指向根目录 dentry）。 | `struct super_operations` (`alloc_inode`, `write_inode`, `sync_fs`) |
| **索引节点 (Inode)**    | `struct inode`       | 磁盘持久化，使用时缓存在内存           | 物理文件创建时分配，直至文件彻底删除且无引用            | 记录物理文件的属性与存储位置。**不含文件名**。关键字段：`i_ino`（节点编号）、`i_size`（大小）、`i_nlink`（硬链接计数）、`i_mapping`（关联 Page Cache）。 | `struct inode_operations` (`create`, `lookup`, `mkdir`, `link`, `unlink`) |
| **目录项 (Dentry)**     | `struct dentry`      | **仅存在于内存**（由路径解析动态构建） | LRU 机制管理，内存不足或显式回收时销毁                  | 建立“路径/文件名”到“Inode”的粘合映射，维护目录树层级。关键字段：`d_name`（文件名）、`d_parent`（父目录指针）、`d_inode`（关联的 inode 指针）。 | `struct dentry_operations` (`d_compare`, `d_delete`, `d_hash`) |
| **文件对象 (File)**     | `struct file`        | **仅存在于内存**                       | 进程调用 `open()` 时创建，调用 `close()` 引用归零时释放 | 记录**进程打开文件**后的运行时上下文。关键字段：`f_pos`（当前读写字节偏移游标）、`f_flags`（打开属性如只读/追加）、`f_count`（引用计数）、`f_path`（包含 dentry 指针）。 | `struct file_operations` (`read`, `write`, `open`, `poll`, `mmap`, `ioctl`) |

## 核心数据流与底层映射链路

### 1. 从进程控制块到磁盘元数据的检索路径

一个进程发起读写操作时，内核自上而下的寻址追踪链路如下：



```
[进程 task_struct]
       │
       └──> files_struct
               │
               └──> fd_array[] (文件描述符指针数组，下标即用户态的 fd)
                       │
                       └──> [struct file] (维护打开状态: f_pos, f_flags, f_count)
                               ├── f_op ──> [struct file_operations] (驱动/FS 读写接口)
                               │
                               └── f_path.dentry ──> [struct dentry] (节点名: "log.txt")
                                                          │
                                                          └── d_inode ──> [struct inode] (文件元数据)
                                                                               │
                                                                               ├── i_mapping ──> [address_space] (Page Cache)
                                                                               └── i_sb ──────> [struct super_block] (挂载信息)
```

### 2. 路径解析机制与 Dentry Cache (dcache)

当程序调用 `open("/home/user/log.txt", O_RDONLY)` 时，内核必须依次解析每一级目录：



- **dcache 查询**：内核使用一个基于哈希表的缓存结构（dentry_hashtable）。以“父目录 dentry”与“当前分量名（如 user）”共同计算哈希值，先在内存中查找。
- **缓存命中**：直接获取对应分量的 dentry，继续下一级解析，完全无需访问慢速磁盘。
- **缓存未命中**：调用父目录 Inode 的 `i_op->lookup()` 函数，从磁盘读取该目录对应的数据块（Directory Data Blocks，里面存储了文件名与 Inode 号的映射列表）。找到目标 Inode 后，在内存中分配并构建新的 dentry 节点加入 dcache。
- **负目录项（Negative Dentry）**：如果文件不存在，VFS 同样会构建一个 `d_inode == nullptr` 的 dentry 保留在 dcache 中。作用是快速拒绝频繁访问不存在路径的请求，防止穿透到底层磁盘。

### 3. 页缓存（Page Cache）与回写架构

文件读写一般不直接接触块设备，而是由 Inode 内部的 `struct address_space`（页缓存核心结构）托管：



- **读流程**：通过 `address_space` 查找物理页帧（Page Frame）。若命中使用内存拷贝；若缺失则触发缺页异常（Page Fault），从磁盘读入对应页并加入缓存。
- **写流程（Writeback）**：数据拷贝至对应的物理页帧后，该页被标记为 `Dirty`（脏页）并立即向用户态返回成功。由后台内核线程（如 `kworker/flusher`）在满足周期时限或脏页比例阈值时，异步批量刷盘。
- **强持久化控制**：应用程序如需规避掉电丢失风险，必须显式调用 `fsync()`（刷数据和元数据）或 `fdatasync()`（仅刷数据），强制阻塞刷盘。

## 核心面试高频场景与深度追问

### Q1: 多次 `open()`、`fork()` 与 `dup()` 的底层结构变化是什么？

这是 Linux 系统编程和驱动开发面试中考察频率最高的问题，核心考察点在于 **`struct file` 与 `f_pos` 的共享语义**：



- **同一进程或不同进程多次 `open()` 相同路径**：
  - **底层行为**：每次 `open()` 都会在内核分配一个**全新且独立**的 `struct file`，但它们最终的 `dentry` 和 `inode` 是同一份。
  - **影响**：每个文件句柄拥有**完全独立的 `f_pos`**。两个进程各写各的，会根据各自游标相互覆盖数据（除非使用了 `O_APPEND` 标志，由内核在每次写入前强制原子定位到 Inode 当前末尾）。
- **调用 `fork()` 创建子进程**：
  - **底层行为**：子进程复制父进程的 `files_struct`，但只是把其中的 `fd_array[]` 指针复制了一份，并增加对应 `struct file` 的引用计数 `f_count`。
  - **影响**：父子进程中的 `fd` **共享同一个 `struct file`**，意味着它们**共享同一个读写偏移量 `f_pos`**。子进程写入并推进了 100 字节，父进程随后读取时，会紧接着从第 101 字节开始。
- **调用 `dup()` / `dup2()`**：
  - **底层行为**：在当前进程的 `fd_array[]` 中分配一个新的描述符下标，但该下标直接指向原有 `struct file`，并将其 `f_count` 自增。
  - **影响**：新旧 `fd` 指向同一个 `struct file`，读写操作完全联动。

### Q2: 硬链接（Hard Link）与软链接（Symbolic Link）在 VFS 层面有何本质区别？

- **硬链接**：
  - **本质**：在某个目录的数据块中，新建一个**指向已有 `inode` 的 dentry 映射**。
  - **特征**：
    1. 多个文件名拥有相同的 Inode 号。
    2. Inode 内的硬链接计数 `i_nlink` 加 1。
    3. 不能跨挂载点（跨文件系统），因为 Inode 编号仅在单个具体文件系统内部唯一。
    4. 不能对目录创建硬链接（防止目录树产生循环引用造成死循环）。
- **软链接（符号链接）**：
  - **本质**：是一个**完全独立的物理文件**，拥有自己专属的 Inode 和独立的磁盘块（或将较短路径直接存入 Inode 内部）。
  - **特征**：
    1. 其文件数据块内存储的内容是**目标文件的路径字符串**。
    2. 可以跨文件系统链接，也可以链接目录。
    3. 目标文件被删除后，软链接依旧存在，但读取时会遇到 `ENOENT`（变成死链接）。

### Q3: 为什么一个正在被写入的文件执行 `rm` 删除后，磁盘空间没有释放？

该问题用于考察文件的物理销毁判定条件。在 Inode 中存在两个核心计数器：



- `i_nlink`：硬链接计数（代表磁盘目录项引用次数）。
- `i_count`：内存打开引用计数（代表当前有多少个 `struct file` 指向它）。

**完整销毁流程**：



1. 执行 `rm` 命令本质是调用 `unlink()` 系统调用，内核将该文件名对应的 dentry 解除关联，并将目标 Inode 的 `i_nlink` 减 1。
2. 只有当 `i_nlink == 0`（磁盘上没有任何路径指向它）**且** `i_count == 0`（系统中没有任何进程处于打开该文件的状态）时，内核才会触发具体文件系统的空间回收逻辑（释放数据块与 Inode 节点）。
3. 如果此时有进程正在读写该文件，其 `i_count` 大于 0。虽然目录树上已找不到该文件，但该 Inode 及占用的数据块依然存留在磁盘上，直至持有该 `fd` 的所有进程退出或主动 `close()`，空间才会彻底归还系统。
4. **排查手段**：在生产环境中遇到底层磁盘爆满但 `du` 统计不到大文件时，运行 `lsof | grep deleted` 即可定位到持有已被删除文件句柄的进程。

## 嵌入式驱动与 C++ 工程开发视角

在 Linux C++ 工程（如机器人中间件、自动驾驶日志采集、嵌入式外设控制）中，VFS 的落地应用场景：



### 1. 字符设备驱动中 `file_operations` 的注册与分发

在嵌入式开发中操作硬件（如 CAN 控制器、IMU 传感器、GPIO），底层驱动正是通过向 VFS 注册 `file_operations` 将硬件抽象为文件的：



C

```
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/uaccess.h>

static int sensor_open(struct inode *inode, struct file *file) {
    // 获取设备私有数据，支持多实例复用
    return 0;
}

static ssize_t sensor_read(struct file *file, char __user *buf, size_t count, loff_t *ppos) {
    // 硬件采样 -> copy_to_user 拷贝到用户空间缓冲区
    return count;
}

static const struct file_operations sensor_fops = {
    .owner   = THIS_MODULE,
    .open    = sensor_open,
    .read    = sensor_read,
    // 如需提供参数配置或非阻塞模式，可实现 .unlocked_ioctl 和 .poll
};
```

应用程序调用标准的 `open("/dev/sensor0", O_RDONLY)` 时，内核通过设备的主次设备号（Major/Minor Device Number）找到对应的 `cdev`，并将 `file->f_op` 替换为驱动定义的 `sensor_fops`，完成 VFS 层的对接。



### 2. 伪文件系统的工程妙用

- **`/proc`（Process Information Pseudo-FS）**：
  - 在 C++ 机器人或自动驾驶程序中，可通过读取 `/proc/self/fd/` 自检是否存在句柄泄漏；读取 `/proc/self/status` 获取进程内存（VmRSS）、线程切换频率；读取 `/proc/sys/vm/drop_caches` 协助做极端性能基准测试。
- **`/sys`（Sysfs 设备拓扑）**：
  - 嵌入式系统常直接利用 Linux 现成的 sysfs 接口操作底层硬件。例如通过读写 `/sys/class/gpio/export` 控制引脚方向与电平，或通过 `/sys/devices/system/cpu/` 查看 CPU 实时调度频率。

### 3. C++ 高性能 I/O 选型考量

- **避免频繁的用户态/内核态拷贝**：常规 `read()`/`write()` 需要在 Page Cache 与用户态内存之间进行一次 CPU 拷贝。对于自动驾驶相机、激光雷达的海量点云录制任务，优先采用 `mmap()`（将 Page Cache 物理页映射到进程地址空间）或零拷贝技术（`splice()` / `sendfile()`）。
- **多路复用底层的 VFS 依赖**：`epoll` 能够监听多个文件描述符，前提是目标底层驱动或伪文件系统必须在其 `file_operations` 中正确实现了 `.poll` 钩子函数。如果驱动未实现 `.poll`，该文件描述符加入 epoll 就会报错 `EPERM`。