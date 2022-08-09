---
layout: post
title: "Heap data structure using Go generics"
date: 2022-08-09
categories: programming
tags: programming golang generics
usemathjax: false
---

While working on my [network stack](/networking/2022/08/07/networking-stack-part-1.html) project, I found myself needing to use a heap data structure. While the Go standard library has a [heap implementation](https://pkg.go.dev/container/heap), this uses interfaces, and so I thought this would be a good opportunity to try out [Go's new support for generics](https://go.dev/doc/tutorial/generics).

You can find all the code from this post [here](https://github.com/mattcarp12/generic-heap).

## Intro to Heaps

Heaps are a data structure that can be used to store a collection of elements in a specific order. There are various implementations of heaps, but the most common one is a binary heap, which is what we'll be making here.

Heaps allow you to get the min or max element of your collection in O(1) time, and insert or remove elements in O(log n) time.

Enough theory, show me the code!

## Implementation

For this implementation, we're using go generics. Starting out, our heap data structure will look something like this:

```go
type Heap[T any] struct {}
```

The `[T any]` is the syntax for a generic type. This is a type that can be anything.

Binary heaps are usually implemented using an array to store the individual elements, so lets add that to our heap data structure.

```go
type Heap[T any] struct {
    data []T
}
```

This says that our heap contains a slice of elements of type `T`.

Next, we need a way to compare two elements of type `T`. For this implementation, we'll have the user supply their own comparison function:

```go
type Heap[T any] struct {
    data []T
    less func(T, T) bool
}
```

Next we need a way to instantiate our heap.

```go
func NewHeap[T any](less func(a, b T) bool) *Heap[T] {
	return &Heap[T]{
		data: []T{},
		less: less,
	}
}
```

The cool thing about go generics is that the compiler can usually infer the type of the generic. 
```go
// Call it like this...
heap := NewHeap(func(a, b int) bool { return a < b })

// ...or like this...
heap := NewHeap[int](func(a, b int) bool { return a < b })
```

And that's about it! The rest of the code is just boilerplate implementing the heap data structure. Enjoy!

## Code

```go
package heap

type Heap[T any] struct {
	data []T
	less func(a, b T) bool
}

func NewHeap[T any](less func(a, b T) bool) *Heap[T] {
	return &Heap[T]{
		data: []T{},
		less: less,
	}
}

func (h *Heap[T]) Pop() T {
	v := h.data[0]
	h.data[0] = h.data[len(h.data)-1]
	h.data = h.data[:len(h.data)-1]
	h.heapify()
	return v
}

func (h *Heap[T]) Push(v T) {
	h.data = append(h.data, v)
	h.siftUp(len(h.data) - 1)
}

func (h *Heap[T]) Peek() T {
	return h.data[0]
}

func (h *Heap[T]) heapify() {
	for i := 0; i < len(h.data); i++ {
		h.siftDown(i)
	}
}

func (h *Heap[T]) siftDown(i int) {
	l := h.left(i)
	r := h.right(i)
	smallest := i
	if l < len(h.data) && h.less(h.data[l], h.data[smallest]) {
		smallest = l
	}
	if r < len(h.data) && h.less(h.data[r], h.data[smallest]) {
		smallest = r
	}
	if smallest != i {
		h.data[i], h.data[smallest] = h.data[smallest], h.data[i]
		h.siftDown(smallest)
	}
}

func (h *Heap[T]) siftUp(i int) {
	p := h.parent(i)
	if p >= 0 && h.less(h.data[i], h.data[p]) {
		h.data[p], h.data[i] = h.data[i], h.data[p]
		h.siftUp(p)
	}
}

func (h *Heap[T]) left(i int) int {
	return 2*i + 1
}

func (h *Heap[T]) right(i int) int {
	return 2*i + 2
}

func (h *Heap[T]) parent(i int) int {
	return (i - 1) / 2
}
```