# 请简单介绍一下 Netfilter 的工作原理。

Linux数据包处理全流程：

1. 网卡收到以太网帧，硬件中断：通过DMA将数据拷贝到内存，调用netif_rx(skb)将数据包交到接收队列
2. 软中断（在系统调用返回后触发/用通用中断处理）：注册net_rx_action函数，在这个函数中将数据包传给相对应地数据包处理例程（ipv4/ipv6之类的例程不同）
3. 数据包处理例程：先ip检验，然后就是**第一个钩子NF_INET_PRE_ROUTING**
4. 穿过这个钩子就根据数据包地不同调用不同的函数，有的是发给本机，有的会去转发，有的做多播路由
5. 如果做转发，转发其中有一步是要检查ttl的，如果出错要返回一个ICMP报文给发送者，如果检查都没问题，**就会遇到NF_INET_FORWARD这个钩子**

```c++
网卡收到以太网帧，硬件中断 → 驱动处理 → netif_rx → 软中断触发 → net_rx_action → ip_rcv → 
netfilter钩子 → 路由决策 → 本地交付/转发 → 四层协议处理 → 输出路径 → 
链路层封装 → 硬件发送
```

Netfilter 是 Linux 内核中的一个数据包处理框架，它在内核协议栈的不同位置设置了一系列的钩子点（hook points）。当数据包在网络栈中传输时（包括进入、离开系统，以及在转发过程中），会依次经过这些钩子点。在每个钩子点上，注册的钩子函数（hook functions）会被调用，这些函数可以对数据包进行检查、修改、丢弃等操作。通过这种方式，Netfilter 允许用户实现各种网络功能，如防火墙、NAT 等。

| Hook                 | 调用的时机                                                   |
| -------------------- | ------------------------------------------------------------ |
| NF_INET_PRE_ROUTING  | 刚刚进入网络层的数据包通过此点（刚刚进行完版本号，校验和等检测），目的转换在此点进行； |
| NF_INET_LOCAL_IN     | 经路由查找后，送往本机的通过此检查点，INPUT包过滤在此点进行； |
| NF_INET_FORWARD      | 要转发的包经过此检测点，FORWARD包过滤在此点进行              |
| NF_INET_LOCAL_OUT    | 本机进程发出的包通过此检测点，OUTPUT包过滤在此点进行         |
| NF_INET_POST_ROUTING | 所有马上便要通过网络设备出去的包通过此检测点，内置的源地址转换功能（包括地址伪装）在此点进行 |

![image-20250309085328670](firewall问题.assets/image-20250309085328670.png)

# 为什么选择使用 Netfilter 来实现防火墙？

iptables就是使用netfilter来实现，相比于我自己写的netfilter，它自带了很多通用的设计，而且我的使用了哈希表，更快更有效率

- **内核级处理**：Netfilter 在内核空间工作，能够在数据包到达用户空间应用程序之前进行处理，避免了用户空间和内核空间之间频繁的数据拷贝，提高了处理效率。
- **高度可定制**：它提供了多个钩子点，允许开发者在数据包处理的不同阶段进行干预，根据具体需求实现各种复杂的过滤规则。
- **集成性好**：作为 Linux 内核的一部分，与 Linux 网络栈紧密集成，能够方便地利用内核提供的各种网络功能和数据结构。

# 这个防火墙程序的主要功能是什么？

这个防火墙程序的主要功能是根据预定义的规则对网络数据包进行过滤。它会检查数据包的源 IP、目的 IP、源端口、目的端口以及协议类型等信息，根据规则决定是否允许数据包通过。同时，它还维护了连接状态信息，用于处理基于连接的协议（如 TCP、UDP）的数据包，确保只有符合规则的连接可以继续通信。此外，程序还提供了日志记录功能，记录数据包的处理结果。

# 怎么判断连接超时

在连接匹配成功或者规则匹配成功时，会更新连接的时间戳，将其设置为当前时间加上预设的超时时间。

在每次处理数据包/遍历连接状态列表/遍历 NAT 连接状态列表时，会检查连接的时间戳是否超过当前时间，如果超过则认为连接超时，将其从连接状态列表中删除。

# Netlink 协议

Netlink 协议是 Linux 内核提供的一种用于用户空间进程与内核空间进行通信的机制。以下是关于 Netlink 协议的详细介绍：

## 主要用途

- **配置和管理内核模块**：用户空间程序可以通过 Netlink 向内核模块发送配置信息，如防火墙规则、网络设备参数等，内核模块接收这些信息后进行相应的设置和调整。
- **事件通知**：内核可以使用 Netlink 向用户空间发送异步事件通知，例如网络连接状态的变化、设备插拔事件等。用户空间程序可以通过监听 Netlink 套接字来接收这些通知并做出相应的处理。

## 主要特点

- **异步通信**：Netlink 支持异步通信模式，内核和用户空间可以在不阻塞的情况下进行消息的发送和接收，提高了系统的响应性能。
- **多播和广播**：Netlink 支持多播和广播功能，内核可以将消息发送给多个用户空间进程，或者向所有监听特定 Netlink 协议的进程广播消息。
- **可靠性**：Netlink 协议提供了可靠的消息传递机制，确保消息不会丢失或重复，并且可以处理消息的确认和重传。

## 工作原理

1. **创建套接字**：用户空间程序和内核模块都需要创建 Netlink 套接字来进行通信。用户空间使用 `socket()` 系统调用创建套接字，内核使用 `netlink_kernel_create()` 函数创建内核端的 Netlink 套接字。
2. **绑定套接字**：用户空间程序需要将创建的套接字绑定到一个 Netlink 协议地址上，以便接收来自内核的消息。内核模块也需要将其 Netlink 套接字绑定到相应的协议地址。
3. **消息发送和接收**：用户空间程序和内核模块可以通过 `sendmsg()` 和 `recvmsg()` 函数来发送和接收 Netlink 消息。消息以 Netlink 消息头和消息数据的形式进行封装。

# 事件驱动机制！！！

内核的网络子系统采用了事件驱动的架构。当用户空间通过 Netlink 套接字发送消息时，会触发内核网络栈中的一系列操作。

- **用户空间发送消息**：用户空间程序使用 `sendmsg` 等系统调用将消息发送到内核的 Netlink 套接字。
- **内核网络栈处理**：内核的网络栈接收到消息后，会根据消息的类型和目标地址，将消息路由到对应的 Netlink 套接字。
- **回调函数调用**：一旦消息到达 Netlink 套接字，内核会自动调用之前注册的回调函数（即 `nltest_krecv`）来处理这个消息。

# 选择 Netlink 进行内核和用户空间通信的原因及其他可选方案

选择 Netlink 的原因：Netlink 是一种专门用于内核和用户空间通信的机制，它提供了**全双工、异步**的通信方式，并且支持多播，非常适合用于内核模块和用户空间程序之间的信息交换，例如防火墙模块向用户空间发送日志信息。

其他可选方案：

- 系统调用：通过系统调用可以让用户空间程序和内核进行交互，但**系统调用通常是同步的，对于大量数据传输或频繁的信息交换效率较低**，而且系统调用的设计初衷不是专门用于这种复杂的通信场景。

- proc 文件系统：可以通过读写 proc 文件系统中的文件来实现内核和用户空间的通信，但这种方式**proc不太适合实时**的、大量的信息交互，而且在数据格式和同步方面需要更多的处理。

  `/proc` 是 Linux 内核提供的一个虚拟文件系统（伪文件系统），用于向用户空间提供内核运行状态、进程信息、硬件配置等数据。它通过文件和目录的形式组织，所有内容均动态生成，不占用实际磁盘空间。用户可以通过读取 `/proc` 下的文件获取系统实时信息，也可以通过写入特定文件修改内核参数。

# linux内核与用户空间通信

在 Linux 系统中，除了 Netlink 之外，还有多种内核与用户空间通信的机制。以下是几种常见的方式：

### 1. **系统调用（System Call）**

- **原理**：用户空间通过 `syscall` 指令触发内核函数，直接进入内核态执行特定操作。
- **优点：**
  - 高效且直接，内核与用户空间的交互延迟低。
  - 支持复杂的控制逻辑和数据传递。
- **缺点：**
  - 需要修改内核代码，增加系统调用号，灵活性较低。
  - 系统调用的参数和功能一旦确定，后续扩展困难。
- **应用场景：**
  - 实现用户空间对内核资源的直接访问（如文件操作、进程管理）。
  - 自定义内核功能（如添加新的 `open` 或 `ioctl` 操作）。

### 2. **proc 文件系统**

- **原理**：通过读写 `/proc` 目录下的虚拟文件实现通信。内核将数据导出为文本文件，用户空间通过 `read/write` 操作与内核交互。
- **优点：**
  - 简单易用，无需复杂的代码。
  - 支持动态配置和状态查询。
- **缺点：**
  - 性能较低，适合小数据量交互。
  - 文本格式解析可能导致错误。
- **应用场景：**
  - 内核参数配置（如 `/proc/sys` 下的网络参数）。
  - 系统状态监控（如 `/proc/cpuinfo`、`/proc/meminfo`）。

### 3. **sysfs 文件系统**

- **原理**：基于 `sysfs` 的虚拟文件系统，用于导出内核设备、驱动和配置信息。用户空间通过读写 `/sys` 目录下的文件与内核交互。
- **优点：**
  - 与设备驱动紧密集成，适合硬件相关的配置。
  - 支持动态设备管理（如热插拔）。
- **缺点：**
  - 主要面向设备管理，通用性不如其他机制。
- **应用场景：**
  - 设备驱动参数配置（如 GPIO 控制、传感器校准）。
  - udev 规则触发（通过 `sysfs` 事件）。

### 4. **字符设备驱动（Character Device）**

- **原理**：用户空间通过 `/dev` 下的字符设备节点（如 `/dev/my_device`）与内核通信。内核驱动通过 `read/write/ioctl` 处理用户请求。
- **优点：**
  - 灵活支持双向通信，适合复杂协议。
  - 支持异步通知（如 `poll/select`）。
- **缺点：**
  - 需要编写内核驱动代码，开发成本较高。
- **应用场景：**
  - 硬件驱动控制（如串口、GPIO）。
  - 用户空间与内核的高频数据交互（如视频流处理）。

### 5. **共享内存（Shared Memory）**

- **原理**：内核与用户空间共享一段物理内存，通过内存映射（`mmap`）实现数据交换。
- **优点：**
  - 高效，避免多次数据拷贝。
  - 适合大数据量或高频通信场景。
- **缺点**：需要额外的同步机制（如信号量、自旋锁）。
- **应用场景：**
  - 高性能数据传输（如视频编解码、网络包处理）。
  - 实时通信（如内核与用户空间的实时监控系统）。

### 6. **信号（Signal）**

- **原理**：内核通过发送信号（如 `SIGIO`）通知用户空间事件，用户空间通过信号处理函数响应。
- **优点：**轻量级异步通知。
- **缺点：**仅支持简单事件通知，无法传递复杂数据。
- **应用场景：**设备中断通知（如键盘输入、网络事件）。

### 7. **内存映射文件（Memory-Mapped Files）**

- **原理**：用户空间通过 `mmap` 将内核空间的内存区域映射到用户地址空间，直接访问内核数据。
- **优点：**高效，支持快速数据访问。
- **缺点：**需要内核显式导出内存区域，安全性风险较高。
- **应用场景：**内核调试（如访问内核变量）。

### 8. **命名管道（FIFO）**

- **原理**：用户空间创建命名管道文件，内核通过读写该文件与用户空间通信。
- **优点：**简单易用，支持阻塞 / 非阻塞模式。
- **缺点：**仅支持用户空间进程间通信，内核需作为普通进程参与。
- **应用场景：**简单的配置或状态传递（如守护进程与前端工具的交互）。

# 选择源目 IP 后两组异或哈希方式的原因、优缺点

选择这种哈希方式的原因是可以利用源目 IP 的部分信息快速计算哈希值，而且异或操作相对简单，计算速度快。

优点：计算速度快，能够快速将 IP 信息映射到哈希表的索引位置，提高查找和插入的效率。

缺点：可能存在哈希冲突，因为不同的 IP 组合可能会得到相同的哈希值。

# 哈希冲突的处理及其他哈希函数或哈希表实现方式

使用开放寻址法来处理哈希冲突，即当发生冲突时，寻找下一个可用的槽位。

# 如何优化连接状态哈希表的查找效率？

- 使用链表法处理哈希冲突时，可以考虑将链表转换为红黑树等更高效的数据结构，以提高查找效率。
- **哈希表扩容**：当哈希表的负载因子过高时，适当扩容哈希表，减少冲突的发生。

# 优化哈希表

像我在这里代码使用的哈希表，其下标为源ip和目的ip前16位异或，因此使用了一个66000大小的哈希表（这样大于2^16），有没有办法不使用这么多空间但是查找插入效率接近这个哈希表

| 方法             | 空间节省 | 查找效率 | 实现复杂度 | 适用场景                 |
| ---------------- | -------- | -------- | ---------- | ------------------------ |
| **优化哈希函数** | 中       | 高       | 低         | 冲突率较低的场景         |
| **开放寻址法**   | 高       | 中       | 中         | 内存受限且冲突可控       |
| **分块哈希表**   | 高       | 中       | 高         | 内存严格受限             |
| **动态哈希表**   | 极高     | 高       | 极高       | 内核内存充足且负载变化大 |

分块哈希表：

与传统哈希表类似，分块哈希表有一个数组，数组的每个元素对应一个块。数组的大小通常是一个质数，以减少哈希冲突的可能性。

插入时：

1. 计算元素的哈希值，根据哈希值确定对应的块。
2. 检查该块内是否有空闲槽位。如果有，将元素插入到空闲槽位中；如果没有，则表示该块已满，可能需要进行处理，如扩展块或标记插入失败。

# 这个防火墙程序的性能瓶颈可能在哪里？

- **规则匹配效率**：当规则数量很大时，遍历规则列表进行匹配的时间复杂度较高，可能成为性能瓶颈。
- **哈希表冲突**：连接状态哈希表可能会出现冲突，导致查找效率下降。
- **内存分配和释放**：频繁的内存分配和释放操作（如在创建和删除连接状态时）可能会影响性能。

# 如果规则数量很大，如何提高规则匹配的效率？

- **规则排序**：对规则列表进行排序，例如按照源 IP、目的 IP、协议类型等进行排序，这样在匹配时可以提前终止不必要的遍历。
- **使用更高效的数据结构**：例如，将规则列表转换为二叉搜索树或哈希表，以提高查找效率。
- **规则合并**：合并一些相似的规则，减少规则的数量。

# 这个防火墙程序存在哪些安全风险？

- **缓冲区溢出**：在处理数据包时，如果没有对数据包的长度进行正确的检查，可能会导致缓冲区溢出漏洞。
- **哈希表冲突攻击**：攻击者可能通过**构造特定的数据包，引发哈希表冲突**，导致性能下降或拒绝服务。

# 如何防止规则被恶意篡改？

- **访问控制**：限制对规则文件或规则存储区域的访问权限，只有授权的用户或进程才能修改规则。
- **完整性校验**：对规则文件或规则数据进行数字签名或哈希校验，在加载规则时进行验证，确保规则没有被篡改。

# 如何处理异常数据包，如超大数据包或畸形数据包？

- **数据包长度检查**：在处理数据包之前，先检查数据包的长度是否超过了合理的范围，如果超过则直接丢弃。
- **协议校验**：对数据包的协议头部进行校验，确保其符合协议规范。如果发现畸形数据包，直接丢弃或进行记录。

# 能否将这个防火墙程序与其他安全机制集成，如入侵检测系统？

可以。可以通过以下方式实现集成：

- **数据共享**：将防火墙程序捕获的数据包信息（如源 IP、目的 IP、协议类型等）传递给入侵检测系统，供其进行进一步的分析和检测。
- **协同决策**：根据入侵检测系统的检测结果，动态调整防火墙的过滤规则。例如，当入侵检测系统发现某个 IP 地址存在异常行为时，防火墙可以临时阻止该 IP 地址的访问。

# 除了连接状态哈希表，以下几种数据结构也可以用于提高防火墙规则匹配的效率：

### 1. 二叉搜索树（Binary Search Tree，BST）

- 原理：
  - 二叉搜索树是一种树形数据结构，对于树中的每个节点，其左子树中的所有节点的值都小于该节点的值，右子树中的所有节点的值都大于该节点的值。在规则匹配中，可以将规则按照某个关键属性（如源 IP 地址、目的 IP 地址等）进行排序并构建二叉搜索树。
- 优势：
  - 平均情况下，插入、删除和查找操作的时间复杂度为 \(O(log n)\)，相比于线性遍历规则列表的 \(O(n)\) 复杂度，效率有显著提升。
- 应用场景及示例：
  - 对于防火墙规则，可按照 IP 地址构建二叉搜索树。例如，在匹配数据包时，根据数据包的源 IP 地址在二叉搜索树中进行查找，快速定位可能匹配的规则范围。不过，普通二叉搜索树在最坏情况下（如数据有序插入）会退化为链表，导致时间复杂度变为 \(O(n)\)。为了避免这种情况，可以使用平衡二叉搜索树，如 AVL 树或红黑树。

### 2. 前缀树（Trie Tree）

- 原理：
  - 前缀树又称字典树，是一种多叉树结构，常用于处理字符串匹配问题。在防火墙规则匹配中，可以将 IP 地址看作是由多个字节组成的字符串，将规则的 IP 地址信息构建成前缀树。
- 优势：
  - 前缀树在进行字符串匹配时具有很高的效率，尤其是对于 IP 地址的前缀匹配。通过前缀树可以快速定位到与数据包 IP 地址匹配的规则，时间复杂度为 \(O(m)\)，其中 m 是 IP 地址的长度。
- 应用场景及示例：
  - 当防火墙需要处理大量的 IP 地址前缀规则时，前缀树非常适用。例如，对于包含多个子网掩码的规则，通过前缀树可以快速判断数据包的 IP 地址是否属于某个子网。

### 3. 布隆过滤器（Bloom Filter）

- 原理：
  - 布隆过滤器是一种空间效率极高的概率型数据结构，用于判断一个元素是否存在于一个集合中。它通过多个哈希函数将元素映射到一个位数组中，并将相应的位设置为 1。在判断元素是否存在时，检查对应的位是否都为 1。
- 优势：
  - 布隆过滤器的空间效率非常高，且查询操作的时间复杂度为 \(O(k)\)，其中 k 是哈希函数的个数。它可以快速排除不可能匹配的规则，减少不必要的规则匹配操作。
- 应用场景及示例：
  - 在防火墙规则匹配中，可以使用布隆过滤器对规则的关键属性（如源 IP 地址、目的 IP 地址等）进行过滤。例如，在处理大量规则时，先使用布隆过滤器判断数据包的 IP 地址是否可能匹配某个规则，如果不可能匹配，则直接跳过后续的规则匹配过程，从而提高匹配效率。

### 4. 哈希数组映射前缀树（Hash Array Mapped Trie，HAMT）

- 原理：
  - HAMT 结合了哈希表和前缀树的优点，它通过哈希函数将键映射到一个数组中，数组中的每个元素可以是一个子树。在查找时，先通过哈希值定位到数组中的元素，然后在子树中进行进一步的查找。
- 优势：
  - HAMT 具有较好的空间效率和查找效率，它可以处理大规模的数据，并且在插入、删除和查找操作上都具有较高的性能。
- 应用场景及示例：
  - 在防火墙规则匹配中，当规则数量非常大时，HAMT 可以有效地组织规则，提高规则匹配的效率。例如，将规则的关键属性（如 IP 地址、端口号等）作为键，使用 HAMT 进行存储和查找。

# Linux 内核模块中，入口函数

在 Linux 内核模块中，入口函数的作用类似于普通程序的 `main` 函数，但由内核调用。它通过 `module_init` 宏声明，负责模块的初始化工作。

# 流程

### 1. 初始化阶段

- 内核模块初始化：
  - 在`my_firewall.c`中，通过`MODULE_LICENSE`和`MODULE_AUTHOR`声明模块的许可证和作者信息。
  - 定义了一些常量，如`NETLINK_TEST`、`CONNECT_STATUE_TIME`、`NAT_CONNECT_STATUE_TIME`等。
  - 初始化了一些全局变量，包括 Netlink 套接字`nlsk`、日志进程 ID `LOG_pid`，以及各种规则链表和连接链表等数据结构。
- 用户空间程序初始化：
  - 在`UI.c`和`log.c`中，通过`netlink_init`函数创建 Netlink 套接字，并进行绑定操作，以便与内核模块进行通信。

### 2. 数据包处理阶段

- 数据包过滤：
  - 在`my_firewall.c`的`telnet_filter`函数中，首先提取数据包的 IP 头信息，打印出数据包的源 IP、目的 IP 和协议类型。
  - 然后依次调用`connect_match`和`rule_match`函数进行连接匹配和规则匹配，如果匹配成功则允许数据包通过（返回`NF_ACCEPT`），否则丢弃数据包（返回`NF_DROP`）。
- NAT 处理：
  - 在`my_firewall.c`的`telnet_NAT_in`函数中，根据数据包的协议类型（TCP、UDP 或 ICMP）进行不同的处理。
  - 对于 TCP 和 UDP 协议，提取源端口和目的端口信息，遍历 NAT 连接链表`NAT_connect_index`，如果找到匹配的连接，则修改数据包的目的 IP 和目的端口，并更新连接的时间戳和校验和。
  - 对于 ICMP 协议，提取 ICMP 头信息，同样遍历 NAT 连接链表，找到匹配的连接后修改数据包的目的 IP 和校验和。

### 3. 日志记录阶段

- 内核模块日志记录：
  - 在`my_firewall.c`的`print_log`函数中，根据数据包的信息生成日志数据，然后通过`nltest_ksend`函数将日志数据发送给用户空间的日志程序。
- 用户空间日志记录：
  - 在`log.c`的`main`函数中，初始化 Netlink 套接字后，接收来自内核模块的日志数据。
  - 同时获取当前时间，将时间信息和日志数据组合成一条日志记录，写入到`log.txt`文件中。

### 4. 通信阶段

- Netlink 通信：
  - 内核模块和用户空间程序通过 Netlink 套接字进行通信，内核模块使用`nltest_ksend`函数发送数据，用户空间程序使用`netlink_send`函数发送数据，`recvfrom`函数接收数据。

### 5. 资源释放阶段

- 当模块卸载或程序退出时，需要释放相关的资源，如关闭 Netlink 套接字、释放链表内存等。

总体来说，这个项目通过内核模块和用户空间程序的协作，实现了对网络数据包的过滤、NAT 处理和日志记录功能。

# 中断

中断是 CPU 暂停当前任务，转而处理特定事件的机制。它允许外设、程序或系统在需要时主动 “打断” CPU，提高系统实时性。

![image-20250309140348228](Firewall问题.assets/image-20250309140348228-17417426368261.png)



![image-20250309140417099](Firewall问题.assets/image-20250309140417099-17417426380422.png)

## **系统调用结束后会进行进程调度！！！**

![image-20250309140358686](Firewall问题.assets/image-20250309140358686-17417426406323.png)

# NAT转换中检验和计算

**TCP/UDP：**

1. **构建伪头部**：将源 IP 地址、目的 IP 地址、协议号和 TCP 段长度按 16 位分组相加。
2. **分组求和**：将伪头部、TCP 头部和 **TCP 数据**按 16 位分组，把所有分组的值相加，如果相加过程中产生进位，则将进位加到结果的最低位。
3. **取反**：将求和结果取反，得到的结果就是 TCP 头部的检验和。

I**P 头部检验和只覆盖 IP 头部，不包括数据部分。**计算步骤如下：

1. **初始化**：将 IP 头部的检验和字段（位于 IP 头部偏移 10 字节处，占 2 字节）置为 0。
2. **分组求和**：将 IP 头部按 16 位进行分组，把所有分组的值相加，如果相加过程中产生进位，则将进位加到结果的最低位。
3. **取反**：将求和结果取反，得到的结果就是 IP 头部的检验和。