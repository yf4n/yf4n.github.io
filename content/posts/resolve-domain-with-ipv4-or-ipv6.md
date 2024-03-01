+++
title = "访问域名时使用的是IPv4还是IPv6？"
date = "2024-03-01"
description = "谁来决定控制访问域名时使用的是IPv4还是IPv6地址？"
tags = ["DNS"]
categories = ["技术探讨"]
toc = true
+++

谁来决定控制访问域名时使用的是IPv4还是IPv6地址？
<!--more-->
先说结论，是由DNS Client和建立连接的Client（如TCP Client）来共同决定的，一般情况先是IPv6优先。

在访问域名时，DNS Client会向DNS服务发起请求，获取域名解析结果，该结果根据其发送的请求情况，它可能是一个域名解析列表，包含IPv4和IPv6的地址，也可能只是单一的一个结果。之后再由建立连接的客户端从中选取一个地址进行使用，具体的策略完全取决于客户端的实现。

## 域名解析类型
DNS域名解析类型有很多种，比如A、AAAA、CNAME、TXT等，在提交域名解析工单时也会要求申请者描述域名解析类型，通常情况下使用的是A、AAAA或CNAME。当完成域名解析记录后，DNS客户端在发起域名查询时，DNS服务便会根据查询类型返回对应的结果，如果未查询到则返回未找到。

下面将通过抓包的来更直观的观察，首先先对www.baidu.com域名进行域名解析，根据响应结果可以看到该域名存在4个解析，IPv4、IPv6地址各2个。

```bash
$ nslookup.exe www.baidu.com
服务器:  UnKnown
Address:  10.57.0.96

非权威应答:
名称:    www.a.shifen.com
Addresses:  2409:8c00:6c21:104f:0:ff:b03f:3ae
          2409:8c00:6c21:1051:0:ff:b0af:279a
          39.156.66.18
          39.156.66.14
Aliases:  www.baidu.com
```

观察wireshark中抓到的数据包，总计有2组包，分别对应A类和AAAA类的请求和响应。从请求体中可以观察到，两种请求之间的主要区别在Queries中的Type字段上，该字段表明了DNS查询的类型，也决定了DNS返回的是IPv4地址还是IPv6地址。

## 程序中的域名解析行为
前面已经搞明白了是DNS请求数据包中的Type字段来决定解析出来的地址是IPv4还是IPv6，但是在实际工作场景中大多数情况是通过代码来访问域名，例如请求某个后端服务，或MySQL客户端连接数据库等，客户端在连接时一般不会询问用户访问的域名要解析的是IPv4还是IPv6，此时是谁来决定要解析的类型呢？

答案是程序中的客户端，包括DNS客户端和建立连接的客户端（例如TCP客户端）。DNS客户端一般负责将解析结果列表返回，而建立连接的客户端会根据自身的逻辑选取列表中的某一个地址进行连接。

例如在使用Go建立TCP连接时，会使用net.Dial函数进行建立。由于程序默认不会指定要求解析成IPv4或IPv6，Resolver默认会将所有解析均进行返回，其解析逻辑的部分源码如下，[源码连接](https://github.com/golang/go/blob/a10e42f219abb9c5bc4e7d86d9464700a42c7d57/src/net/dnsclient_unix.go#L612)

```go
func (r *Resolver) goLookupIPCNAMEOrder(ctx context.Context, network, name string, order hostLookupOrder, conf *dnsConfig) (addrs []IPAddr, cname dnsmessage.Name, err error) {
    ...
    // 默认查询A和AAAA类请求，若有特殊要求，则会只查询CNAME、IPv4或IPv6中的一个。
    qtypes := []dnsmessage.Type{dnsmessage.TypeA, dnsmessage.TypeAAAA}
    if network == "CNAME" {
        qtypes = append(qtypes, dnsmessage.TypeCNAME)
    }
    switch ipVersion(network) {
    case '4':
        qtypes = []dnsmessage.Type{dnsmessage.TypeA}
    case '6':
        qtypes = []dnsmessage.Type{dnsmessage.TypeAAAA}
    }
    ...
    // 在查询结果返回后，遍历查询结果将所有解析地址返回
    for _, fqdn := range conf.nameList(name) {
        for _, qtype := range qtypes {
        ...
        loop:
            for {
                ...                
                h, err := result.p.AnswerHeader()
                ...
                switch h.Type {
                case dnsmessage.TypeA:
                    ...
                    addrs = append(addrs, IPAddr{IP: IP(a.A[:])})
                    ...
                case dnsmessage.TypeAAAA:
                    ...
                    addrs = append(addrs, IPAddr{IP: IP(aaaa.AAAA[:])})
                    ...
                case dnsmessage.TypeCNAME:
                default:
                }
            }
        }
        ...
    }
    return addrs, cname, nil
}
```

在拿到解析列表后，会优先尝试与IPv6地址连接，连接失败或超时后，则尝试与IPv4地址连接，[源码连接](https://github.com/golang/go/blob/a10e42f219abb9c5bc4e7d86d9464700a42c7d57/src/net/dial.go#L515)

```go
// 该函数参数中的primaries和fallbacks传入的分别为IPv6地址列表和IPv4地址列表
// 具体逻辑中，先尝试连接primaries中的地址，失败或超时后连接fallbacks的地址
// 若全部失败，则返回error
func (sd *sysDialer) dialParallel(ctx context.Context, primaries, fallbacks addrList) (Conn, error) {
    startRacer := func(ctx context.Context, primary bool) {
        ras := primaries
        if !primary {
            ras = fallbacks
        }
        // 尝试连接地址，成功返回建立的连接c，失败返回错误信息err
        c, err := sd.dialSerial(ctx, ras)
        select {
        case results <- dialResult{...}:
        case <-returned:
            ...
        }
    }
    ...
    for {
        select {
        case <-fallbackTimer.C:
            fallbackCtx, fallbackCancel := context.WithCancel(ctx)
            defer fallbackCancel()
            go startRacer(fallbackCtx, false)
        case res := <-results:
            if res.error == nil {
                return res.Conn, nil
            }
            ...
        }
    }
}
```

另外也查阅了[RFC 6724 Section - 2](https://datatracker.ietf.org/doc/html/rfc6724#section-2)，该文档对Internet协议版本6（IPv6）的默认地址选择进行了描述和约定。在对算法运行上下文描述中，存在下方的一段描述


> In this implementation architecture, applications use APIs such as getaddrinfo() [RFC3493] that return a list of addresses to the application. This list might contain both IPv6 and IPv4 addresses (sometimes represented as IPv4-mapped addresses). The application then passes a destination address to the network stack with connect() or sendto(). The application would then typically try the first address in the list, looping over the list of addresses until it finds a working address. In any case, the network layer is never in a situation where it needs to choose a destination address from several alternatives. The application might also specify a source
>
> ...
>
> Well-behaved applications SHOULD NOT simply use the first address returned from an API such as getaddrinfo() and then give up if it fails. For many applications, it is appropriate to iterate through the list of addresses returned from getaddrinfo() until a working address is found. For other applications, it might be appropriate to try multiple addresses in parallel (e.g., with some small delay in between) and use the first one to succeed.

该描述表明获取域名解析的函数，例如getaddrinfo()会返回一个地址列表，其可能会同时包含IPv4和IPv6的地址，应用程序可以从列表中依次尝试每个地址直到找到一个可以使用的为止。但好的应用不应简单的遍历，而是应该并行的尝试多个地址，快速地返回一个可用的。也就是说目前没有一个强制约束要求客户端如何处理DNS返回的解析结果，一切都是客户端的研发人员/组织决定的。

总结一下，以建立TCP连接为例，在程序中尝试与域名创建连接时，DNS负责解析，TCP客户端负责从解析结果中选取地址进行连接，但由于对两种客户端的实现并无约束，所以访问域名时使用的到底是IPv4还是IPv6，还需要根据具体使用的客户端而定（一般情况是IPv6优先），在使用前需要进行充分测试以验证结果是否符合预期。
