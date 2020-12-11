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



* 核心原理
  * 利用sync.WaitGroup管理并执行goroutine
* 主要功能
  * 并行工作流
  * 处理错误 或者 优雅降级
  * context 传播与取消
  * 利用局部变量+闭包
* 设计缺陷 --- [改进](https://github.com/go-kratos/kratos/blob/master/pkg/sync/errgroup/errgroup.go)
  * 没有捕获panic，导致程序异常退出 --- 改进 加defer recover
  * 没有限制goroutine数量，存在大量创建goroutine --- 改进 增加一个channel用来消费func
  * WithContext 返回的context可能被异常调用，当其在errgroup中被取消时，影响其它函数 --- 改进 代码内嵌context

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



