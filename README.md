# XJTU-netlab-ChatRoom

西安交通大学计算机网络最终实验-聊天室

## 功能要求

- [x] 使用用户名和密码验证用户登陆
- [x] 允许用户注册, 返回一个10位数字的账号
- [x] 用户之间可以用文字聊天
- [x] 离线传输文件
- [x] 双方在线时用NAT传输文件
- [x] 语音聊天

## 实现策略

1. 服务器端使用数据库维护用户信息
2. 服务器端暂存用户发送的消息, 用户上线时通知服务器, 并接受消息. 服务器传达消息后删除暂存的消息
3. 在传输文件之前, 先通告文件的大小, 文件名和哈希值, 服务端检查文件名是否存在, 哈希值是否相等, 如果存在且哈希值不相等则启动断点续传, 从服务端接受到的文件大小处开始接受文件
4. 当收发双方都在线时, 则启动NAT协议, 服务器端告知双方IP地址和端口, 双方直接连接
5. 语音聊天仅当双方都在线时有效, 双方维护两条UDP连接, 分别发送自己的语音流和接收对方的语音流

## 实现细节

### 用户登陆

客户端在本地查询要求输入用户名和密码

客户端向服务器端申请连接, 然后发送登录报文

```json
{
    "type": "login",
    "username": "<username>",
    "password": "<password>"
}
```

服务端收到报文后, 维持和客户端的连接, 并在数据库中查询用户信息, 如果用户信息存在且正确, 则维持连接并发送登录成功的报文; 否则发送登录失败的报文, 并断开连接

接着服务端维护用户名到用户地址的映射

服务端检查消息队列, 如果队列中有目的为这个用户的消息, 则发送之

### 用户注册

客户端在本地获得用户信息后向服务器端申请连接, 然后发送注册报文

```json
{
    "type": "signup",
    "username": "<username>",
    "password": "<password>"
}
```

服务端收到报文后, 维持和客户端的连接, 并在数据库中查询用户信息, 如果用户信息不存在, 则将用户信息写入数据库, 并返回注册成功的报文; 如果用户名已存在, 则返回要求改名的报文; 否则返回注册失败的报文, 并断开连接

### 文字聊天

客户端首先向服务器发送报文

```json
{
    "type": "chat",
    "from": "<selfname>",
    "to": "<username>",
    "content": "<content>"
}
```

服务端收到消息后, 首先检查收信人是否在线, 若在线则直接转送, 若不在线则暂存于消息队列中. 每有一个用户登录, 服务器都检查消息队列, 并将消息转送之

### 文件传输

客户端首先向服务器发送文件传输请求报文

```json
{
    "type": "ftp_request",
    "from": "<selfname>",
    "to": "<username>",
    "filename": "<filename>",
    "filesize": <filesize>,
    "hash": "<hash>"
}
```

服务端收到请求报文后, 首先检查收信人是否在线

如果收信人在线, 则服务端向收信人转发该ftp请求报文

收信人收到连接开启报文后, 检查本地文件的哈希是否相等, 若不相等则在本地开辟一个新的TCP服务端并返回IP地址和端口

```json
{
    "type": "ftp_relpy",
    "ip": "<ip>",
    "port": <port>,
    "offset": <offset>
}
```

服务端受到收信人信息后将其转送给发信人, 发信人依照IP地址和端口建立连接, 并发送文件传输请求报文

若收信人不在线, 则服务端开始接收文件. 收信人接收文件的行为和服务端类似

发信人收到连接开启报文后, 在本地开辟一个新的TCP连接, 并循环发送固定大小的文件片段, 直到文件发送完成

### 断点续传

发信人接收到键盘事件`Crtl+C`后中止发送, 收信人保存本地未接受完的文件

### 语音聊天

和文件传输类似, 客户端首先在本地开辟一个UDP服务端, 并发送连接开启报文

```json
{
    "type": "voice",
    "from": "<selfname>",
    "to": "<username>",
    "ip": "<ip>",
    "port": <port>
}
```

服务端受到信息后转送给收信人, 收信人收到连接开启报文后, 同样开辟一个UDP连接, 并发送连接开启报文

```json
{
    "type": "voice",
    "from": "<selfname>",
    "to": "<username>",
    "ip": "<ip>",
    "port": <port>
}
```
