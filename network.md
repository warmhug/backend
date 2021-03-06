# Network

## HTTP

### HTTP 3


### SPDY / HTTP 2

核心优势就是多路复用，简单说来就是将多个请求通过一个 TCP 连接发送。浏览器能不能将 100 个请求通过一个 TCP 连接发送？会出现什么问题？那就是 TCP 协议的 head of line blocking,队头阻塞。
设想这样一个场景，一个页面有 100 个请求，第 99 个请求时，TCP 丢了一个包，TCP 自然会重传，重传时间是 T1，重传成功后，浏览器才能获取到完整页面的响应内容，然后渲染和展示整个页面。也就是说整个页面的加载时间延迟了 T1 时间。在此之前，用户没有得到任何内容。

[http2讲解](http://http2-explained.haxx.se/content/zh/index.html)、
[htt2 and UDP](http://2014.jsconf.eu/speakers/iliyan-peychev-http-20-and-quic-protocols-of-the-near-future.html)

### HTTP 1

在网络体系结构中，TCP 是运输层而 HTTP 是应用层。HTTP 增加了技术复杂性，是因为它需要支持「分块传输编码」。分块传输编码可以在响应数据未完全生成时进行数据传输，此时还无法确定响应信息的具体大小。如果分块中所包含信息的长度为 0，则表示响应信息的结束。

HTTP 协议根本没有长短连接这一说，HTTP 协议是基于请求 / 响应模式的，因此只要服务端给了响应，本次 HTTP 连接就结束了。

HTTP 分为长连接和短连接，其实本质上是说的 TCP 连接。TCP 连接是一个双向的通道，它是可以保持一段时间不关闭的，因此 TCP 连接才有真正的长连接和短连接这一说。HTTP 协议说到底是应用层的协议，而 TCP 才是真正的传输层协议，只有负责传输的这一层才需要建立连接。

HTTP1.1 默认是长连接，也就是默认 Connection 的值就是 keep-alive。好处是：长连接情况下，多个 HTTP 请求可以复用同一个 TCP 连接，这就节省了很多 TCP 连接建立和断开的消耗。

对于客户端来说，不管是长轮询还是短轮询，客户端的动作都是一样的，就是不停的去请求，不同的是服务端，短轮询情况下服务端每次请求不管有没有变化都会立即返回结果，而长轮询情况下，如果有变化才会立即返回结果，而没有变化的话，则不会再立即给客户端返回结果，直到超时为止。但是长轮询也是有坏处的，因为把请求挂起同样会导致资源的浪费，假设还是 1000 个人停留在某个商品详情页面，那就很有可能服务器这边挂着 1000 个线程，在不停检测库存量，这依然是有问题的。　因此，从这里可以看出，不管是长轮询还是短轮询，都不太适用于客户端数量太多的情况，因为每个服务器所能承载的 TCP 连接数是有上限的，这种轮询很容易把连接数顶满。

一种轮询方式是否为长轮询，是根据服务端的处理方式来决定的，与客户端没有关系。轮询的长短，是服务器通过编程的方式手动挂起请求来实现的。

发起一个 HTTP 请求的过程就是建立一个 socket 通信的过程。可以模拟浏览器发起 HTTP 请求，如用 HttpClient 发起；Linux 中的 `curl` 命令，通过 `curl+url` 就可以发起 HTTP 请求。

- 搞清楚 `Expires`、`Last-Modified`、`Etag` 等
- Content-type in a request refers to the type of the data you are sending!
  - [Do I need a content type for http get requests](http://stackoverflow.com/questions/5661596/do-i-need-a-content-type-for-http-get-requests)：Get requests should not have content-type
- Accept：Content-Types that are acceptable for the response.

HTTP 协议本身是一种面向资源的应用层协议，但对 HTTP 协议的使用实际上存在着两种不同的方式：一种是 RESTful 的，它把 HTTP 当成应用层协议，比较忠实地遵守了 HTTP 协议的各种规定；另一种是 SOA 的，它并没有完全把 HTTP 当成应用层协议，而是把 HTTP 协议作为了传输层协议，然后在 HTTP 之上建立了自己的应用层协议。

幂等性并不属于特定的协议，它是分布式系统的一种特性；所以，不论是 SOA 还是 RESTful 的 Web API 设计都应该考虑幂等性。（幂等性是数学中的一个概念，表达的是 N 次变换与 1 次变换的结果相同）

- HTTP GET 方法用于获取资源，不应有副作用，所以是幂等的。（不会改变资源的状态，但不是每次 GET 的结果相同）
- HTTP DELETE 方法用于删除资源，有副作用，但它应该满足幂等性。
- HTTP POST 和 PUT 的区别容易被简单地误认为 “POST 表示创建资源，PUT 表示更新资源”；而实际上，二者均可用于创建资源，更为本质的差别是在幂等性方面。
- POST 所对应的 URI 并非创建的资源本身，而是资源的接收者。比如：POST `http://www.forum.com/articles` 的语义是在这里创建一篇帖子，HTTP 响应中应包含帖子的创建状态以及帖子的 URI。两次相同的 POST 请求会在服务器端创建两份资源，它们具有不同的 URI；所以，POST 方法不具备幂等性。
- 而 PUT 所对应的 URI 是要创建或更新的资源本身。比如：PUT `http://www.forum/articles/4231` 的语义是创建或更新 ID 为 4231 的帖子。对同一 URI 进行多次 PUT 的副作用和一次 PUT 是相同的；因此，PUT 方法具有幂等性。

[合并 HTTP 请求是否真的有意义？](http://www.zhihu.com/question/34401250)

浏览器针对每个域名并发建立的最大 TCP 连接数基本都是 6 个，然后每个连接上串行发送若干个请求。HTTP1.1 协议规定请求只能串行发送。

- 100 个请求下：在 http1.1，keep-alive 是默认的，现代浏览器都有 DNS 缓存，DNS 寻址时间可忽略。
  - 寻址还是会花很少量时间，考虑个别情况下 DNS 缓存失效时需要更多点时间（10ms 左右）。另外 url 检查时间，一般可忽略。
- 3 次握手由于有 keep-alive，一条和一百条都只需一次 TCP 握手 -- 无差别。
- 发送报文 -- 增多了 99 次的 http 请求头，请求之间有停顿（网络延迟 RTT），如果合并后节省延迟时间 RTT*(n-1)。网络延迟低或请求数 n 比较小时，可忽略不计。（4G 以上网络延迟很低）。
  - PC 上的 RTT 大概是 50ms, wifi 为 100ms， 3G 为 200ms，2G 为 400ms。例如：一个 200M 带宽、2000ms 延迟的网络，和一个 2M 带宽，20ms 延迟的网络。
  - 无线环境下头部大小每减少 100 个字节，速度能够提升 20~30ms。因为：上下行带宽严重不对称，上行带宽太小。假设一个请求头部是 800 个字节，如果上行带宽是 100 个字节，那至少得传 8 次才能将一个请求传完。
- 考虑丢包（比如移动网络），合并请求会更有优势。
  - 丢的是 tcp 包？服务器怎么知道丢了，丢了哪些内容 (如 get 请求内容一部分丢了)？浏览器会重新发送，还是自动重发？
- 据说 keep-alive 在经过代理或者防火墙的时候可能会被断开。

[http pipelining](https://en.wikipedia.org/wiki/HTTP_pipelining) pipeline 原理是 客户端可以并行发送多个请求，但是服务器的响应必须按次序返回。一些服务器和代理不支持 pipeline；在 pipeline 中的前一个链接可能会阻塞后边的链接；减缓页面加载速度。Chrome 默认禁止了 pipelining。[原因](https://www.chromium.org/developers/design-documents/network-stack/http-pipelining)

-----

## DNS域名解析

- 输入域名并按下回车后
- 第一步，浏览器会检查缓存中有没有这个域名对应的解析过的 IP 地址，有就结束，没有进入下一步
- 第二步，浏览器查找操作系统缓存中是否有。操作系统也有一个域名解析过程，在 hosts 文件里设置可以将任何域名解析到任何能够访问的 IP 地址。如果指定了，浏览器会使用这个 IP 地址。（早期 Windows 中的域名被入侵黑客劫持问题）
- 前两步都是在本机完成的，如果无法完成解析，就会请求域名服务器了。我们的网络配置中都会有「DNS 服务器地址」，操作系统会把域名发送给 LDNS，也就是本地区的域名服务器。大约 80% 的域名解析到这里完成。
- 第四步，如果 LDNS 没命中，就到 Root Server 域名服务器请求解析。然后 `gTLD Server`，`Name Server 域名服务器`，返回该域名对应的 `IP 和 TTL 值` 被 Local DNS Server 缓存，解析结果返回给用户、缓存到本地系统缓存中、域名解析过程结束。（这中间还有 GTM 负载均衡控制等）
- 可以用 `nslookup`、`dig www.taobao.com` 等命令，跟踪解析过程

CDN 工作机制：CDN = 镜像（Mirror）+ 缓存（Cache）+ 整体负载均衡（GSLB），主要缓存网站中的静态数据。

三种负载均衡架构：链路负载均衡、集群负载均衡、操作系统负载均衡。  
链路负载均衡就是通过 DNS 解析成不同的 IP，用户根据这个 IP 来访问不同的目标服务器。  
集群负载均衡分为硬件和软件负载均衡。硬件负载均衡设备昂贵、如 F5，性能非常好，但访问量超出极限时不能进行动态扩容。软件负载均衡成本低，缺点是一般一次访问请求要经过多次代理服务器，会增加网络延时，如 LVS、HAProxy。  
操作系统负载均衡，是利用操作系统级别的软中断或硬中断，设置多队列网卡等来实现。
