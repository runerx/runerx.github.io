---
title: 使用cloudflare的worker功能进行http日志收集
categories: 安全建设
tags: 
- 安全建设
---

cloudflare的log push必须要企业版，每个月至少4000刀以上，可以用worker进行替代

<!-- more -->

## worker代码



```js
// Worker Secrets
// Define LOGDNAINGESTIONKEY
let requests = [];
let workerInception, workerId, requestStartTime, requestEndTime;
let batchIsRunning = false;
const maxRequestsPerBatch = 150;

const logRequests = async (event) => {
  if (!batchIsRunning) {
    event.waitUntil(handleBatch(event));
  }
  if (requests.length >= maxRequestsPerBatch) {
    event.waitUntil(postRequests());
  }
  requestStartTime = Date.now();
  if (!workerInception) workerInception = Date.now();
  if (!workerId) workerId = makeid(6);
  const response = await fetch(event.request);
  requestEndTime = Date.now();
  requests.push(getRequestData(event.request, response));
  return response;
};

const handleBatch = async (event) => {
  batchIsRunning = true;
  await sleep(10000);
  try {
    if (requests.length) event.waitUntil(postRequests());
  } catch (e) {}
  requests = [];
  batchIsRunning = false;
};

const sleep = (ms) => {
  return new Promise((resolve) => {
    setTimeout(resolve, ms);
  });
};

const getRequestData = (request, re) => {
  // console.log(new Map(request.headers));
  // console.log(re)
  let data = {
    app: "cf-http-log",
    timestamp: Date.now(),
    meta: {
      //clientIP: request.headers.get("x-real-ip")|| "empty",
      host: request.headers.get("host"),
      ua: request.headers.get("user-agent"),
      referer: request.headers.get("Referer") || "empty",
      ip: request.headers.get("CF-Connecting-IP"),
      countryCode: (request.cf || {}).country,
      //colo: (request.cf || {}).colo,
      workerInception: workerInception,
      workerId: workerId,
      url: request.url,
      method: request.method,
      x_forwarded_for: request.headers.get("x_forwarded_for") || "0.0.0.0",
      asn: (request.cf || {}).asn,
      cfRay: request.headers.get("cf-ray"),
      //tlsCipher: (request.cf || {}).tlsCipher,
      //tlsVersion: (request.cf || {}).tlsVersion,
      clientTrustScore: (request.cf || {}).clientTrustScore,
      status: (re || {}).status,
      originTime: requestEndTime - requestStartTime,
      //cfCache: re ? re.headers.get("CF-Cache-Status") || "miss" : "MISS",
    },
  };
  //data.line = `${data.meta.status} ${data.meta.countryCode} ${data.meta.cfCache} ${data.meta.originTime}ms ${data.meta.clientIP} ${data.meta.host} ${data.meta.url}`;
  return data;
};

const postRequests = () => {
  let data = JSON.stringify({ lines: requests });
  let myHeaders = new Headers();
  myHeaders.append("Content-Type", "application/json; charset=UTF-8");
  try {
    return fetch(
      `https://log-sec.xxxx.com/cloudflare-log`,
      {
        method: "POST",
        headers: myHeaders,
        body: data,
      }
    ).then((r) => {
      requests = [];
    });
  } catch (err) {
    console.log(err.stack || err);
  }
};

const makeid = (lenght) => {
  let text = "";
  const possible = "ABCDEFGHIJKLMNPQRSTUVWXYZ0123456789";
  for (let i = 0; i < lenght; i++)
    text += possible.charAt(Math.floor(Math.random() * possible.length));
  return text;
};

addEventListener("fetch", (event) => {
  event.passThroughOnException();
  event.respondWith(logRequests(event));
});
```

配置worker路由

```
xxx.com/*
.xxx.com/*
```



## 接收日志的代码

cloudfalre-http-log.go

```go
package main

import (
	"cloudfalre-http-log/es"
	"cloudfalre-http-log/loghub"
	"io"
	"log"
	"net/http"
	"time"

	"gopkg.in/natefinch/lumberjack.v2"
)

func main() {
	log.SetFlags(log.Ldate | log.Ltime | log.Lshortfile)
	//log.SetOutput(io.MultiWriter(os.Stdout, &lumberjack.Logger{
	log.SetOutput(io.Writer(&lumberjack.Logger{
		Filename:   "/security/log/cloudflare/cloudflare-http.log",
		MaxSize:    10,
		MaxBackups: 10,
		MaxAge:     30,
		Compress:   false,
	}))
	//创建es client
	// esClient := es.EsClient()

	// 创建一个channel用于存储事件
	logHub := loghub.NewLoghub()
	// Controller(logHub)
	// // go tet()
	esClient := es.EsClient()
	go es.WriteToES(logHub, esClient)
	controller(logHub)
	log.Fatal(http.ListenAndServe(":33033", nil))
}

func controller(logHub *loghub.Loghub) {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		handleEvent(w, r, logHub)
	})
}

// 处理http监听事件，将其写入到管道
func handleEvent(w http.ResponseWriter, r *http.Request, logHub *loghub.Loghub) {
	body, err := io.ReadAll(r.Body)
	if err != nil {
		log.Printf("Error reading request body: %v", err)
		http.Error(w, "Bad Request", http.StatusBadRequest)

	}

	event := loghub.Event{
		Message:   string(body),
		Timestamp: time.Now(),
	}
	log.Println(event)
	w.WriteHeader(http.StatusOK)

	logHub.EventChan <- event

}
```





loghub.go

```go
package loghub

import "time"

// 定义一个结构体Event用于表示事件来源
type Event struct {
	Message   string    `json:"message"`
	Timestamp time.Time `json:"@timestamp"`
}

type Loghub struct {
	// 创建一个channel用于存储事件
	EventChan chan Event
}

func NewLoghub() *Loghub {
	logHub := Loghub{
		EventChan: make(chan Event, 2000),
	}

	return &logHub
}
```



es.go

```go
package es

import (
	"bytes"
	"cloudfalre-http-log/loghub"
	"context"
	"encoding/json"
	"log"
	"time"

	"github.com/elastic/go-elasticsearch/esapi"
	"github.com/elastic/go-elasticsearch/v7"
	"github.com/google/uuid"
)

func EsClient() *elasticsearch.Client {
	// 配置 Elasticsearch 客户端
	cfg := elasticsearch.Config{
    Addresses: []string{"http://es-server:9200"},
		Username:  "elastic",
		Password:  "sDxxxxxxxxxxxxxV",
	}
	esClient, err := elasticsearch.NewClient(cfg)
	if err != nil {
		log.Fatalf("Error creating Elasticsearch client: %s", err)
	}
	return esClient
}

func WriteToES(logHub *loghub.Loghub, esClient *elasticsearch.Client) {

	// 从channel中读取事件，并写入Elasticsearch
	for event := range logHub.EventChan {
		// 将数据发送到 Elasticsearch
		// 将 event 转换为 JSON 格式的字节数组
		eventBytes, err := json.Marshal(event)
		if err != nil {
			// 错误处理
			log.Println(err)
		}
		// log.Println(string(eventBytes))
		// log.Println(event)
		docID := uuid.New().String()
		// docID := now.Format("2006-01-02T15:04:05") // 使用当前时间作为文档 ID
		req := esapi.IndexRequest{
			Index:      "cf-http-log-" + time.Now().Format("2006-01-02"),
			DocumentID: docID,
			Body:       bytes.NewReader(eventBytes),
			Refresh:    "true",
		}
		res, err := req.Do(context.Background(), esClient)
		// log.Println(res)
		if err != nil {
			log.Println(err)
			//http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		if res.IsError() {
			log.Printf("Error indexing event: %s", res.String())
		}

	}

}

```













