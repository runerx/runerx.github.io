---
title: AWS S3 复制桶
tags:
  - code
  - null
categories: code
date: 2024-06-17 10:34:12
---


使用aws go sdk批量复制存储桶内容以及权限。使用goroutine提高复制效率。

<!-- more -->



```go
package main

import (
	"bytes"
	"context"
	"fmt"
	"io"
	"log"
	"sync"

	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/config"
	"github.com/aws/aws-sdk-go-v2/service/s3"
	"github.com/aws/aws-sdk-go-v2/service/s3/types"
)

var sourceBucket = "source-buckt"
var targetBucket = "target-bucket"

func GetObjectBody(client *s3.Client, bucketName *string, key *string) io.Reader {
	getObjectOutput, err := client.GetObject(context.TODO(), &s3.GetObjectInput{
		Bucket: bucketName,
		Key:    key,
	})
	if err != nil {
		log.Printf("unable to get object %s, %v", *key, err)
		return nil
	}
	defer getObjectOutput.Body.Close()

	// Read the object's content
	body, err := io.ReadAll(getObjectOutput.Body)
	if err != nil {
		log.Printf("unable to read object %s, %v", *key, err)
		return nil
	}

	// Do something with the object's content, for example, print it
	// fmt.Printf("Object content: %s\n", string(body))

	return bytes.NewReader(body)
}

func main() {
	sourceBucket := &sourceBucket
	targetBucket := &targetBucket
	cfg, err := config.LoadDefaultConfig(context.TODO())
	if err != nil {
		log.Fatal(err)
	}

	// Create an Amazon S3 service client
	client := s3.NewFromConfig(cfg)

	input := &s3.ListObjectsV2Input{
		Bucket: sourceBucket,
	}
	paginator := s3.NewListObjectsV2Paginator(client, input)

	var wg sync.WaitGroup
	concurrency := make(chan struct{}, 100) // 限制并发的通道

	for paginator.HasMorePages() {
		page, err := paginator.NextPage(context.TODO())
		if err != nil {
			fmt.Println("Error listing objects:", err)
			return
		}

		// Iterate over the objects in the current page
		for _, object := range page.Contents {
			wg.Add(1)
			concurrency <- struct{}{} // 向通道发送信号，表示占用一个并发槽

			go func(objectKey string) {
				defer wg.Done()
				// <- concurrency
				defer func() { <-concurrency }() // 释放一个并发槽
				// <- concurrency

				aclcoutput, err := client.GetObjectAcl(context.TODO(), &s3.GetObjectAclInput{
					Bucket: sourceBucket,
					Key:    &objectKey,
				})
				if err != nil {
					log.Println(err)
					return
				}

				// 打印需要处理的内容
				log.Printf("key: %s", objectKey)

				// 获取对象内容
				body := GetObjectBody(client, sourceBucket, &objectKey)
				if body == nil {
					return
				}

				client.PutObject(context.TODO(), &s3.PutObjectInput{
					Bucket: targetBucket,
					Key:    &objectKey,
					Body:   body,
				})

				client.PutObjectAcl(context.TODO(), &s3.PutObjectAclInput{
					Bucket: targetBucket,
					Key:    &objectKey,
					AccessControlPolicy: &types.AccessControlPolicy{
						Grants: aclcoutput.Grants,
						Owner:  aclcoutput.Owner,
					},
				})
			}(aws.ToString(object.Key))
		}
	}

	wg.Wait()
}

```

