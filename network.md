### http
超文本传输协议
## http的常见状态码？
2xx表示成功 204没有数据返回 200有数据返回
3xx表示重定位需要重新发送请求 301永久重定向/302临时重定向 304进行采用本地缓存
4xx表示客户端的错误 403是被禁止访问没有权限 404表示请求的资源在服务器上未找到
5xx表示服务器端的错误 501表示暂不支持客户端请求的内容 502表示后端错误 503服务端网络忙
## http的常见字段有哪些？
host：用来制定域名，区分同一个服务器上不同的域名比如4399.com/7k7k.com
length：用来告诉对方自己的数据到多少个字节为止，同时http的head用回车符、换行符隔开，防止tcp粘包
type：告诉对方自己发送的是什么类型的数据以及自己接收什么类型的数据
connection：用来表述自己是不是保持长连接 keep-alive
Encoding：压缩格式
## get和post的区别？
get的请求参数放在URL里的？有限制，一般用做查询/只读 因为不改变数据是安全且幂等的（可以缓存get请求）
post是放在body里，对请求的资源进行处理，用来做添加数据之类 是不安全的，不幂等的（不可以缓存）
## http缓存的实现
强制缓存 通过http响应头中的cache-control，网页先判断缓存是否过期，未过期就用缓存。
协商缓存 通过etag和last-modify
首先进行强制缓存的判断没有通过就采用协商缓存
## http1.1/2.0/3.0的区别
http1.1 无状态（通过cookie来） 长连接(没有解决响应队头的阻塞) 明文（不安全）
http2.0 是基于https的有几个新特点，头部压缩，通过维护头部表的方式，消除一样的头部信息。

## http和https的区别
http是明文，所以为了安全在HTTP和TCP中间增加SSL/TLS，https增加了SSL/TLS握手，向CA（证书权威机构）申请数字证书。
https采用混合加密的方式（对称和非对称组合）
数字签名：对内容的哈希进行公钥加密，服务器通过私钥解密，防止冒充
数字证书：通过CA进行认证把（数字签名、公钥）通过ca的私钥进行加密，CA再通过公钥进行验证。
加密的方式：
1.服务端将公钥名放到CA进行加密获得数字证书
2.将数字证书发送给客户端
3.客户端对数字证书的ca的数字签名对CA进行认证获得公钥。
4.客户端通过公钥对内容的哈希值（指纹）进行加密，加入数字签名
5.服务端通过私钥解密数字签名
