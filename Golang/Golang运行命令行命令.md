### Go运行Shell命令

```go
cmd := exec.Command("echo 1") //此处填命令
// cmd := exec.Command("kill","-9","$pid") 带参数形式
if output, err := cmd.CombinedOutput(); err != nil {
		logx.Info(string(output))  //获取执行命令的输出
		logx.Error(err)
		return err
}
```

