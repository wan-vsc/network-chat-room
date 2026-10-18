# 网络聊天室（Windows 版）

一个从 **Linux epoll 聊天室** 移植到 **Windows Winsock** 的 TCP 群聊小程序。
原始版本依赖 Linux 的 `epoll` 多路复用，Windows 没有 epoll，因此改用 `select()`
承担同样的「一次等待、多个 socket 就绪通知」职责，应用层协议与交互完全一致。

- 语言：C++（需要 C++11，实际用到 `std::thread` / `std::atomic`）
- 平台：Windows（Winsock 2.2），x64
- 依赖：仅标准库 + `ws2_32.lib`

## 文件说明

| 文件 | 作用 |
| --- | --- |
| `utility_win.h` | 公共头：宏定义、全局客户端链表、收发辅助函数、单条消息处理 `handleClientMessage()` |
| `epoll_server_win.cpp` | 服务端：建监听 → `select()` 主循环 → 接收连接 / 广播消息 |
| `epoll_client_win.cpp` | 客户端：连接 → 键盘线程发消息 + 主线程 `select()` 收消息 |

## 移植映射表

| Linux 原始实现 | Windows 版本 | 说明 |
| --- | --- | --- |
| `epoll_create / epoll_ctl / epoll_wait` | `select()` + `fd_set` | 职责等价：阻塞等待一批 fd 就绪 |
| `close(fd)` | `closesocket(fd)` | Winsock 专用关闭函数 |
| `fork()` + `pipe()` 读键盘 | `std::thread` 读键盘 | Windows 无 `fork`，用线程更自然 |
| `strncasecmp` | `_strnicmp` | 大小写不敏感比较 |
| `errno` | `WSAGetLastError()` | Winsock 的错误查询方式 |
| — | `WSAStartup / WSACleanup` | 使用 Winsock 前必须初始化、退出前必须清理 |

## 构建与运行

在「x64 Native Tools Command Prompt for VS」中执行（或先运行 `vcvars64.bat`）：

```bat
cl /EHsc /W4 /Fe:chat_server.exe epoll_server_win.cpp
cl /EHsc /W4 /Fe:chat_client.exe epoll_client_win.cpp
```

然后开两个（或更多）终端：

```bat
chat_server.exe
chat_client.exe
```

客户端连上后直接输入文字回车即发送，输入 `exit`（不区分大小写）退出；
服务端按 `Ctrl+C` 干净退出并释放所有 socket。

## 功能与规则

- 新客户端连入后收到欢迎语，并按 socket 句柄获得一个 `Client #<句柄>` 编号
- 任意客户端发言，服务端把 `ClientID <句柄> say >> <内容>` 广播给**其他**所有人
- 房间里只有一个人时，只会收到一句提示，不会回声
- 单条消息上限 64 KB（`BUF_SIZE 0xFFFF`），沿用原始设计

## 已知限制（移植时有意保留的原始设计）

- **单线程 + `select()`**：`FD_SETSIZE` 被抬到 512，同时在线上限约 500，远低于原 epoll 的 5000
- **未处理 TCP 粘包 / 拆包**：消息以 `\0` 结尾、一次一发，本机回环下通常正常；严格做法应加长度前缀
- `clients_list` 定义在头文件中：本项目每个可执行文件只编译一个 `.cpp`，因此安全；若拆成多文件需改为 `extern` 声明
- 没有心跳与超时，客户端异常断开只能等 `recv` 返回 0 才发现
