---
title: 集成dns爆破
tags:
  - 开发
  - golang
categories: 开发
---

通过dns爆破获取子域名，通过对比多款工具，最终选择[dnsbrute](https://github.com/Q2h1Cg/dnsbrute.git)集成到自己的工具里面。



## 1. 前言

最先测试的是dnsx，这款工具速度快，但是没有解决泛解析问题，所以放弃，尝试gobuster发现结果显示有bug，也放弃，还有一款dnsub是非常好用的，但是只开源了部分代码，所以也放弃了，最终选择了[dnsbrute](https://github.com/Q2h1Cg/dnsbrute.git)，这个工具是对发包速率的控制实现dns的快速查询，比多协程更加的底层，并且泛解析做的非常好。

### 1.1 并发

关于并发与发速率控制，这里参考原作者的下面这段话。

一般通过多线程/多进程的方式将查询并发进行，这样做存在的问题有：
- 多线程/多进程并发的上限取决于系统的文件描述符的限制（linux 默认为 1024，可手动调整：ulimit -n），这使得实际网络 IO 速
度远远小于上下行带宽限制。
- 每次查询都是基于一个完整的 socket 连接，处理连接、等待 IO 造成了大量不必要的时间、性能开销。

解决办法参考了 masscan/zmap 的无状态思路：对于每个 DNS socket 连接，开启两个协程：一个协程负责发送，另一个负责接收。

### 1.2 泛解析

也引用作者的原文。

泛解析一直都是域名爆破中的大问题，目前的解决思路是根据确切不存在的子域名记录（md5(domain).domain）获取黑名单 IP，对爆破
过程的结果进行黑名单过滤。
但这种宽泛的过滤很容易导致漏报，如泛解析记录为 1.1.1.1，但某存在子域名也指向 1.1.1.1，此时这个子域名便可能会被黑名单过
滤掉。
胖学弟提到，可以将 TTL 也作为黑名单规则的一部分，评判的依据是：在权威 DNS 中，泛解析记录的 TTL 肯定是相同的，如果子域名
记录相同，但 TTL 不同，那这条记录可以说肯定不是泛解析记录。



## 2. dns查询

我对dnsbrute和dnsub用同一个字典，相同的协程做了一次对比，dnsbrute有明显的优势。

dnsbrute的主文件是dnsbrute.go。通过命令行传入配置，因为我们要集成到自己的工具，这部分肯定要去掉。

接下来是调用`mixInAPIDict(domain, dict)`函数将子域名字典和域名进行拼接

### 2.1 `mixInAPIDict()`

```go
func mixInAPIDict(domain, dict string) <-chan string {
	subDomainsToQuery := make(chan string)
	mix := make(chan string)

	// mix in
	go func() {
		defer close(subDomainsToQuery)

		domains := map[string]struct{}{}
		for sub := range mix {
			domains[sub] = struct{}{}
		}

		for domain := range domains {
			subDomainsToQuery <- domain
		}
	}()


	go func() {
		defer close(mix)

		// Domain
		mix <- domain

		// Dict
		file, err := os.Open(dict)
		if err != nil {
			log.Fatal(err)
		}
		defer file.Close()

		scanner := bufio.NewScanner(file)
		for scanner.Scan() {
			mix <- scanner.Text() + "." + domain
		}
		if err := scanner.Err(); err != nil {
			log.Fatal(err)
		}
	}()

	return subDomainsToQuery
}
```

分析一下这个函数。

1. 该函数有两个参数，一个是domain，一个是字典的路径。

2. 返回值是一个读出的字符串管道。

3. 前两行定义了两个字符串类型的管道：subDomainsToQuery和mix。

4. 创建一个协程，并且结束后关闭subDomainsToQuery。里面定义了一个map，key,value的类型是string,struct。

   这里需要知道的是通过`struct{}{}`将map变成set，因为go标准库没有set的实现。对于集合来说，只需要 map 的键，而不需要值。struct{}{}的内存占用为空，所以一般选择struct{}{}。集合(set)是无序不重复的，所以比较方便和节省资源。

5. 另一个协程是读取文件内容，写入管道。

   


接下来是调用`Configure()`进行初始化配置



### 2.2 `Configure()`

```go
// Configure 设置发包速率、DNS 服务器地址
func Configure(domain, server string, rate, retry int) (err error) {
	rootDomain = domain
	dnsServerAddress = server
	transmitRate = server
	retryLimit = retry

	if client, err = dns.DialTimeout("udp", server, timeout); err != nil {
		return err
	}

	go send()
	go receive()

	return nil
}
```

把domain server server retryd都赋给全局变量。

```
if client, err = dns.DialTimeout("udp", server, timeout); err != nil {
		return err
	}
```

用来测试dns服务器是否可用，如果超时返回错误

然后启用协程开启send和receive方法



### 2.3 `send()`

```go
// send 发送查询
func send() {
	//1毫秒
	delay := time.Second / time.Nanosecond / time.Duration(transmitRate)

	for {
		select {
		case domain := <-Queries:
			time.Sleep(delay)

			req, _ := requests.LoadOrStore(domain, &Request{0, nil})
			request := req.(*Request)

			// 超出重试次数
			if request.SentCount >= retryLimit {
				requests.Delete(domain)
				continue
			}

			// 发送 DNS 查询
			msg := &dns.Msg{}
			msg.SetQuestion(dns.Fqdn(domain), dns.TypeA)
			if err := client.SetWriteDeadline(time.Now().Add(timeout)); err != nil {
				requests.Delete(domain)
				continue
			}
			if err := client.WriteMsg(msg); err != nil {
				requests.Delete(domain)
				continue
			}

			// 超时重发请求
			request.SentCount++
			request.Timer = time.AfterFunc(timeout, func() {
				log.Println("retry", domain, request.SentCount)
				Queries <- domain
			})
		case <-time.After(3 * timeout):
			log.Println("no more queries")
			close(noMoreQueries)
			return
		}
	}
}
```



1. 定义了一个延迟变量delay，delay的值取决于我们定义的rate的值，也就是用来控制发包速度的变量，通过该变量取代一般工具的并发。

2. 在for循环里，读取出Queries管道里的值，这段代码里requests是一个sync.Map变量，把管道里的值读出来并存入到req里，key是域名，value是定义的Request结构体，request结构体如下

   ```go
   type Request struct {
   	SentCount int
   	Timer     *time.Timer
   }
   ```

   SentCount用来记录发送的次数，如果次数大于我们定义的重发次数，则终止这个域名的查询。Timer等待过时之后

   Timer用来计量超过timeout后重新加入队列

3. `client.SetWriteDeadline`设置一个超时时间，无论怎么样过了这个超时时间直接停止查询
4. `client.WriteMsg`，检查是否成功查询
5. 最后是超过3秒还没有从管道中获取到值就关闭noMoreQueries



### 2.4 `receive()`

1. 如果从noMoreQueries中读取出了数据，就关闭Records管道
2. 创建一个新的接收DNS的msg，设置Deadline和读取信息
3. trimSuffixPoint方法去掉结果中的`.`号，
4. 接收rcord值



