# XLog增强版

基于XLog 1.11.0分支拓展，源仓库地址：

https://github.com/elvishew/xLog

相关使用配置可以参考源项目。

## 修改的动机

原项目整体设计来讲还是很不错的，但是在动态tag输出上简直就是一大败笔！！！

和原生的Android Log调用设计完全不同。如果想动态输出tag，那么，需要这样：

```
     XLog.tag("xppp").build().i("xxxxx");
```
而且这是一次性的打印，无法记录到日志，如果想要记录到日志还要重新设置一堆的参数、printer...


从性能角度来看内部实现，每一次动态的输出tag，都会new 新的Logger：

```

    public static com.elvishew.xlog.Logger.Builder tag(String tag) {
        return (new com.elvishew.xlog.Logger.Builder()).tag(tag);
    }

```

完全不考虑性能开销啊！！！


## 修改源码

核心的实现类在Logger中：

```java

  private void printlnInternal(int logLevel, String msg) {
        printlnInternal(null,logLevel,msg);
        }
        
  private void printlnInternal(String tag,int logLevel, String msg) {
    if(tag == null){
      tag = logConfiguration.tag;
    }
    String thread = logConfiguration.withThread
        ? logConfiguration.threadFormatter.format(Thread.currentThread())
        : null;
    String stackTrace = logConfiguration.withStackTrace
        ? logConfiguration.stackTraceFormatter.format(
        StackTraceUtil.getCroppedRealStackTrack(new Throwable().getStackTrace(),
            logConfiguration.stackTraceOrigin,
            logConfiguration.stackTraceDepth))
        : null;

    if (logConfiguration.interceptors != null) {
      LogItem log = new LogItem(logLevel, tag, thread, stackTrace, msg);
      for (Interceptor interceptor : logConfiguration.interceptors) {
        log = interceptor.intercept(log);
        if (log == null) {
          // Log is eaten, don't print this log.
          return;
        }

        // Check if the log still healthy.
        if (log.tag == null || log.msg == null) {
          Platform.get().error("Interceptor " + interceptor
              + " should not remove the tag or message of a log,"
              + " if you don't want to print this log,"
              + " just return a null when intercept.");
          return;
        }
      }

      // Use fields after interception.
      logLevel = log.level;
      tag = log.tag;
      thread = log.threadInfo;
      stackTrace = log.stackTraceInfo;
      msg = log.msg;
    }

    printer.println(logLevel, tag, logConfiguration.withBorder
        ? logConfiguration.borderFormatter.format(new String[]{thread, stackTrace, msg})
        : ((thread != null ? (thread + SystemCompat.lineSeparator) : "")
        + (stackTrace != null ? (stackTrace + SystemCompat.lineSeparator) : "")
        + msg));
  }

```

通过重载printlnInternal方法，添加tag入参，如果传入了自定义的tag，就使用自定义的tag。如果没有传就使用全局的tag



## 使用

为了避免重载参数过多导致调用错误，在保持原有方法不变的前提下，新增it、dt、wt、et等支持动态tag输出的方法：

```java
   XLog.it("xppxxx",resUrl);

   XLog.dt("xppxxx",resUrl);

   XLog.et("xppxxx",resUrl);
```
