### 一、例子

#### 1.1 返回错误的例子

```go
g := new(errgroup.Group)
 var urls = []string{
  "http://www.golang.org/",
  "http://www.google.com/",
  "http://www.somestupidname.com/",
 }
 for _, url := range urls {
  // Launch a goroutine to fetch the URL.
  url := url // https://golang.org/doc/faq#closures_and_goroutines
  g.Go(func() error {
   // Fetch the URL.
   resp, err := http.Get(url)
   if err == nil {
    resp.Body.Close()
   }
   return err
  })
 }
 // Wait for all HTTP fetches to complete.
 if err := g.Wait(); err == nil {
  fmt.Println("Successfully fetched all URLs.")
 }
```

### 解析

* errgroup实现依赖了sync.WaitGroup，使用sync.Once的保证下，把第一个业务返回的err直接保存，然后在Wait函数里面返回
* errgroup还提供了第二个功能，限制并发数，如果对一个有容量的chan进入写操作，满了之后可以卡住的特性。TryGo可以使用了select default 实现的

### 源代码

```go
// Copyright 2016 The Go Authors. All rights reserved.
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file.

// Package errgroup provides synchronization, error propagation, and Context
// cancelation for groups of goroutines working on subtasks of a common task.
//
// [errgroup.Group] is related to [sync.WaitGroup] but adds handling of tasks
// returning errors.
package errgroup

import (
 "context"
 "fmt"
 "sync"
)

type token struct{}

// A Group is a collection of goroutines working on subtasks that are part of
// the same overall task.
//
// A zero Group is valid, has no limit on the number of active goroutines,
// and does not cancel on error.
type Group struct {
 cancel func(error)

 wg sync.WaitGroup

 sem chan token

 errOnce sync.Once
 err     error
}

func (g *Group) done() {
 if g.sem != nil {
  <-g.sem
 }
 g.wg.Done()
}

// WithContext returns a new Group and an associated Context derived from ctx.
//
// The derived Context is canceled the first time a function passed to Go
// returns a non-nil error or the first time Wait returns, whichever occurs
// first.
func WithContext(ctx context.Context) (*Group, context.Context) {
 ctx, cancel := withCancelCause(ctx)
 return &Group{cancel: cancel}, ctx
}

// Wait blocks until all function calls from the Go method have returned, then
// returns the first non-nil error (if any) from them.
func (g *Group) Wait() error {
 g.wg.Wait()
 if g.cancel != nil {
  g.cancel(g.err)
 }
 return g.err
}

// Go calls the given function in a new goroutine.
// It blocks until the new goroutine can be added without the number of
// active goroutines in the group exceeding the configured limit.
//
// The first call to return a non-nil error cancels the group's context, if the
// group was created by calling WithContext. The error will be returned by Wait.
func (g *Group) Go(f func() error) {
 if g.sem != nil {
  g.sem <- token{}
 }

 g.wg.Add(1)
 go func() {
  defer g.done()

  if err := f(); err != nil {
   g.errOnce.Do(func() {
    g.err = err
    if g.cancel != nil {
     g.cancel(g.err)
    }
   })
  }
 }()
}

// TryGo calls the given function in a new goroutine only if the number of
// active goroutines in the group is currently below the configured limit.
//
// The return value reports whether the goroutine was started.
func (g *Group) TryGo(f func() error) bool {
 if g.sem != nil {
  select {
  case g.sem <- token{}:
   // Note: this allows barging iff channels in general allow barging.
  default:
   return false
  }
 }

 g.wg.Add(1)
 go func() {
  defer g.done()

  if err := f(); err != nil {
   g.errOnce.Do(func() {
    g.err = err
    if g.cancel != nil {
     g.cancel(g.err)
    }
   })
  }
 }()
 return true
}

// SetLimit limits the number of active goroutines in this group to at most n.
// A negative value indicates no limit.
//
// Any subsequent call to the Go method will block until it can add an active
// goroutine without exceeding the configured limit.
//
// The limit must not be modified while any goroutines in the group are active.
func (g *Group) SetLimit(n int) {
 if n < 0 {
  g.sem = nil
  return
 }
 if len(g.sem) != 0 {
  panic(fmt.Errorf("errgroup: modify limit while %v goroutines in the group are still active", len(g.sem)))
 }
 g.sem = make(chan token, n)
}
```
