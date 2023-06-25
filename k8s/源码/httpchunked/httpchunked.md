# httpchunked

## 概述
在一次http请求/响应中，客户端/服务端可以在请求头/响应头中添加`Transfer-Encoding: chunked`来表明此次请求体/响应体是一次分段响应，服务端/客户端将会保持连接直到接收完整个请求/响应

## Chunked体
chunked类型的请求/响应体由一个个chunk组成，chunk的格式为：  
`[Chunk size][Chunk data][Chunk boundnary]`
1. Chunk size：本次Chunk的大小，大小由十六进制数字的字符串形式+`\r\n`表示
2. Chunk data：本次Chunk的内容
3. Chunk boundnary：本次Chunk的结束符，为`\r\n`


## Chunked结束标记
chunked结束标记由一个chunk大小为0的chunk来表示，`0\r\n\r\n`(字符形式，字节形式为`0x300d0a0d0a`)

## Chunked完整结构
`[Chunk size][Chunk data][Chunk boundnary]...0\r\n\r\n`
                                                |
                                             结束标记