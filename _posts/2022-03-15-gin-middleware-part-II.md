# Gin的中间件探索之Recovery

## 简介

 Recovery是Gin官方提供的一个开箱即用的中间件，用于恢复http服务。

## 有趣的栈打印函数

​	我们在平常写代码的时候常常和panic函数打交道，每当遇到panic时，总可以捕捉到具体的代码行以及调用链，这是怎么做到的呢？Recovery中间件的stack函数可以给到你答案：

```go
func stack(skip int) []byte {
   buf := new(bytes.Buffer) // the returned data
   // As we loop, we open files and read them. These variables record the currently
   // loaded file.
   var lines [][]byte
   var lastFile string
  // 这里的意思是在框架里已知的调用栈深度就不再打印了
  // 但这里有待提升的点，如果有除0情况，第一行打印的还是panic信息而不是对应panic的第一位置
   for i := skip; ; i++ { // Skip the expected number of frames
      pc, file, line, ok := runtime.Caller(i)
      if !ok {
         break
      }
      // 打印栈信息
      fmt.Fprintf(buf, "%s:%d (0x%x)\n", file, line, pc)
      if file != lastFile {
         data, err := ioutil.ReadFile(file)
         if err != nil {
            continue
         }
         lines = bytes.Split(data, []byte{'\n'})
         lastFile = file
      }
     // 打印方法名和对应的代码行
      fmt.Fprintf(buf, "\t%s: %s\n", function(pc), source(lines, line))
   }
   return buf.Bytes()
}
```

​	可以看出，stack函数为了打印用户的panic信息做了很多额外的处理，包括拿到panic的文件里对应的方法和对应的行，可以说是更方便了我们debug。

## 核心函数CustomRecoveryWithWriter

```go
func CustomRecoveryWithWriter(out io.Writer, handle RecoveryFunc) HandlerFunc {
   var logger *log.Logger
   if out != nil {
     // 红色警告
      logger = log.New(out, "\n\n\x1b[31m", log.LstdFlags)
   }
   return func(c *Context) {
      defer func() {
         if err := recover(); err != nil {
            // Check for a broken connection, as it is not really a
            // condition that warrants a panic stack trace.
            var brokenPipe bool
            if ne, ok := err.(*net.OpError); ok {
               var se *os.SyscallError
               if errors.As(ne, &se) {
                  if strings.Contains(strings.ToLower(se.Error()), "broken pipe") || strings.Contains(strings.ToLower(se.Error()), "connection reset by peer") {
                     brokenPipe = true
                  }
               }
            }
            if logger != nil {
              // 忽略已知的、框架内的调用栈
               stack := stack(3)
              // dump下http request
               httpRequest, _ := httputil.DumpRequest(c.Request, false)
               headers := strings.Split(string(httpRequest), "\r\n")
               for idx, header := range headers {
                  current := strings.Split(header, ":")
                 // 隐藏鉴权信息，防止安全问题
                  if current[0] == "Authorization" {
                     headers[idx] = current[0] + ": *"
                  }
               }
              // 复原request
               headersToStr := strings.Join(headers, "\r\n")
               if brokenPipe {
                 // 就像上面说的，这里不认为broken pipe是一个panic错误，所以这里普通化打印了
                  logger.Printf("%s\n%s%s", err, headersToStr, reset)
               } else if IsDebugging() {
                  logger.Printf("[Recovery] %s panic recovered:\n%s\n%s\n%s%s",
                     timeFormat(time.Now()), headersToStr, err, stack, reset)
               } else {
                  logger.Printf("[Recovery] %s panic recovered:\n%s\n%s%s",
                     timeFormat(time.Now()), err, stack, reset)
               }
            }
           // 如果链接以及断了，就没必要继续处理该请求了
            if brokenPipe {
               // If the connection is dead, we can't write a status to it.
               c.Error(err.(error)) // nolint: errcheck
               c.Abort()
            } else {
              // 用户定义函数处理err
               handle(c, err)
            }
         }
      }()
     // 上面👆只是在注册recover函数，所以不像logger函数，这里直接顺序处理
      c.Next()
   }
}
```

## 总结

​	整体来说，这里的recovery函数写的还是很贴近真实场景的，简洁并且方便用户定位方法以及具体的产生panic的函数。其中还特殊处理了网络相关的（connection reset 对应 tcp 的reset标识位）的错误。
