---
layout: post
title: "Goroutine pool using generics"
date: 2022-11-13
categories: programming
tags: programming golang generics
usemathjax: false
---

A goroutine pool is a way to achieve "bounded concurrency". For example, if you want to start several concurrent network request, but don't want to overload the server with thousands of concurrent requests, you can use a goroutine pool to achieve bounded concurrency, and have, say, 10 concurrent request at once.

I have used this pattern so frequntly in my applications I got tired of looking up the implementation each time, so I decided to write an implementation using generics.

The original source for this pattern is [here](https://gobyexample.com/worker-pools).

Here is the implementation in less than 50 lines of code:

```golang
package main

type workFunc[P any, R any] func(P) R

type WorkPool[P any, R any] struct {
	workFunc   workFunc[P, R]
	numWorkers int
}

func NewWorkerPool[P any, R any](numWorkers int, f workFunc[P, R]) WorkPool[P, R] {
	return WorkPool[P, R]{f, numWorkers}
}

func (wp WorkPool[P, R]) Run(ps []P) []R {
	parameterChan := make(chan P, len(ps))
	returnChan := make(chan R, len(ps))

	// start the goroutines
	for w := 0; w < wp.numWorkers; w++ {
		go func(pc <-chan P, rc chan<- R) {
			for p := range pc {
				rc <- wp.workFunc(p)
			}
		}(parameterChan, returnChan)
	}

	// put all jobs on queue (channel)
	for _, p := range ps {
		parameterChan <- p
	}
	close(parameterChan) // important

	// read all responses
	rs := []R{}
	for i := 0; i < len(ps); i++ {
		rs = append(rs, <-returnChan)
	}

	return rs
}
```

Here is an example of using this code to make concurrent http requests. I recommend you try this out for yourself with various values for the goroutine pool size and see the results.

```golang
package main

import (
	"io"
	"net/http"
	"strconv"
)

func callHttp() {
	// this work function will make an http request and return
	// a string of it's response
	httpWorkFunc := func(url string) string {
		resp, err := http.Get(url)
		if err != nil {
			return ""
		}
		defer resp.Body.Close()

		body, err := io.ReadAll(resp.Body)
		if err != nil {
			return ""
		}

		return string(body)
	}

	httpPool := NewWorkerPool(40, httpWorkFunc)

	// make input array
	const arrLen = 200
	const baseUrl = "https://jsonplaceholder.typicode.com/todos/"
	urls := make([]string, 0, arrLen)
	for i := 0; i < arrLen; i++ {
		urls = append(urls, baseUrl+strconv.Itoa(i))
	}

	// run the batch job
	httpPool.Run(urls)
}

func main() {
	callHttp()
}

```