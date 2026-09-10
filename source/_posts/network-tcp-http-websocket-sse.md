---
title: TCP、HTTP、WebSocket 与 SSE：一根管道上的四种用法
date: 2026-09-10 10:46:00
description: 四个协议分处两层：TCP 是传输层的可靠字节流，本身没有消息边界；HTTP、SSE、WebSocket 是其上的应用层协议，分别用报文规则、事件流、帧协议在字节流上切出消息。本文全部配可运行的 Node.js 代码与真实输出：从 seq/ack 的语义讲透三次握手与四次挥手（含半关闭实验），再到 HTTP 报文解析器、SSE 事件流、WebSocket 帧逐字节分析。
categories:
  - [前端基础, 网络]
tags:
  - 网络
  - TCP
  - HTTP
  - WebSocket
  - SSE
---

这四个名字经常一起出现，但它们并不在同一层面上竞争：

> **TCP 是传输层的「管道」**：保证字节可靠、有序地到达，但对内容的格式一无所知。**HTTP、SSE、WebSocket 都是应用层协议**，跑在同一根 TCP 管道上——区别只在于怎么用这根管道：HTTP 用报文规则切出「一问一答」，SSE 让一次响应永远不结束从而单向推送，WebSocket 握手后换成自己的帧协议实现全双工。

```text
应用层    HTTP(一问一答)   SSE(单向事件流)   WebSocket(全双工帧)
              │               │                  │
传输层        └───────────── TCP(可靠字节流) ─────────────┘
网络层                          IP(尽力送达)
```

<!-- more -->

## 一、TCP：一根没有消息边界的字节流

IP 层只管「尽力送达」，丢包、乱序、重复都不负责。TCP 在其上补齐三件事：

- **连接**：握手同步双方的起始序号，挥手协商关闭；
- **可靠**：每个字节都有编号，接收方逐段确认（ACK），超时未确认就重传；
- **不撑死对方**：滑动窗口做流量控制（别超过接收方缓冲区），拥塞窗口做拥塞控制（别超过网络承载力）。

### 1.1 先搞懂 seq 和 ack，握手才看得懂

TCP 报文头里最容易「背了就忘」的就是这两个数字，其实语义很简单：

- **seq（序号）**：本报文段携带的数据，从字节流的**第几个字节**开始编号；
- **ack（确认号）**：你发来的数据我已经按序收到了 `ack - 1` 字节，**期望你下一个从 `ack` 开始发**。

数据传输阶段两者如何走，一个例子就够：

```text
客户端                                服务端
  │ ── seq=1001, 数据 "Hello" ──► │    发 5 个字节
  │ ◄─ ack=1006 ──────────────── │    1001+5，"前 1005 个都收到了，下次从 1006 发"
```

「确认 = 收到的最后字节序号 + 1」这条规则，在握手和挥手里会反复出现。

### 1.2 三次握手：交换的不是数据，是初始序号

建立连接前，双方对彼此的编号体系一无所知。握手要做的事：**交换初始序号（ISN），并让双方都确认「自己和对方的收发能力都正常」**。带具体数字走一遍：

```text
客户端                                       服务端（LISTEN）
  │                                            │
  │ ──① SYN, seq=1000 ──────────────────────► │   "请求连接，我的数据从 1000 开始编号"
  │   SYN_SENT                                 │   SYN_RCVD
  │                                            │
  │ ◄─② SYN+ACK, seq=5000, ack=1001 ────────── │   "同意；我的数据从 5000 开始，
  │   ESTABLISHED                              │      你的 1000 我收到了(占一个序号)"
  │                                            │
  │ ──③ ACK, ack=5001 ──────────────────────► │   ESTABLISHED
```

每一步之后，「谁确认了什么」：

| 报文 | 客户端已确认 | 服务端已确认 |
| --- | --- | --- |
| ① SYN | —（还什么都不知道） | 客户端能发、自己能收 |
| ② SYN+ACK | 自己能发能收、服务端能发能收 | （同①） |
| ③ ACK | （同②） | 自己能发、客户端能收 → 双方齐全 |

**为什么不是两次**：走到②就停的话，服务端已经单方面建立连接，但它发出的 `SYN+ACK` 有没有被客户端收到，永远没有验证——包丢了它就只能傻等。另一个经典场景：网络里滞留的**旧 SYN**（历史连接的残包）迟到抵达服务端，两次握手下服务端会直接建连白占资源；三次握手下客户端发现 `ack` 跟自己的序号对不上，回 RST 拒绝。

**为什么不是四次**：② 里服务端的「确认对方」(ACK) 和「介绍自己」(SYN) 没有理由分开发，合并成一个包；三次已经让双方的确认闭环，多一次纯浪费。

**为什么握手包也要占序号**：SYN 和 FIN 虽然不携带数据，但都是「改变字节流状态」的关键报文，各占一个序号，所以②里的 ack 是对方 seq + 1。丢包重传和数据包共用同一套 seq/ack 体系——比如③丢了，服务端会重传 `SYN+ACK`；若客户端已经直接开始发数据，数据报文自带的 `ack=5001` 同样能让服务端进入 ESTABLISHED。

### 1.3 四次挥手：FIN 只关一个方向

TCP 是**全双工**的——两个方向各自独立收发，所以关闭也要**两边各关各的**。FIN 的语义不是「断开连接」，而是「**我的数据发完了**」：只关写方向，读方向还开着（半关闭）。用代码验证这一点：

```js
const net = require('net');
const log = (who, msg) => console.log(`[${who}] ${msg}`);

// allowHalfOpen: true —— Node 默认收到 FIN 会自动回 FIN，
// 关掉这个默认行为，才能看到协议本来的半关闭
const server = net.createServer({ allowHalfOpen: true }, (socket) => {
  socket.on('data', () => {});            // 消费数据，'end' 才会触发
  socket.write('服务端：连接刚建立时发的数据\n');
  socket.on('end', () => {                // 收到客户端 FIN：它不再发了
    log('server', '收到 FIN，但我这边还有话没说完');
    socket.write('收到你的FIN，我这边还有话没说完\n');   // 半关闭下照常发送
    setTimeout(() => socket.end('说完了，我也关\n'), 200); // 这才是我的 FIN
  });
});

server.listen(3000, () => {
  const client = net.connect(3000, () => {
    client.end('客户端：我的数据发完了，先关写端\n');     // 数据 + FIN 一起发出
  });
  client.on('data', (c) => log('client', '仍能收到: ' + c.toString().trim()));
});
```

运行输出（真实运行结果）：

```text
[client] 已连接，发完数据就关写端
[client] 仍能收到: 服务端：连接刚建立时发的数据
[server] 收到 FIN，但我这边还有话没说完
[client] 仍能收到: 收到你的FIN，我这边还有话没说完
[client] 仍能收到: 说完了，我也关
[client] 连接完全关闭
```

客户端发出 FIN 之后，**还在持续收到服务端的数据**——这就是「半关闭」，也是四次挥手的全部秘密。对着状态机走一遍（序号接上文数据阶段的 1006 / 5003）：

```text
客户端                                       服务端
  │ ──① FIN, seq=1006 ─────────────────────► │   "我的数据发完了"
  │   FIN_WAIT_1                              │   CLOSE_WAIT（注意：还能发数据！）
  │ ◄─② ACK, ack=1007 ────────────────────── │   "知道了"
  │   FIN_WAIT_2（半关闭：只收不发）           │
  │            ……服务端把剩余数据发完……        │
  │ ◄─③ FIN, seq=5003 ─────────────────────── │   "我也发完了"
  │   TIME_WAIT                               │   LAST_ACK
  │ ──④ ACK, ack=5004 ──────────────────────► │
  │   等 2MSL 后 → CLOSED                      │   收到 ACK → CLOSED
```

**为什么是四次而不是三次**：收到对方的 FIN，ACK 必须**立刻**回（这是 TCP 的确认义务）；但自己的 FIN 意味着「我发完了」，要**等应用层把剩余数据写完**才能发。②和③中间隔着「剩余数据」的发送窗口，没法像握手的 SYN+ACK 那样合并成一个包。上面的运行输出里，②和③之间隔了服务端的两条消息——间隔里发生的事，正是多出那一次挥手的原因。

**为什么主动方要等 2MSL（TIME_WAIT）**：

1. 若④这个 ACK 丢了，服务端会重传 FIN；客户端若已经 CLOSED 就无法应答，服务端将永远停在 LAST_ACK。等 2MSL = ACK 去程最多 1 个 MSL + 重传 FIN 回程最多再 1 个 MSL，兜住这次重传；
2. 让本连接的旧报文在网络上自然消亡，防止污染之后复用同一四元组（同 IP 同端口）的新连接。

### 1.4 字节流无边界：应用层协议的起点

TCP 对上层只交付「一串连续字节」，**写入几次和读取几次没有对应关系**：

```js
const net = require('net');
// 服务端：连着写两条"消息"
net.createServer((socket) => {
  socket.write('hello');
  socket.write('world');
  socket.end();
}).listen(3001);

// 客户端
const client = net.connect(3001);
client.on('data', (c) => console.log(`收到 "${c}"`));
```

运行输出——两次 `write` 合并成了一次 `data`（也可能反过来被拆开，取决于网络）：

```text
第 1 次 data 事件, 长度 10: "helloworld"
```

这不是 bug，是字节流的本性（即所谓粘包/拆包）。于是得到贯穿全文的主线：**每个应用层协议的第一件事，都是在无边界的字节流上定义「消息边界」**。接下来三个协议的报文设计，本质是在回答同一个问题：一段字节，从哪开始、到哪结束。

## 二、HTTP：用报文规则切出「一问一答」

HTTP 的边界方案浓缩在一条请求的解析规则里：**头部以 `\r\n\r\n` 结束，正文长度由 `Content-Length` 声明**。在裸 TCP 上手写一个解析器，规则立刻具象化：

```js
const net = require('net');

net.createServer((socket) => {
  let buf = Buffer.alloc(0);                       // 字节流先攒起来
  socket.on('data', (chunk) => {
    buf = Buffer.concat([buf, chunk]);
    const idx = buf.indexOf('\r\n\r\n');           // 头部结束标记
    if (idx === -1) return;                        // 头没到齐：继续等
    const head = buf.subarray(0, idx).toString();
    const len = Number(/Content-Length: (\d+)/i.exec(head)?.[1] ?? 0);
    if (buf.length - idx - 4 < len) return;        // body 没到齐：继续等
    const body = buf.subarray(idx + 4, idx + 4 + len).toString();
    console.log('--- 解析出完整请求 ---\n' + head + '\n[body] ' + body);
    socket.end('HTTP/1.1 200 OK\r\nContent-Length: 2\r\n\r\nok');
  });
}).listen(3002);

// 客户端：故意分两次写，模拟 TCP 分段到达
const client = net.connect(3002, () => {
  const body = '{"user":"a","pass":"b"}';
  const head = `POST /api/login HTTP/1.1\r\nHost: localhost\r\nContent-Length: ${Buffer.byteLength(body)}\r\n\r\n`;
  client.write(head);
  setTimeout(() => client.write(body), 100);       // 100ms 后 body 才到
});
```

运行输出——头和 body 分两次到达，解析器靠缓冲 + 边界规则拼出完整请求：

```text
--- 解析出完整请求 ---
POST /api/login HTTP/1.1
Host: localhost
Content-Length: 23
[body] {"user":"a","pass":"b"}
```

调试这段代码时踩过一个坑，恰好证明这套规则的分量：`Content-Length` 手写成了 26 而实际 body 是 23 字节，解析器**永远等不齐第 24~26 字节**，连接一挂到底。边界声明差一个字节，整条连接就死掉。

在此之上是 HTTP 的模型与演进：

- **一问一答 + 无状态**：请求永远由客户端发起，协议不记得上一个请求是谁，会话靠 Cookie / Token 补；
- **keep-alive**（HTTP/1.1 默认）：一次对话结束不拆 TCP 连接，下一个请求直接复用。能复用的前提正是边界明确——解析器必须准确知道上一个响应在哪里结束；
- **HTTP/1.1 队头阻塞**：一条连接上对话必须排队。HTTP/2 的解法是把报文拆成**二进制帧**、标记流 ID 交错传输（多路复用）——注意这步和 WebSocket 殊途同归，都转向「帧 + 长度字段」的边界方案；TCP 层的队头阻塞要等 HTTP/3 换 QUIC 才解决。

这个模型的能力边界也由此清楚：**服务端无法主动开口**——这是下面两个协议存在的原因。

## 三、SSE：让一次响应永远不结束

SSE（Server-Sent Events）的思路简单：**正常发起 GET 请求，服务端不写 Content-Length，让响应一直「没结束」，有新消息就往里写一段**。Node.js 十几行实现一个行情推送：

```js
const http = require('http');

http.createServer((req, res) => {
  // 断线重连时浏览器自动带上最后收到的事件 id
  let id = Number(req.headers['last-event-id'] || 0);

  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
  });

  const timer = setInterval(() => {
    const price = (100 + Math.random() * 10).toFixed(2);
    res.write(`id: ${++id}\nevent: price\ndata: ${price}\n\n`);
  }, 1000);

  req.on('close', () => clearInterval(timer));   // 客户端断开必须清理
}).listen(3000);
```

线上的数据流（`Transfer-Encoding: chunked` 分块，永不主动结束）：

```text
HTTP/1.1 200 OK
Content-Type: text/event-stream

id: 1
event: price
data: 104.37
                                  ← 空行 = 一个事件结束，这就是 SSE 的消息边界
id: 2
event: price
data: 107.91

```

格式规则：每行是 `字段: 值`，支持 `id` / `event` / `data` / `retry`（重连等待毫秒数）；**事件之间用空行分隔**。客户端是原生 API：

```js
const es = new EventSource('http://localhost:3000');
es.addEventListener('price', (e) => console.log('最新价格', e.data));
```

重连机制值得一读：连接断开后 `EventSource` 自动重试，并把最后收到的 `id` 放进 `Last-Event-ID` 请求头——服务端读这个头就能续传（上面服务端代码第一行）。断点续传是协议内建的。能力边界：**单向**（客户端要说话得另发普通请求）、**纯文本**（二进制要 Base64，体积膨胀 1/3）、HTTP/1.1 下浏览器对同域只开 6 条 TCP 连接（HTTP/2 多路复用后无此顾虑）。

## 四、WebSocket：握手换协议，边界靠帧

要双向实时（聊天、协同编辑、联机游戏），「SSE + 普通请求」太别扭。WebSocket 的做法更彻底：**先用 HTTP 发起一次「换协议」谈判，成功后这条 TCP 连接不再说 HTTP**。

### 4.1 握手：一次特殊的 HTTP 请求

```text
GET /chat HTTP/1.1
Upgrade: websocket                          ← 请求升级协议
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ== ← 随机 16 字节的 Base64

HTTP/1.1 101 Switching Protocols            ← 101：协议切换成功
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

`Sec-WebSocket-Accept` 是一次防篡改校验，算法固定，用 Node 起手就能复现：

```js
const crypto = require('crypto');
const GUID = '258EAFA5-E914-47DA-95CA-C5AB0DC85B11';   // 协议规定的固定串
const accept = crypto
  .createHash('sha1')
  .update('dGhlIHNhbXBsZSBub25jZQ==' + GUID)          // key + GUID 做 SHA-1
  .digest('base64');
// → s3pPLMBiTxaQ9kYGzzhZRbK+xOo=，与响应头一致
// 双方各自计算比对，确保中间的代理没有篡改握手
```

### 4.2 帧：逐字节分析边界方案

握手之后，字节流上跑的换成 WebSocket 帧。先看真实字节——服务端发一个 `"Hello"`，线上就是这 7 个：

```text
81 05 48 65 6c 6c 6f
│  │  └────────┬────────┘
│  │        "Hello" 的 ASCII
│  └ Payload len = 5（无掩码）
└ FIN=1 且 opcode=0x1（文本帧，单帧即完整消息）
```

帧头各字段的职责：

| 字段 | 作用 |
| --- | --- |
| FIN | 1 表示消息的最后一帧（大消息可分片） |
| opcode | `0x1` 文本、`0x2` 二进制、`0x0` 延续帧、`0x8` 关闭、`0x9` ping、`0xA` pong |
| MASK | 是否掩码。**客户端发的帧必须掩码，服务端发的不掩码**（防中间代理被恶意客户端欺骗解析） |
| Payload len | 7 位存长度；不够用时置 126 / 127，再跟 2 或 8 字节扩展长度 |

按这套规则手写解码器，粘包在此天然解决——读完帧头知道长度，收满即一条完整消息：

```js
function decodeFrame(buf) {
  const fin = (buf[0] & 0x80) !== 0;       // 最高位
  const opcode = buf[0] & 0x0f;            // 低 4 位
  const masked = (buf[1] & 0x80) !== 0;
  let len = buf[1] & 0x7f;
  let offset = 2;
  if (len === 126) { len = buf.readUInt16BE(offset); offset += 2; }
  else if (len === 127) { len = Number(buf.readBigUInt64BE(offset)); offset += 8; }
  let mask = null;
  if (masked) { mask = buf.subarray(offset, offset + 4); offset += 4; }
  const payload = buf.subarray(offset, offset + len);
  if (mask) for (let i = 0; i < payload.length; i++) payload[i] ^= mask[i % 4];  // 异或解掩码
  return { fin, opcode, masked, data: payload.toString() };
}

// 验证：服务端帧 "Hello"（无掩码）
console.log(decodeFrame(Buffer.from([0x81, 0x05, ...Buffer.from('Hello')])));
```

运行输出（含一个真实构造的掩码客户端帧 `81 82 37fa213d 7f93`，第二字节 `82` 的最高位就是 MASK=1）：

```text
服务端帧 : 810548656c6c6f
解码结果 : { fin: true, opcode: 1, masked: false, data: 'Hello' }
客户端帧 : 818237fa213d7f93
解码结果 : { fin: true, opcode: 1, masked: true, data: 'Hi' }
```

心跳也是帧：一方发 opcode=0x9 的 ping，另一方必须回 0xA 的 pong，超时无响应即可判定连接已死。实际开发用 `ws` 库，不必手撕：

```js
const { WebSocketServer } = require('ws');          // 服务端
const wss = new WebSocketServer({ port: 3000 });
wss.on('connection', (ws) => {
  ws.on('message', (data) => console.log('收到', data.toString()));
  ws.send('welcome');
});
```

```js
const ws = new WebSocket('ws://localhost:3000');    // 客户端（浏览器原生）
ws.onmessage = (e) => console.log(e.data);
ws.send('hello');
```

代价是**脱离 HTTP 生态**：没有状态码、缓存、Content-Type 语义，鉴权要塞进 URL 或首条消息，断线重连、心跳都要自己实现。

## 五、怎么选

| | HTTP | SSE | WebSocket |
| --- | --- | --- | --- |
| 通信方向 | 一问一答 | 服务端 → 客户端单向流 | 全双工 |
| 底层依赖 | TCP | 普通 HTTP 长响应（chunked） | TCP（HTTP 握手后换帧协议） |
| 消息边界 | `Content-Length` / chunked | 事件间空行 | 帧头 Payload len |
| 数据格式 | 任意 | 纯文本 | 文本 + 二进制 |
| 断线重连 | — | 内建（Last-Event-ID 续传） | 自己写 |
| 典型场景 | 常规 API、静态资源 | 通知、行情、构建进度、AI 流式输出 | 聊天、协同编辑、联机游戏 |

**普通的用 HTTP；只需要服务端推、客户端无需实时回话的用 SSE；双端都要高频互发的才上 WebSocket。** 能用 SSE 就别上 WebSocket——它就是个不结束的 HTTP 响应，代理、CDN、鉴权全部白嫖 HTTP 生态。

## 六、总结

一条线索贯穿四个协议——**在 TCP 无边界的字节流上，各自如何定义消息**：

- TCP 用 seq/ack + 重传交付有序字节，握手交换初始序号，挥手按方向各自关闭（FIN 只关写端，所以是四次）；
- HTTP 用 `\r\n\r\n` 切头部、`Content-Length` / chunked 切正文，换得一问一答的确定性；
- SSE 借 chunked 让响应不结束，用空行切事件，把「响应」变成「事件流」；
- WebSocket 用 Upgrade 握手接管连接，用帧头长度字段切消息，把「对话」变成「双向管道」。

方向性上是三个台阶：一问一答 → 单向推送 → 全双工。理解了「边界」和「方向」这两个维度，四个协议就不再是名词，而是同一根管道上的四种用法。
