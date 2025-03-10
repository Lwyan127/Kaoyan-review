# UI.c

### main

1. nelink_init
2. 处理 ls rule：从./rule文件里找到所有rule并输出
3. 处理 ls connect：构造一个包含特定标识（`0x02020202`）的消息，并调用 `netlink_send` 函数将其发送给内核。然后收到内核的响应就打印所有连接的信息。
4. 处理 ls nat 和 ls connnect 与上面两个一样
5. 处理 ch rule：将 rule 通过 netlink 发送给内核
6. 处理 ch add：在 rule 文件里增加 rule
7. 处理 start：加载内核模块并更新规则
8. 处理 exit：退出
9. 关闭套接字

### netlink_init

- 调用socket创建一个netlink的套接字。

- 绑定套接字到本地地址。

### ipTint

将点分十进制表示的 IPv4 地址字符串转换为 32 位无符号整数形式的 IP 地址

### ipTchar

将 32 位无符号整数形式的 IPv4 地址转换为点分十进制表示的字符串形式

### charTint

将表示数字的字符串转换为无符号整数

### netlink_send

通过 Netlink 套接字向内核发送消息

### file_wc

打开文件，以每次读取 50000 字节的方式将文件内容读入缓冲区，然后遍历缓冲区中的字符，统计换行符 `'\n'` 的数量，从而得到文件的行数

# log.c

### main

初始化另一个netlink套接字，持续从 Netlink 套接字接收内核发送的消息，并将这些消息添加时间戳后记录到 `log.txt` 文件中，同时在控制台输出日志信息。

### netlink_init

### netlink_send

# my_firewall.c

### myfw_init

1. 初始化netlink套接字netlink_kernel_create()
2. 注册防火墙钩子函数（fw_out.hook = telnet_filter;）
3. 注册NAT钩子函数（nat_in.hook = telnet_NAT_in;）

### myfw_exit

1. 释放netlink套接字和注销防火墙和钩子函数

### telnet_filter

一旦钩子函数收到了一个报文，就会调用这个函数。

这个函数调用`connect_macth`函数决定放不放行。

### nltest_krecv（netlink 接收函数）

nltest_krecv在netlink_kernel_cfg结构体中被注册，也就是一旦消息到达netlink套接字，就会调用这个函数。

1. 接收 Netlink 消息的所有信息
2.  根据消息内容进行不同处理
   - `0x02020202`：遍历 `Connect_index` 数组，清理超时的连接状态。将有效的连接状态信息复制到 `send_data` 数组中。调用 `nltest_ksend` 函数将连接状态信息发送回用户空间。
   - `0x03030303`：将发送消息的用户空间进程的 PID 存储到 `LOG_pid` 变量中。
   - `0x04040404`：调用 `mepty_nat_connect` 函数清空 NAT 连接状态。从消息数据中提取 NAT 列表的数量 `nat_list_sum`。将消息数据中的 NAT 信息复制到 `nat_list` 数组中，并打印每条 NAT 信息。
   - `0x05050505`：遍历 `NAT_connect_index` 数组，清理超时的 NAT 连接状态。将有效的 NAT 连接状态信息复制到 `send_data` 数组中。调用 `nltest_ksend` 函数将 NAT 转换表信息发送回用户空间。

### print_log

`print_log` 函数的主要功能是生成一条网络连接日志信息，并通过 Netlink 套接字将该日志信息发送给用户空间中指定 PID 的进程。该日志信息记录了网络连接的源 IP 地址、源端口、目的 IP 地址、目的端口、协议类型以及连接的处理规则（接受或丢弃）。

### delete_connect

`delete_connect` 函数的主要功能是从 `Connect_list` 链表中删除指定的连接状态节点。

### connect_match

`connect_match` 函数用于对网络数据包进行连接匹配操作。它会检查传入的网络数据包（通过 `struct sk_buff *skb` 表示）是否与已记录的连接状态相匹配。如果匹配成功，则更新连接状态的时间戳，并根据配置决定是否记录日志；如果是 ICMP 协议且匹配成功，还会删除对应的连接状态记录。

**哈希表重点：**通过对处理后的源 IP 和目的 IP 地址的高 16 位进行异或运算，得到一个索引值，从 `Connect_index` 数组中获取对应的连接状态链表头指针。

**`Connect_index` 数组是一个链表头数组，数组每个元素都是链表头，其下标为源IP和目的IP地址高16位的异或运算，这样每个链表都是一对源和目的的连接，最大为2^16^ =65536，因此这个数组设为66000的大小，这样就是一个可以通过下标直接找到数组元素的哈希表，NAT_connect_list也是相同的哈希表，使用目的 IP 地址（`daddr`）和目的端口号（`dport`）计算出一个索引值**

### update_ip_checksum等等update什么什么的函数

全是用来NAT转换时重新计算udp、tcp等等头校验和，注意要先把传输层的头部算了再来计算网络层的头部。

### generate_random_port

`generate_random_port` 函数的主要功能是为指定协议（TCP 或 UDP）生成一个在 20000 到 40000 范围内的随机端口号，并尝试将该端口号绑定到指定的外部 IP 地址。如果绑定成功，则将该端口号返回，并将绑定的套接字指针存储在 `NAT_connect_list` 结构体中；如果绑定失败，则继续生成新的随机端口号并尝试绑定，直到找到一个可用的端口号为止。

### `delete_nat_connect` 函数

此函数的作用是从 NAT 连接状态链表中删除指定的节点。在删除节点之前，会处理链表指针的调整，确保链表的连续性，同时会释放节点占用的套接字资源，最后释放节点本身的内存。

### `mepty_nat_connect` 函数

该函数的功能是清空所有 NAT 连接状态。通过遍历 `NAT_connect_index` 数组，将数组中的每个元素都置为 `NULL`，从而清空所有链表头指针，实现清空 NAT 连接状态的目的。

### telnet_NAT_out/telnet_NAT_in

**NAT_connect_list也是相同的哈希表，使用目的 （源）IP 地址（`daddr`）和目的（源）端口号（`dport`）计算出一个索引值**

进行NAT转换即可。