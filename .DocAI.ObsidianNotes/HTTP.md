**HTTP = 客户端和服务端之间约定好的"对话格式"**，约定了请求长什么样、响应长什么样。

举个例子，OpenFeign 发起的 `GET /api/files/123`，实际上在网络里传输的是这个文本：

```
GET /api/files/123 HTTP/1.1
Host: file-service:8082
Authorization: Bearer xxxxx
X-User-Id: 1
                            ← 空行
                            ← 请求体（GET没有）
```

文件服务解析这个文本后返回：

```
HTTP/1.1 200 OK
Content-Type: application/json
                            ← 空行
{"id": 123, "name": "报告.pdf", "url": "..."}
```

**具体调用链路：**

```
你的代码                                        文件服务
                                                
fileClient.getFile(123)                         
  ↓                                             
OpenFeign（生成HTTP请求）                        
  ↓                                             
Nacos（服务发现：file-service → 192.168.1.102:8082）
  ↓                                             
负载均衡（有多个实例时选一个）                     
  ↓                                             
HTTP请求 →→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→  Controller
  ↑                                             ↓
  ←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←  JSON响应
  ↓
OpenFeign 把 JSON 反序列化成 Java 对象
  ↓
返回 File 对象给你
```

所以 HTTP 协议本质就是**文本格式的约定**——请求行（方法 + 路径 + 版本）+ 请求头 + 空行 + 请求体。OpenFeign 帮你拼这个文本、发出去、解析回来的文本变成 Java 对象。