# RDMA 学习笔记

> 本文档基于 [RDMA-tutorial](https://github.com/StarryVae/RDMA-tutorial) 项目，记录 RDMA 入门学习过程中的所有知识点。

---

## 目录

- [一、核心概念——心智模型](#一核心概念心智模型)
- [二、用 IBV Verbs 写 SEND/RECV 程序](#二用-ibv-verbs-写-sendrecv-程序)
- [三、QP 状态转换详解](#三qp-状态转换详解)
- [四、RDMA WRITE 与 SEND 的区别](#四rdma-write-与-send-的区别)
- [五、Makefile 逐行解析](#五makefile-逐行解析)
- [六、为什么不能只使用 Unsignal](#六为什么不能只使用-unsignal)
- [七、C 宏技巧：TEST_NZ/TEST_Z 与 do-while(0)](#七c-宏技巧test_nztest_z-与-do-while0)
- [八、三种 Context 详解](#八三种-context-详解)
- [九、rdmacm 库的作用](#九rdmacm-库的作用)
- [十、Server 完整流程逐函数解析](#十server-完整流程逐函数解析)

---

## 一、核心概念——心智模型

### 1.1 RDMA 是什么

传统网络传输：`应用 → 内核协议栈(TCP/IP) → 网卡 → 网络 → 网卡 → 内核协议栈 → 应用`

RDMA 传输：`应用 → 网卡(RNIC) → 网络 → 网卡(RNIC) → 应用`

核心优势：**绕过内核（kernel bypass）**，数据直接从用户态内存发到网卡，不需要内核拷贝，也不需要 CPU 参与协议栈处理。结果是：延时 ~1.5 微秒（传统 TCP 是几十到几百微秒）。

### 1.2 四大核心对象

| 对象 | 全称 | 类比 | 作用 |
|------|------|------|------|
| **PD** | Protection Domain | 命名空间 | 隔离资源，确保安全 |
| **QP** | Queue Pair (SQ+RQ) | socket fd | 发送队列 + 接收队列，是收发数据的通道 |
| **CQ** | Completion Queue | epoll | 通知你"数据发完了"或"收到了" |
| **MR** | Memory Region | 已注册的缓冲区 | 让网卡知道它可以直接访问哪块内存 |

---

## 二、用 IBV Verbs 写 SEND/RECV 程序

项目中的 `example code/rdma_send_recv/` 是一个完整的 SEND/RECV 示例。

### 步骤 1：初始化环境——创建 PD、CQ、完成通道

Server 和 Client 的 `build_context()` 函数做同一件事（以 client.c 为例）：

```c
void build_context(struct ibv_context *verbs)
{
  // 如果已经初始化过就跳过
  if (s_ctx) {
    if (s_ctx->ctx != verbs)
      die("cannot handle events in more than one context.");
    return;
  }

  s_ctx = (struct context *)malloc(sizeof(struct context));
  s_ctx->ctx = verbs;

  TEST_Z(s_ctx->pd = ibv_alloc_pd(s_ctx->ctx));                         // 1. 创建保护域
  TEST_Z(s_ctx->comp_channel = ibv_create_comp_channel(s_ctx->ctx));     // 2. 创建完成通道
  TEST_Z(s_ctx->cq = ibv_create_cq(s_ctx->ctx, 10, NULL, s_ctx->comp_channel, 0)); // 3. 创建完成队列
  TEST_NZ(ibv_req_notify_cq(s_ctx->cq, 0));                             // 4. 注册事件通知

  TEST_NZ(pthread_create(&s_ctx->cq_poller_thread, NULL, poll_cq, NULL)); // 5. 启动轮询线程
}
```

5 件事：
1. `ibv_alloc_pd()` — 创建保护域（所有资源必须属于同一个 PD）
2. `ibv_create_comp_channel()` — 创建完成通道（用于事件通知机制）
3. `ibv_create_cq()` — 创建完成队列（容量 10 个 CQE）
4. `ibv_req_notify_cq()` — 注册事件通知（收到 CQE 时会通过 comp_channel 通知）
5. 启动一个线程 `poll_cq` 来轮询完成事件

### 步骤 2：创建 QP（Queue Pair）

```c
void build_qp_attr(struct ibv_qp_init_attr *qp_attr)
{
  memset(qp_attr, 0, sizeof(*qp_attr));

  qp_attr->send_cq = s_ctx->cq;
  qp_attr->recv_cq = s_ctx->cq;
  qp_attr->qp_type = IBV_QPT_RC;      // 可靠连接，类似 TCP

  qp_attr->cap.max_send_wr = 10;       // 发送队列最多 10 个请求
  qp_attr->cap.max_recv_wr = 10;       // 接收队列最多 10 个请求
  qp_attr->cap.max_send_sge = 1;       // 每个请求最多 1 个 scatter-gather 元素
  qp_attr->cap.max_recv_sge = 1;
}
```

然后通过 `rdma_create_qp(id, s_ctx->pd, &qp_attr)` 创建 QP。

### 步骤 3：注册内存（Memory Registration）

```c
void register_memory(struct connection *conn)
{
  conn->send_region = malloc(BUFFER_SIZE);
  conn->recv_region = malloc(BUFFER_SIZE);

  TEST_Z(conn->send_mr = ibv_reg_mr(
    s_ctx->pd, conn->send_region, BUFFER_SIZE,
    IBV_ACCESS_LOCAL_WRITE | IBV_ACCESS_REMOTE_WRITE));

  TEST_Z(conn->recv_mr = ibv_reg_mr(
    s_ctx->pd, conn->recv_region, BUFFER_SIZE,
    IBV_ACCESS_LOCAL_WRITE | IBV_ACCESS_REMOTE_WRITE));
}
```

**为什么需要注册内存？** 因为 RDMA 网卡需要直接访问用户态内存，必须：
1. 把虚拟地址映射固定（pin）到物理地址，防止被换页
2. 让网卡知道这块内存的物理地址
3. 获得 `lkey`（本地访问密钥）和 `rkey`（远端访问密钥）

### 步骤 4：建立连接

**Server 端**：
```c
// 类比 socket: socket() → bind() → listen()
TEST_Z(ec = rdma_create_event_channel());
TEST_NZ(rdma_create_id(ec, &listener, NULL, RDMA_PS_TCP));
TEST_NZ(rdma_bind_addr(listener, (struct sockaddr *)&addr));
TEST_NZ(rdma_listen(listener, 10));

// 收到连接请求后 accept
TEST_NZ(rdma_accept(id, &cm_params));
```

**Client 端**：
```c
TEST_Z(ec = rdma_create_event_channel());
TEST_NZ(rdma_create_id(ec, &conn, NULL, RDMA_PS_TCP));
TEST_NZ(rdma_resolve_addr(conn, NULL, addr->ai_addr, TIMEOUT_IN_MS));
// 地址解析完成 → 路由解析 → 发起连接
TEST_NZ(rdma_connect(id, &cm_params));
```

### 步骤 5：发起接收请求（Post Receive）

```c
void post_receives(struct connection *conn)
{
  struct ibv_recv_wr wr, *bad_wr = NULL;
  struct ibv_sge sge;

  wr.wr_id = (uintptr_t)conn;
  wr.next = NULL;
  wr.sg_list = &sge;
  wr.num_sge = 1;

  sge.addr = (uintptr_t)conn->recv_region;  // 往哪写
  sge.length = BUFFER_SIZE;                   // 最多写多少
  sge.lkey = conn->recv_mr->lkey;            // 本地访问密钥

  TEST_NZ(ibv_post_recv(conn->qp, &wr, &bad_wr));
}
```

**核心原则：接收端必须先 post receive，发送端才能 send。** 否则会导致 RNR（Receiver Not Ready）错误。

### 步骤 6：发送数据（Post Send）

```c
int on_connection(void *context)
{
  struct connection *conn = (struct connection *)context;
  struct ibv_send_wr wr, *bad_wr = NULL;
  struct ibv_sge sge;

  snprintf(conn->send_region, BUFFER_SIZE, "message from active/client side with pid %d", getpid());

  memset(&wr, 0, sizeof(wr));
  wr.wr_id = (uintptr_t)conn;
  wr.opcode = IBV_WR_SEND;               // 操作类型：SEND
  wr.sg_list = &sge;
  wr.num_sge = 1;
  wr.send_flags = IBV_SEND_SIGNALED;     // 完成后产生 CQE

  sge.addr = (uintptr_t)conn->send_region;
  sge.length = BUFFER_SIZE;
  sge.lkey = conn->send_mr->lkey;

  TEST_NZ(ibv_post_send(conn->qp, &wr, &bad_wr));
  return 0;
}
```

### 步骤 7：处理完成事件

```c
void * poll_cq(void *ctx)
{
  struct ibv_cq *cq;
  struct ibv_wc wc;

  while (1) {
    TEST_NZ(ibv_get_cq_event(s_ctx->comp_channel, &cq, &ctx));  // 阻塞等待事件
    ibv_ack_cq_events(cq, 1);                                    // 确认事件
    TEST_NZ(ibv_req_notify_cq(cq, 0));                           // 重新注册通知

    while (ibv_poll_cq(cq, 1, &wc))   // 取出完成的工作请求
      on_completion(&wc);              // 处理
  }
  return NULL;
}
```

这是 **event-triggered polling**（事件触发式轮询）。另一种方式是 **busy polling**（忙轮询），直接循环调用 `ibv_poll_cq()`，延时更低但 CPU 占用更高。

### 完整流程总结图

```
Server                                    Client
──────                                    ──────
create_event_channel                      create_event_channel
create_id                                 create_id
bind_addr + listen                        resolve_addr
     |                                         |
     |  <── CONNECT_REQUEST ──                 |
     |                                    resolve_route
on_connect_request:                            |
  build_context (PD/CQ)                   on_addr_resolved:
  create_qp                                 build_context (PD/CQ)
  register_memory                           create_qp
  post_receives ← 关键！先准备好接收         register_memory
  rdma_accept                                  |
     |                                    on_route_resolved:
     |  ── ESTABLISHED ──>                  rdma_connect
     |                                         |
     |                                    on_connection:
     |                                      post_send (IBV_WR_SEND)
     |  <── 数据在网卡层直接传输 ──            |
on_completion:                            on_completion:
  IBV_WC_RECV → 读取数据                    IBV_WC_SEND → 发送完成
```

---

## 三、QP 状态转换详解

QP 有一个严格的状态机，必须按顺序转换：

```
RESET → INIT → RTR → RTS → [数据传输中] → ERROR/RESET
```

| 转换 | 设置了什么 | 为什么需要 |
|------|-----------|-----------|
| **RESET → INIT** | Port 号、PKey、访问权限标志 | 告诉网卡：这个 QP 绑定在哪个物理端口上，允许哪些远程操作（Read/Write）。此时 QP 可以开始 **post receive**（但还不能发送） |
| **INIT → RTR** (Ready to Receive) | 对端 QP 号、对端 LID/GID、MTU、RNR 重试次数 | 告诉网卡：我要跟谁通信（对端的身份信息）。此时网卡知道了对端地址，**可以接收数据了** |
| **RTR → RTS** (Ready to Send) | 超时、重试次数、起始 PSN | 配置发送侧参数。此时 QP **既能收也能发**，进入正式工作状态 |

**为什么不能跳过？**
- 不到 INIT，你连 post receive 都不行（网卡不知道绑定哪个端口）
- 不到 RTR，网卡不知道对端是谁，收到的数据无法匹配 QP
- 不到 RTS，发送队列不会被处理，`ibv_post_send()` 发出去的请求不会被执行

在本项目中，`rdma_connect()` 和 `rdma_accept()` 内部自动完成了 `INIT → RTR → RTS` 的转换，并自动交换了双方的 QP 信息。如果不使用 `rdma_cm`，你需要手动调用 `ibv_modify_qp()` 三次来完成这些转换，并通过 TCP socket 交换对端信息。

---

## 四、RDMA WRITE 与 SEND 的区别

### 4.1 根本区别：对端 CPU 是否参与

| 特性 | SEND/RECV（双边操作） | RDMA WRITE（单边操作） |
|------|----------------------|----------------------|
| **对端 CPU** | 需要参与（必须先 post receive） | **不需要参与**（远端 CPU 完全不知道） |
| **对端准备** | 必须提前 post_recv | 只需一次性交换内存地址和 rkey |
| **CQE** | 双方都产生 | 只有发起端产生（远端无感知） |
| **类比** | 打电话（双方都得拿起话筒） | 直接往对方的笔记本上写字（对方不用知道） |
| **适用场景** | 控制消息、小数据 | 大批量数据传输、分布式存储 |

### 4.2 代码对比

**SEND 操作**（from `rdma_send_recv/client.c`）：

```c
// 发送端
wr.opcode = IBV_WR_SEND;        // 操作类型：SEND
wr.sg_list = &sge;              // 只需要指定本地数据
ibv_post_send(conn->qp, &wr, &bad_wr);

// 接收端必须提前：
ibv_post_recv(conn->qp, &wr, &bad_wr);  // 告诉网卡"把数据放这里"
```

**RDMA WRITE 操作**（from `rdma_write_read/rdma-common.c`）：

```c
wr.opcode = IBV_WR_RDMA_WRITE;
wr.sg_list = &sge;
wr.send_flags = IBV_SEND_SIGNALED;
wr.wr.rdma.remote_addr = (uintptr_t)conn->peer_mr.addr;   // 远端内存地址
wr.wr.rdma.rkey = conn->peer_mr.rkey;                      // 远端访问密钥

sge.addr = (uintptr_t)conn->rdma_local_region;
sge.length = RDMA_BUFFER_SIZE;
sge.lkey = conn->rdma_local_mr->lkey;

ibv_post_send(conn->qp, &wr, &bad_wr);
```

关键差异：WRITE 比 SEND 多了**远端地址**（`remote_addr`）和**远端密钥**（`rkey`）。

### 4.3 WRITE 的前置步骤：交换内存信息

因为 WRITE 需要知道远端的内存地址和 rkey，所以双方需要先用 SEND/RECV 交换这些信息：

```c
void send_mr(void *context)
{
  struct connection *conn = (struct connection *)context;
  conn->send_msg->type = MSG_MR;
  memcpy(&conn->send_msg->data.mr, conn->rdma_remote_mr, sizeof(struct ibv_mr));
  send_message(conn);
}
```

### 4.4 内存注册的权限差异

```c
// SEND/RECV 只需要本地写权限
IBV_ACCESS_LOCAL_WRITE

// RDMA WRITE 需要远端写权限
IBV_ACCESS_LOCAL_WRITE | IBV_ACCESS_REMOTE_WRITE

// RDMA READ 需要远端读权限
IBV_ACCESS_LOCAL_WRITE | IBV_ACCESS_REMOTE_READ
```

### 4.5 状态机驱动的协作流程（rdma_write_read 示例）

```
发送状态: SS_INIT → SS_MR_SENT → SS_RDMA_SENT → SS_DONE_SENT
接收状态: RS_INIT → RS_MR_RECV → RS_DONE_RECV

完整流程：
1. 双方建立连接
2. 双方通过 SEND/RECV 交换各自的 MR 信息（addr + rkey）
3. 当 send_state == SS_MR_SENT && recv_state == RS_MR_RECV 时：
   → 执行 RDMA WRITE（直接写入远端内存）
   → 发送 MSG_DONE 通知对端
4. 当 send_state == SS_DONE_SENT && recv_state == RS_DONE_RECV 时：
   → 打印远端缓冲区内容，断开连接
```

---

## 五、Makefile 逐行解析

文件：`example code/rdma_send_recv/Makefile`

```makefile
.PHONY: clean
```
`.PHONY` 声明 `clean` 是一个"伪目标"——即使目录下有叫 `clean` 的文件，`make clean` 也会无条件执行。

```makefile
CFLAGS  := -Wall -g
```
编译选项：`-Wall` 开启所有常见警告，`-g` 生成调试信息。`:=` 是立即赋值。虽然下面没有显式使用 `${CFLAGS}`，但 Make 的隐式规则在 `.c → .o` 时会自动使用。

```makefile
LD      := gcc
```
链接器设为 `gcc`。

```makefile
LDLIBS  := ${LDLIBS} -lrdmacm -libverbs -lpthread
```
链接库——RDMA 编程的"标准三件套"：
- `-lrdmacm` — RDMA 连接管理 API
- `-libverbs` — RDMA 核心操作 API
- `-lpthread` — POSIX 线程库（用于 CQ 轮询线程）

```makefile
APPS    := server client
all: ${APPS}
```
默认目标 `all`，构建 `server` 和 `client` 两个程序。

```makefile
client: client.o
        ${LD} -o $@ $^ ${LDLIBS}

server: server.o
        ${LD} -o $@ $^ ${LDLIBS}
```
`$@` = 目标名，`$^` = 所有依赖项。`.c → .o` 由 Make 隐式规则自动完成。

```makefile
clean:
        rm -f ${APPS}
```
清理可执行文件。（小瑕疵：没有清理 `.o` 中间文件）

**实际执行的完整命令：**
```bash
gcc -Wall -g -c -o client.o client.c
gcc -o client client.o -lrdmacm -libverbs -lpthread
gcc -Wall -g -c -o server.o server.c
gcc -o server server.o -lrdmacm -libverbs -lpthread
```

---

## 六、为什么不能只使用 Unsignal

### 6.1 Signal 与 Unsignal

- **Signaled**（`IBV_SEND_SIGNALED`）：Send 完成后，网卡往 CQ 放一个 CQE
- **Unsignaled**（不设标志）：完成后不产生 CQE，静默完成

Unsignal 好处：不产生 CQE = 不需要 poll = 减少 CPU 开销 = 降低延时。

### 6.2 全部 Unsignal 的灾难

**网卡只有在你 poll 到对应的 CQE 之后，才会释放 SQ 中该 Work Request 占用的槽位。**

```
第 1 次 post_send (unsignal)  → SQ: [wr1]                    ← 没有 CQE，无法回收
第 2 次 post_send (unsignal)  → SQ: [wr1][wr2]               ← 没有 CQE，无法回收
  ...
第10 次 post_send (unsignal)  → SQ: [wr1][wr2]...[wr10]      ← 队列满了！
第11 次 post_send             → ❌ 失败！SQ 已满，无法再 post
```

因为没有 CQE，你永远不会 poll CQ，网卡也永远不会释放槽位。队列被"幽灵请求"填满。

### 6.3 正确做法：定期插入 Signaled 请求

```
post_send (unsignal)   → SQ: [wr1]
post_send (unsignal)   → SQ: [wr1][wr2]
  ...
post_send (SIGNALED)   → SQ: [wr1]...[wr10]  ← 产生 CQE
                           ↓
poll_cq()              → 取出 CQE
                           ↓
                         SQ: [ ][ ]...[ ][ ]  ← 网卡释放 wr1~wr10 所有槽位！
```

**一个 Signaled 的 CQE 被 poll 出来后，它之前所有的 Unsignaled WR 也会被一并回收。**

### 6.4 工程实践

```c
#define SIGNAL_INTERVAL 5

int send_count = 0;

void do_send(struct connection *conn) {
    struct ibv_send_wr wr = {0};
    // ... 设置其他字段 ...

    send_count++;
    if (send_count % SIGNAL_INTERVAL == 0) {
        wr.send_flags = IBV_SEND_SIGNALED;  // 每 5 个 signal 一次
    } else {
        wr.send_flags = 0;                   // unsignal
    }
    ibv_post_send(conn->qp, &wr, &bad_wr);
}
```

> 类比：Unsignal 只管往垃圾桶（SQ）扔垃圾不清理，Signaled 相当于通知清洁工来倒垃圾桶。永远不通知，垃圾桶就满了。

---

## 七、C 宏技巧：TEST_NZ/TEST_Z 与 do-while(0)

### 7.1 宏定义

```c
#define TEST_NZ(x) do { if ( (x)) die("error: " #x " failed (returned non-zero)." ); } while (0)
#define TEST_Z(x)  do { if (!(x)) die("error: " #x " failed (returned zero/null)."); } while (0)
```

### 7.2 `die()` 函数

```c
void die(const char *reason)
{
  fprintf(stderr, "%s\n", reason);
  exit(EXIT_FAILURE);
}
```

打印错误信息，直接终止程序。

### 7.3 两个宏的区别

- **`TEST_NZ`**（Test Non-Zero）— 返回值非零就报错。用于"成功返回 0"的函数（`rdma_bind_addr`、`ibv_post_send` 等）
- **`TEST_Z`**（Test Zero）— 返回值为零/NULL 就报错。用于"成功返回指针/非零值"的函数（`ibv_alloc_pd`、`ibv_create_cq` 等）

### 7.4 `#x` 字符串化操作符

`#x` 把宏参数原样变成字符串，报错时能看到具体哪个函数失败了：

```c
TEST_NZ(rdma_listen(listener, 10));
// 展开后：
do { if (rdma_listen(listener, 10)) die("error: rdma_listen(listener, 10) failed (returned non-zero)."); } while (0);
```

### 7.5 为什么用 `do { ... } while(0)`

目的：让宏在任何语法环境下都安全。

**不用的话会出问题：**
```c
#define TEST_NZ(x) if ((x)) die("error")

if (condition)
    TEST_NZ(some_func());
else                    // ← else 会和内层 if 配对，逻辑全错！
    do_something();
```

**用了 `do { ... } while(0)` 后：**
```c
if (condition)
    do { if ((some_func())) die("error"); } while (0);
else                    // ← 明确与外层 if 配对
    do_something();
```

`while(0)` 条件永远为假，循环体只执行一次，没有运行时开销。

---

## 八、三种 Context 详解

`server.c` 中出现了三种不同的 "context"，含义完全不同。

### 8.1 `struct ibv_context`——网卡的"身份证"

```c
struct context {
  struct ibv_context *ctx;   // ← 这个
  ...
};
```

代表**一块 RDMA 网卡设备**。后续所有操作都需要指定在哪块网卡上进行。
- 类比：去银行办业务，`ibv_context` 就是"哪个银行网点"。

### 8.2 自定义 `struct context`——"全局工具箱"

```c
struct context {
  struct ibv_context *ctx;               // 网卡设备（在哪家银行）
  struct ibv_pd *pd;                     // 保护域（银行账户）
  struct ibv_cq *cq;                     // 完成队列（叫号屏）
  struct ibv_comp_channel *comp_channel; // 完成通道（叫号喇叭）
  pthread_t cq_poller_thread;            // 轮询线程（盯着叫号屏的员工）
};
```

项目自定义的结构体，把所有全局共享的 RDMA 资源打包在一起。全局只有一个实例 `s_ctx`。

### 8.3 `id->context`——连接上的"附件口袋"

`rdma_cm_id` 里有一个 `void *context` 字段——库**留给你的空指针**，让你挂自定义数据。

```c
// 绑定
id->context = conn;   // 把 conn 塞进口袋

// 取回
struct connection *conn = (struct connection *)id->context;  // 从口袋里拿出来
```

这样在事件回调中，拿到 `id` 就能取回对应连接的所有数据。

### 8.4 三者关系图

```
┌─────────────────────────────────────────────────────────┐
│  struct context (全局唯一，全局工具箱)                      │
│  ┌──────────────────────┐                               │
│  │ ibv_context *ctx ────────→ [RDMA 网卡设备]            │
│  │ ibv_pd *pd           │    （身份证/银行网点）           │
│  │ ibv_cq *cq           │                               │
│  │ comp_channel          │                               │
│  │ cq_poller_thread     │                               │
│  └──────────────────────┘                               │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────┐     ┌─────────────────────┐
│ rdma_cm_id (连接1)       │     │ rdma_cm_id (连接2)   │
│                         │     │                     │
│ void *context ──────┐   │     │ void *context ──┐   │
│ (口袋/钩子)          │   │     │                  │   │
└─────────────────────│───┘     └──────────────────│───┘
                      ↓                            ↓
              ┌───────────────┐          ┌───────────────┐
              │ connection    │          │ connection    │
              │  qp           │          │  qp           │
              │  send_mr      │          │  send_mr      │
              │  recv_mr      │          │  recv_mr      │
              │  send_region  │          │  send_region  │
              │  recv_region  │          │  recv_region  │
              └───────────────┘          └───────────────┘
               (连接1的私有数据)            (连接2的私有数据)
```

---

## 九、rdmacm 库的作用

### 9.1 核心作用

rdmacm（RDMA Communication Manager）的核心作用：**帮你在两台机器的 QP 之间建立连接。**

### 9.2 没有 rdmacm 的繁琐过程

```
1. 创建 QP，得到 QP number
2. 查询本地 LID/GID
3. 自己写 TCP socket 交换双方的 QP号、LID、GID
4. ibv_modify_qp(RESET → INIT)  — 填入 port, pkey, access_flags
5. ibv_modify_qp(INIT → RTR)    — 填入对端QP号, 对端LID, MTU...
6. ibv_modify_qp(RTR → RTS)     — 填入 timeout, retry_cnt, PSN...
7. 再用 TCP 同步确认双方都就绪
```

### 9.3 有了 rdmacm 之后

```
传统 Socket              rdmacm                     帮你做了什么
──────────              ──────                     ──────────────
socket()          rdma_create_id()                创建连接标识
bind()            rdma_bind_addr()                绑定地址
listen()          rdma_listen()                   监听连接
connect()         rdma_connect()         ← 自动交换QP信息 + 自动 INIT→RTR→RTS
accept()          rdma_accept()          ← 自动交换QP信息 + 自动 INIT→RTR→RTS
close()           rdma_disconnect()               断开连接
```

### 9.4 两个库的分工

```
┌─────────────────────────────────────────────────────────┐
│                    你的应用程序                            │
├──────────────────────┬──────────────────────────────────┤
│     librdmacm        │         libibverbs               │
│  （连接管理层）        │       （数据传输层）                │
│                      │                                  │
│  rdma_create_id()    │  ibv_alloc_pd()                  │
│  rdma_resolve_addr() │  ibv_create_cq()                 │
│  rdma_connect()      │  ibv_reg_mr()                    │
│  rdma_accept()       │  ibv_post_send()                 │
│  rdma_disconnect()   │  ibv_post_recv()                 │
│                      │  ibv_poll_cq()                   │
│  负责：建立/断开连接   │  负责：注册内存、收发数据、轮询完成  │
├──────────────────────┴──────────────────────────────────┤
│                  RDMA 网卡硬件 (RNIC)                     │
└─────────────────────────────────────────────────────────┘
```

- **libibverbs**：核心数据面 API。**必须用。**
- **librdmacm**：连接管理 API。**可以不用，但不用就得自己写 TCP + 手动转 QP 状态。**

---

## 十、Server 完整流程逐函数解析

### 10.1 执行流程总览

```
main()
  │
  ├─ 1. 创建监听
  │
  └─ 2. 事件循环 ──→ on_event() 分发事件
                        │
                        ├─ CONNECT_REQUEST ──→ on_connect_request()
                        │                        ├─ build_context()     ← 初始化全局资源（只执行一次）
                        │                        ├─ build_qp_attr()     ← 配置 QP 参数
                        │                        ├─ rdma_create_qp()    ← 创建 QP
                        │                        ├─ register_memory()   ← 注册内存
                        │                        ├─ post_receives()     ← 提前挂好接收请求
                        │                        └─ rdma_accept()       ← 接受连接
                        │
                        ├─ ESTABLISHED ──→ (被注释掉了，什么也不做)
                        │
                        └─ DISCONNECTED ──→ on_disconnect()  ← 清理所有资源

  [另一个线程] poll_cq() ──→ on_completion()  ← 处理收发完成事件
```

### 10.2 阶段一：启动监听——`main()`

```c
memset(&addr, 0, sizeof(addr));
addr.sin_family = AF_INET;
addr.sin_port = htons(22222);
```
配置监听地址：IPv4，端口 22222。

```c
TEST_Z(ec = rdma_create_event_channel());
```
创建事件通道——后续所有连接管理事件从这里取。

```c
TEST_NZ(rdma_create_id(ec, &listener, NULL, RDMA_PS_TCP));
```
创建 `rdma_cm_id`，相当于 `socket()`。

```c
TEST_NZ(rdma_bind_addr(listener, (struct sockaddr *)&addr));
```
绑定地址和端口，相当于 `bind()`。

```c
TEST_NZ(rdma_listen(listener, 10));
```
开始监听，backlog=10，相当于 `listen()`。

```c
while (rdma_get_cm_event(ec, &event) == 0) {
    struct rdma_cm_event event_copy;
    memcpy(&event_copy, event, sizeof(*event));
    rdma_ack_cm_event(event);
    if (on_event(&event_copy))
        break;
}
```
**主事件循环**：阻塞等待事件 → 拷贝（因为 ack 会释放原事件内存）→ ack → 分发处理。

### 10.3 阶段二：事件分发——`on_event()`

```c
int on_event(struct rdma_cm_event *event)
{
  if (event->event == RDMA_CM_EVENT_CONNECT_REQUEST)
    r = on_connect_request(event->id);
  else if (event->event == RDMA_CM_EVENT_ESTABLISHED)
    r = 0;    // 被注释掉了
  else if (event->event == RDMA_CM_EVENT_DISCONNECTED)
    r = on_disconnect(event->id);
}
```

返回 0 继续循环，返回非 0 退出主循环。

### 10.4 阶段三：核心——`on_connect_request()`

Client 的 `rdma_connect()` 触发此事件。按执行顺序：

1. **`build_context(id->verbs)`** — 初始化全局资源（PD/CQ/comp_channel/轮询线程），只执行一次
2. **`build_qp_attr(&qp_attr)`** — 配置 QP 参数（RC 类型，队列容量各 10）
3. **`rdma_create_qp()`** — 创建 QP，此时 QP 处于 RESET 状态
4. **创建 `connection`** — 分配连接私有数据，挂到 `id->context`
5. **`register_memory()`** — 分配并注册 send/recv 两块缓冲区
6. **`post_receives()`** — 在接收队列挂好接收请求（**必须在 accept 之前**）
7. **`rdma_accept()`** — 接受连接，自动完成 RESET → INIT → RTR → RTS

### 10.5 阶段四：后台线程——`poll_cq()` + `on_completion()`

**`poll_cq()`** 每轮做 4 件事：

| 步骤 | 函数 | 做了什么 |
|------|------|---------|
| 1 | `ibv_get_cq_event()` | 阻塞等待 CQ 中出现新 CQE |
| 2 | `ibv_ack_cq_events()` | 确认通知（不确认会内存泄漏） |
| 3 | `ibv_req_notify_cq()` | 重新注册通知（通知是一次性的） |
| 4 | `ibv_poll_cq()` + `on_completion()` | 取出所有 CQE 逐个处理 |

**`on_completion()`** 处理逻辑：
- `wc->status != IBV_WC_SUCCESS` → 报错退出
- `wc->opcode & IBV_WC_RECV` → 接收完成，打印 `recv_region` 中的数据
- `wc->opcode == IBV_WC_SEND` → 发送完成

### 10.6 阶段五：清理——`on_disconnect()`

按创建的逆序释放所有资源：

```
1. rdma_destroy_qp     ← 销毁 QP
2. ibv_dereg_mr × 2    ← 注销内存注册
3. free × 2            ← 释放缓冲区
4. free(conn)          ← 释放连接结构体
5. rdma_destroy_id     ← 销毁连接标识
```

### 10.7 关于被注释掉的 `on_connection()`

`ESTABLISHED` 事件中的 `on_connection()` 被注释掉了：

```c
else if (event->event == RDMA_CM_EVENT_ESTABLISHED)
//    r = on_connection(event->id->context);
    r = 0;
```

`on_connection()` 的功能是连接建立后向 Client 发一条消息。被注释说明当前示例是 **Client 单方面发消息，Server 只接收**。如果要双向通信，取消注释即可（但 Client 也需要先 `post_receives()`）。

### 10.8 完整时间线

```
时间 ──→

Server main()                                      后台 poll_cq 线程
─────────────                                      ─────────────────
rdma_create_event_channel()
rdma_create_id()
rdma_bind_addr(:22222)
rdma_listen()
  │
  │  等待事件...
  │
  ├── 收到 CONNECT_REQUEST
  │     build_context()                             poll_cq() 启动
  │       ibv_alloc_pd()                              │
  │       ibv_create_comp_channel()                    │ ibv_get_cq_event()
  │       ibv_create_cq()                              │ [阻塞等待中...]
  │       ibv_req_notify_cq()                          │
  │       pthread_create(poll_cq)  ───启动──→          │
  │     build_qp_attr(RC, 10, 10)                      │
  │     rdma_create_qp()                               │
  │     register_memory() → send_mr + recv_mr          │
  │     post_receives() → 挂一个接收请求到 RQ          │
  │     rdma_accept() → QP: RESET→INIT→RTR→RTS        │
  │                                                    │
  ├── 收到 ESTABLISHED                                 │
  │     (什么都不做)                                    │
  │                                                    │
  │    [Client 发数据过来，网卡直接写入 recv_region]     │
  │                                                    ├── 被唤醒！
  │                                                    │ ibv_ack_cq_events()
  │                                                    │ ibv_req_notify_cq()
  │                                                    │ ibv_poll_cq() → 取出 CQE
  │                                                    │ on_completion():
  │                                                    │   "received message: ..."
  │                                                    │
  ├── 收到 DISCONNECTED                                │
  │     on_disconnect()                                │
  │       销毁 QP、注销 MR、释放内存                     │
  │                                                    │
  └── 事件循环继续等待（或退出）
```

---

## 核心知识速查表

| 概念 | 要点 |
|------|------|
| **PD** | `ibv_alloc_pd()` — 所有资源（QP/CQ/MR）必须在同一个 PD 下 |
| **CQ** | `ibv_create_cq()` — 操作完成的通知队列，通过 `ibv_poll_cq()` 获取 |
| **MR** | `ibv_reg_mr()` — 注册内存后获得 lkey（本地用）和 rkey（远端用） |
| **QP** | `rdma_create_qp()` — 类型选 RC（可靠连接）最常用 |
| **Post Recv** | `ibv_post_recv()` — **必须在对端 send 之前调用** |
| **Post Send** | `ibv_post_send()` — opcode 决定是 SEND/WRITE/READ |
| **Signal** | `IBV_SEND_SIGNALED` — 产生 CQE；unsignal 不产生，但需定期 signal 防止队列满 |
| **Inline** | `IBV_SEND_INLINE` — 小数据内联在请求中，减少一次网卡取数据的过程 |

## 十一、Client 完整流程逐函数解析

### 11.1 执行流程总览

```
main()
  │
  ├─ 1. 解析地址 + 发起地址解析
  │
  └─ 2. 事件循环 ──→ on_event() 分发事件
                        │
                        ├─ ADDR_RESOLVED ──→ on_addr_resolved()
                        │                      ├─ build_context()
                        │                      ├─ build_qp_attr()
                        │                      ├─ rdma_create_qp()
                        │                      ├─ register_memory()
                        │                      └─ rdma_resolve_route()
                        │
                        ├─ ROUTE_RESOLVED ──→ on_route_resolved()
                        │                      └─ rdma_connect()
                        │
                        ├─ ESTABLISHED ──→ on_connection()
                        │                    └─ ibv_post_send()
                        │
                        └─ DISCONNECTED ──→ on_disconnect() ← 清理，退出循环

  [另一个线程] poll_cq() ──→ on_completion()
                               ├─ IBV_WC_SEND → "send completed"
                               └─ 收到 1 个完成 → rdma_disconnect()
```

### 11.2 Client 与 Server 的结构体差异

Client 的 `struct connection` 比 Server 多两个字段：

- `rdma_cm_id *id` — Client 需要在 `on_completion()` 中主动调用 `rdma_disconnect(conn->id)` 断开连接
- `int num_completions` — 计数完成事件，达到 1 次就主动断开

### 11.3 阶段一：启动——`main()`

```c
// 1. 解析 server 地址
TEST_NZ(getaddrinfo(argv[1], argv[2], NULL, &addr));

// 2. 创建事件通道和连接标识（同 Server）
TEST_Z(ec = rdma_create_event_channel());
TEST_NZ(rdma_create_id(ec, &conn, NULL, RDMA_PS_TCP));

// 3. 发起地址解析（异步，完成后触发 ADDR_RESOLVED 事件）
TEST_NZ(rdma_resolve_addr(conn, NULL, addr->ai_addr, TIMEOUT_IN_MS));

// 4. 进入事件循环（同 Server）
while (rdma_get_cm_event(ec, &event) == 0) { ... }
```

Client vs Server 启动方式：
```
Server:  bind() → listen() → 等待连接
Client:  resolve_addr() → [ADDR_RESOLVED] → resolve_route() → [ROUTE_RESOLVED] → connect()
```

### 11.4 阶段二：事件分发——`on_event()`

Client 处理 4 种事件（比 Server 多 2 种）：

| 事件 | Server | Client | 含义 |
|------|:------:|:------:|------|
| `CONNECT_REQUEST` | 有 | 无 | 有人来连接（只有被动端） |
| `ADDR_RESOLVED` | 无 | **有** | 地址解析完成 |
| `ROUTE_RESOLVED` | 无 | **有** | 路由解析完成 |
| `ESTABLISHED` | 有(注释) | **有(活跃)** | 连接建立 |
| `DISCONNECTED` | 有 | 有 | 连接断开 |

### 11.5 阶段三：地址解析完成——`on_addr_resolved()`

```c
int on_addr_resolved(struct rdma_cm_id *id)
{
  build_context(id->verbs);        // 初始化全局资源（PD/CQ/线程）
  build_qp_attr(&qp_attr);        // 配置 QP 参数
  rdma_create_qp(id, ...);        // 创建 QP

  conn->id = id;                   // 保存 id（Server 不需要）
  conn->num_completions = 0;       // 初始化计数器

  register_memory(conn);           // 注册 send/recv 缓冲区
  //post_receives(conn);           // 被注释！Client 只发不收

  rdma_resolve_route(id, TIMEOUT); // 发起路由解析（异步）
}
```

- `rdma_resolve_addr` 的作用：IP 地址 → RDMA 设备的 GID（类似 ARP 解析 IP → MAC）
- `rdma_resolve_route` 的作用：确定从本机到 Server 的网络路径
- `post_receives` 被注释：当前示例 Client 只发不收

### 11.6 阶段四：路由解析完成——`on_route_resolved()`

```c
int on_route_resolved(struct rdma_cm_id *id)
{
  memset(&cm_params, 0, sizeof(cm_params));
  rdma_connect(id, &cm_params);   // 正式发起连接
}
```

`rdma_connect()` 在背后：
1. 把本端 QP 信息打包发给 Server
2. 等待 Server `rdma_accept()`
3. 收到 Server 的 QP 信息后自动完成 RESET → INIT → RTR → RTS
4. 触发 `ESTABLISHED` 事件

### 11.7 阶段五：连接建立——`on_connection()`

```c
int on_connection(void *context)
{
  snprintf(conn->send_region, BUFFER_SIZE,
    "message from active/client side with pid %d", getpid());

  wr.wr_id = (uintptr_t)conn;
  wr.opcode = IBV_WR_SEND;             // SEND 操作
  wr.send_flags = IBV_SEND_SIGNALED;   // 完成后产生 CQE

  sge.addr = (uintptr_t)conn->send_region;
  sge.length = BUFFER_SIZE;
  sge.lkey = conn->send_mr->lkey;

  ibv_post_send(conn->qp, &wr, &bad_wr);  // 提交到发送队列
}
```

连接一建立就立即发送消息。Server 的同名函数被注释掉了。

### 11.8 阶段六：完成处理——`on_completion()`（关键区别）

```c
void on_completion(struct ibv_wc *wc)
{
  if (wc->status != IBV_WC_SUCCESS)
    die("...");

  if (wc->opcode & IBV_WC_RECV)
    printf("received message: %s\n", conn->recv_region);
  else if (wc->opcode == IBV_WC_SEND)
    printf("send completed successfully.\n");

  if (++conn->num_completions == 1)   // ← 关键！
    rdma_disconnect(conn->id);         // 收到 1 个完成就主动断开
}
```

Client 在第 1 个完成事件（send 完成）后就主动 disconnect。Server 没有这个逻辑，是被动等待。

### 11.9 阶段七：断开——`on_disconnect()`

和 Server 几乎相同，但 **返回 1**（退出事件循环，程序结束）。Server 返回 0（继续等待下一个连接）。

### 11.10 Client vs Server 全面对比

| 维度 | Client（主动端） | Server（被动端） |
|------|-----------------|-----------------|
| **启动** | `resolve_addr` → 异步 | `bind` + `listen` → 等待 |
| **事件数** | 4 种 | 3 种 |
| **资源初始化时机** | `ADDR_RESOLVED` 事件中 | `CONNECT_REQUEST` 事件中 |
| **连接动作** | `rdma_connect()` | `rdma_accept()` |
| **post_receives** | 注释掉（只发不收） | 活跃（需要收消息） |
| **on_connection** | 活跃（连接后立即 send） | 注释掉（只收不发） |
| **谁主动断开** | Client（1 个完成后 disconnect） | 被动（等 Client 断开） |
| **disconnect 返回值** | 1（退出程序） | 0（继续等待） |

### 11.11 完整时间线

```
Client main()                          后台 poll_cq 线程         Server 端
─────────────                          ─────────────────         ──────────
rdma_create_event_channel()
rdma_create_id()
rdma_resolve_addr()                                              (listening...)
  │
  ├── ADDR_RESOLVED
  │     build_context()                poll_cq() 启动
  │     rdma_create_qp()                │ [阻塞等待中]
  │     register_memory()               │
  │     rdma_resolve_route()            │
  │                                     │
  ├── ROUTE_RESOLVED                    │
  │     rdma_connect() ───────────────────────→ Server: CONNECT_REQUEST
  │       (QP: RESET→INIT→RTR→RTS)     │        Server: rdma_accept()
  │                                     │
  ├── ESTABLISHED                       │
  │     on_connection():                │
  │       ibv_post_send(SEND)           │
  │         │                           │
  │         └── 网卡直接传输 ───────────────→ Server recv_region
  │                                     │
  │                                     ├── 被唤醒 (send CQE)
  │                                     │ "send completed"
  │                                     │ num_completions=1 → disconnect
  │                                     │
  ├── DISCONNECTED                      │
  │     on_disconnect()                 │
  │       清理资源，return 1            │
  │                                     │
  └── 程序结束
```

---

## 推荐学习路径

1. **先精读 `rdma_send_recv/`** — 理解最基础的双边操作
2. **再读 `rdma_write_read/`** — 理解单边操作以及 MR 信息交换
3. **读 `tutorial.md`** — 项目自带的中文教程，有参数性能对比实验
4. **读 `interview/` 目录** — 面试常见问题，检验理解
5. **尝试修改代码** — 比如让 client/server 互发多条消息、尝试 unsignal 模式等
