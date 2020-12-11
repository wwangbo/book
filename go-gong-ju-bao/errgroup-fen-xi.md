# errgroup分析

go里面被吐槽的最多的应该就是 `if err != nil` 了吧，今天要介绍的一个神器是多个goroutine中错误统一处理的工具包：[errgroup](https://pkg.go.dev/golang.org/x/sync/errgroup)。

它的代码非常少，我们直接贴上所有代码:



```text
package errgroup

import (
	"context"
	"sync"
)

type Group struct {
	cancel func() //用于一个goroutine返回error，将信号发送给其他goroutine

	wg sync.WaitGroup //用于等待所有goroutine执行完成

	errOnce sync.Once //保证只会将第一个error返回
	err     error //记录error
}

// 要注意，这里返回的是一个新的ctx
func WithContext(ctx context.Context) (*Group, context.Context) {
	ctx, cancel := context.WithCancel(ctx)
	return &Group{cancel: cancel}, ctx
}

// 等待所有的goroutine执行完，或有一个返回错误
func (g *Group) Wait() error {
	g.wg.Wait()
	if g.cancel != nil {
		g.cancel() 
	}
	return g.err
}


func (g *Group) Go(f func() error) {
	g.wg.Add(1)

	go func() {
		defer g.wg.Done()

		if err := f(); err != nil {//执行函数，如果返回了err，调用cancel停止其他goroutine继续执行
			g.errOnce.Do(func() {
				g.err = err
				if g.cancel != nil {
					g.cancel()
				}
			})
		}
	}()
}
```

官网上有它的样例代码，使用起来非常简单，不再具体分析了。

```text
package main

import (
	"fmt"
	"golang.org/x/sync/errgroup"
	"net/http"
)

func main() {
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
}
```



