

# http

## http请求格式

```json
Method Request-URI HTTP-Version
headers CRLF

message-body
```

example

```json
[
    "GET / HTTP/1.1",
    "Host: 127.0.0.1:7878",
    "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:99.0) Gecko/20100101 Firefox/99.0",
    "Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8",
    "Accept-Language: en-US,en;q=0.5",
    "Accept-Encoding: gzip, deflate, br",
    "DNT: 1",
    "Connection: keep-alive",
    "Upgrade-Insecure-Requests: 1",
    "Sec-Fetch-Dest: document",
    "Sec-Fetch-Mode: navigate",
    "Sec-Fetch-Site: none",
    "Sec-Fetch-User: ?1",
    "Cache-Control: max-age=0",
]
```

第一行 Method 是请求的方法，例如 GET、POST 等，Request-URI 是该请求希望访问的目标资源路径，例如 /、/hello/world 等
类似 JSON 格式的数据都是 HTTP 请求报头 headers，例如 "Host: 127.0.0.1:7878"
至于 message-body 是消息体， 它包含了用户请求携带的具体数据，例如更改用户名的请求，就要提交新的用户名数据，至于刚才的 GET 请求，它是没有 message-body 的

## 请求应答

```json
HTTP-Version Status-Code Reason-Phrase CRLF
headers CRLF

message-body
```
应答的格式与请求相差不大，其中 Status-Code 是最重要的，它用于告诉客户端，当前的请求是否成功，若失败，大概是什么原因，它就是著名的 HTTP 状态码，常用的有 200: 请求成功，404 目标不存在，等等。

```json
HTTP/1.1 200 OK\r\n\r\n
```
