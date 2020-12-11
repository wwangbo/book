# errgroup分析

go里面被吐槽的最多的应该就是 `if err != nil` 了吧，今天要介绍的一个神器是多个goroutine中错误统一处理的工具包：[errgroup](https://pkg.go.dev/golang.org/x/sync/errgroup)。





