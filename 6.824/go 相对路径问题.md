一直报错
```bash
(base) ➜  main git:(master) ✗ go build -race -buildmode=plugin ../mrapps/wc.go
../mrapps/wc.go:14:2: "../mr" is relative, but relative import paths are not supported in module mode
```

看这个解决：
https://learnku.com/go/t/41396

go 项目启用了包管理之后，相对路径的引入就会失效（目前还未找到解决方法），引入的方式需要改成以 go mod 配置的 module 为根的路径。附上 `go.mod` ：

```go
// 6.824-golabs-2020/go.mod
module 6.824-golabs-2020

go 1.14
```

后面将 `6.824-golabs-2020/src/mrapps/wc.go` 中的引入 `../mr` 改成 `6.824-golabs-2020/src/mr` 即可。