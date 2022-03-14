# Gin的中间件探索之Logger

## 简介

​	Logger是Gin官方提供的一个开箱即用的中间件，用于格式化输出请求日志。

## LoggerConfig对象

```golang
// LoggerConfig defines the config for Logger middleware.
type LoggerConfig struct {
   // Optional. Default value is gin.defaultLogFormatter
   Formatter LogFormatter

   // Output is a writer where logs are written.
   // Optional. Default value is gin.DefaultWriter.
   Output io.Writer

   // SkipPaths is a url path array which logs are not written.
   // Optional.
   SkipPaths []string
}
```

​	LoggerConfig是Logger中间件默认的配置结构体。

|   成员    |                    作用                    |
| :-------: | :----------------------------------------: |
| Formatter | 将LogFormatterParams对象格式化成字符串输出 |
|  Output   |                  输出句柄                  |
| SkipPaths |             忽略日志的路径列表             |

## LoggerWithConfig方法

​	这个方法是Logger包内代码的主逻辑，下面主要通过注释来分析一下该方法：

```go
// LoggerWithConfig instance a Logger middleware with config.
func LoggerWithConfig(conf LoggerConfig) HandlerFunc {
  // 先初始化一下formatter和output
   formatter := conf.Formatter
   if formatter == nil {
      formatter = defaultLogFormatter
   }

   out := conf.Output
   if out == nil {
      out = DefaultWriter
   }

   notlogged := conf.SkipPaths

   isTerm := true

   if w, ok := out.(*os.File); !ok || os.Getenv("TERM") == "dumb" ||
      (!isatty.IsTerminal(w.Fd()) && !isatty.IsCygwinTerminal(w.Fd())) {
      isTerm = false
   }
	// 构建忽略路径的Set，这里只var一下，应该是为了节省内存分配
   var skip map[string]struct{}

   if length := len(notlogged); length > 0 {
      skip = make(map[string]struct{}, length)

      for _, path := range notlogged {
         skip[path] = struct{}{}
      }
   }

   return func(c *Context) {
     // 记录请求的一些元信息，这里存储path的原因应该是为了防止context指针传递中被修改
      start := time.Now()
      path := c.Request.URL.Path
      raw := c.Request.URL.RawQuery

      // 这里将先执行后面注册的中间件，包括用户注册的逻辑
      c.Next()

      // 判断是否需要Log
      if _, ok := skip[path]; !ok {
         param := LogFormatterParams{
            Request: c.Request,
            isTerm:  isTerm,
            Keys:    c.Keys,
         }

         // Stop timer
         param.TimeStamp = time.Now()
         param.Latency = param.TimeStamp.Sub(start)

         param.ClientIP = c.ClientIP()
         param.Method = c.Request.Method
         param.StatusCode = c.Writer.Status()
         param.ErrorMessage = c.Errors.ByType(ErrorTypePrivate).String()

         param.BodySize = c.Writer.Size()

         if raw != "" {
            path = path + "?" + raw
         }

         param.Path = path
				// 输出到日志
         fmt.Fprint(out, formatter(param))
      }
   }
}
```

​	可以看到整个Logger的核心在于c.Next()函数的调用，如果这边不先调用该函数，那么记录的时间只是该函数的处理时间，而不是整个请求的时间。

​	因此，Logger中间件最好放在作为整个Handler处理链的 ***第一个中间件*** 注册。

## 默认实现的输出方法

```go
// defaultLogFormatter is the default log format function Logger middleware uses.
var defaultLogFormatter = func(param LogFormatterParams) string {
   var statusColor, methodColor, resetColor string
   if param.IsOutputColor() {
      statusColor = param.StatusCodeColor()
      methodColor = param.MethodColor()
      resetColor = param.ResetColor()
   }
		// 如果延迟大于一分钟则截断掉秒级的尾巴
   if param.Latency > time.Minute {
      param.Latency = param.Latency.Truncate(time.Second)
   }
   return fmt.Sprintf("[GIN] %v |%s %3d %s| %13v | %15s |%s %-7s %s %#v\n%s",
      param.TimeStamp.Format("2006/01/02 - 15:04:05"),
      statusColor, param.StatusCode, resetColor,
      param.Latency,
      param.ClientIP,
      methodColor, param.Method, resetColor,
      param.Path,
      param.ErrorMessage,
   )
}
```

​	这边的实现就没什么可说的，主要是日志彩色输出和延迟的处理打印。
