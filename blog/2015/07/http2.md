We are happy to announce that Warp version 3.1.0 and WarpTLS version 3.1.0 have been released. These versions include the following changes:

- Warp: the APIs have been cleaned up as explained in [Cleaning up the Warp APIs](http://www.yesodweb.com/blog/2015/06/cleaning-up-warp-apis).
- WarpTLS: RC4 has been removed from `defaultTlsSettings` since RC4 is not safe anymore. Please read [RFC7465: Prohibiting RC4 Cipher Suites](https://tools.ietf.org/html/rfc7465) and [RC4 NOMORE](http://www.rc4nomore.com/) for more information.

But the main new feature is [HTTP/2](https://tools.ietf.org/html/rfc7540) support!
The latest versions of Firefox and Chrome support HTTP/2 over TLS.
WarpTLS uses HTTP/2 instead of HTTP/1.1 if [TLS ALPN(Application-Layer Protocol Negotiation)](https://tools.ietf.org/html/rfc7301) selects HTTP/2.
So, if you upgrade Warp and WarpTLS in your site serving TLS and anyone visits your site with Firefox or Chrome, your contents are automatically transferred via HTTP/2 over TLS.

HTTP/2 retains the semantics of HTTP/1.1 such as request and response headers, meaning you don't have to modify your WAI applications, just link them to the new version of WarpTLS. Rather, HTTP/2 redesigned its transport to solve the following issues:

1. Redundant headers: HTTP/1.1 repeatedly transfers almost exactly the same headers for every request and response, wasting bandwidth.
2. Poor concurrency: only one request or response can be sent in one TCP connection at a time(request pipelining is not used in practice). What HTTP/1.1 can do is make use of multiple TCP connections, up to 6 per site.
3. Head-of-line blocking: if one request is blocked on a server, no other requests can be sent in the same connection.

To solve the issue 1, HTTP/2 provides a header compression mechanism called [HPACK](https://tools.ietf.org/html/rfc7541).
To fix the issue 2 and 3, HTTP/2 makes just one TCP connection per site and multiplex frames of requests and responses asynchronously. The default number of concurrency is 100.

I guess that the HTTP/2 implementors agree that the most challenging parts of HTTP/2 are HPACK and priority. HPACK is used to define reference sets as well as indices and Huffman encoding. During standardization activities, I found that [reference sets make the spec really complicated, but does not contribute to the compression ratio](http://d.hatena.ne.jp/kazu-yamamoto/20140129/1391057824). My big contribution to HTTP/2 was a proposal to remove reference sets from HPACK. The final HPACK became much simpler.

Since multiple requests and responses are multiplexed in one TCP connection,
priority is important. 
Without priority, the response of a big file download would occupy the connection.
I surveyed priority queues but could not find a suitable technology.
Thus, I needed to invent random heaps by myself.
If time allows, I would like to describe random heaps in this blog someday.
The [http2 library](http://hackage.haskell.org/package/http2) provides
well-tested HPACK and structured priority queues as well as
frame encoders/decoders.

My interest on implementing HTTP/2 in Haskell was how to map
Haskell threads to HTTP/2 elements.
In HTTP/1.1, the role of Haskell threads is clear.
That is, one HTTP (TCP) connection is a Haskell thread.
After trial and error, I finally reached an answer.
Streams of HTTP/2 (roughly, a pair of request and response) is a Haskell thread.
To avoid the overhead of spawning Haskell threads,
I introduced thread pools to Warp.
Yes, Haskell threads shine even in HTTP/2.

HTTP/2 provides plain (non-encrypted) communications, too.
But since Firefox and Chrome require TLS,
TLS is a MUST in practice.
`TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256` is a mandate cipher suite in HTTP/2.
Unfortunately, many pieces were missing in the [tls library](http://hackage.haskell.org/package/tls).
So, it was necessary for me to implement
ALPN, ECDHE(Elliptic curve Diffie-Hellman, ephemeral) and AES GCM(Galois/Counter Mode). They are already merged into the tls and [cryptonite library](http://hackage.haskell.org/package/cryptonite).

My next targets are improving performance of HTTP/2 over TLS and implementing TLS 1.3.

I would like to thank Tatsuhiro Tsujikawa, the author of [nghttp2](https://nghttp2.org/) -- the reference implementation of HTTP/2 and Moto Ishizawa, the author of [h2spec](https://github.com/summerwind/h2spec). Without these tools, I could not make such mature Warp/WarpTLS libraries. They also answered my countless questions.
RFC 7540 says "the Japanese HTTP/2 community provided invaluable contributions,
including a number of implementations as well as numerous technical
and editorial contributions". 
I'm proud of being a member of the community.

Enjoy HTTP/2 in Haskell!
