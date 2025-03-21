---

title: "【Kratos】中间件(1):Trace与Trace中间件"
date: 2025-03-15 13:53:00 +0800
tags: golang
categories: 【框架与工具】

---

```text
在我们的日常服务监控中，可以通过一条请求链路中不同模块的耗时，分析得到链路中的性能瓶颈处于在哪个位置。
另外，在请求链路日志中添加唯一标识(TraceID)，也能让我们更方便的进行日志分析。
Trace是用来跟踪调试多服务间同一链路下的请求的方式，在Kratos框架中，预置了Trace中间件，使用者通过简单方便的配置，
即可使自己的服务拥有链路追踪的能力。
```

### 1.注入trace中间件

在kratos中，默认支持两种trace模式：服务端(server)和客户端(client)

#### 1.1.tracing.Server()

```go
  // Server returns a new server middleware for OpenTelemetry.
  func Server(opts ...Option) middleware.Middleware { //opts 支持的选择器，见下
  	tracer := NewTracer(trace.SpanKindServer, opts...)
  	return func(handler middleware.Handler) middleware.Handler {
  		return func(ctx context.Context, req interface{}) (reply interface{}, err error) {
  			if tr, ok := transport.FromServerContext(ctx); ok { //从上下文中读取transport
  				var span trace.Span
  				ctx, span = tracer.Start(ctx, tr.Operation(), tr.RequestHeader()) //Start一个新的span，标识当前请求
  				setServerSpan(ctx, span, req)  //设置span中的tag，如请求地址、请求头等信息，方便追踪
  				defer func() { tracer.End(ctx, span, reply, err) }()  //请求结束之后，回收span，完成统计并发送到追踪平台
  			}
  			return handler(ctx, req) //继续执行后续中间件
  		}
  	}
  }
```

设置trace中间件时支持选项如下：

```golang
  // WithPropagator with tracer propagator.
  // 设置传播器
  func WithPropagator(propagator propagation.TextMapPropagator) Option {}

  // WithTracerProvider with tracer provider.
  // By default, it uses the global provider that is set by otel.SetTracerProvider(provider).
  // 设置trace provider，如果没有该选项，则从全局获取trace provider 【见下】
  func WithTracerProvider(provider trace.TracerProvider) Option {}

  // WithTracerName with tracer name
  // 设置tracer name ，标识当前trace
  func WithTracerName(tracerName string) Option {}
```

除了通过选择器注入trace provider，也可以配置全局trace provider

```golang
  var endpoint = "localhost:4318"

  // 本地trace provider
  func InitTracer() {
  	// 创建 exporter,将trace信息导出到追踪平台，如jaeger
  	exporter, err := otlptracehttp.New(context.Background(),
  		otlptracehttp.WithEndpoint(endpoint), //追踪平台endpoint
  		otlptracehttp.WithInsecure(), //开启http访问，不开启则需要https
  		otlptracehttp.WithTimeout(),
  		otlptracehttp.WithProxy(),
      //...
  	)
  	if err != nil {
  		panic(err)
  	}
  	tp := trace.NewTracerProvider(
  		// 在资源中记录有关应用程序的信息
  		trace.WithResource(resource.NewWithAttributes(
  			semconv.SchemaURL, // 语义约定版本的schema URL,不同服务间的语义版本应一致
  			semconv.ServiceNameKey.String("trace_demo"), // 服务名称
  		)),
  		// 基于父span的采样率设置为100%，即不论父采样率为多少，此处的采样率都为100%，尽量不要在生产环境中使用，会有性能影响
  		trace.WithSampler(trace.AlwaysSample()),
  		// 始终确保在生产中批量处理
  		trace.WithBatcher(exporter),
  	)
  	otel.SetTracerProvider(tp)
  }

  func main() {
    //...
    InitTracer()
  }
```

#### 1.2.tracing.Client()

```golang
// Client returns a new client middleware for OpenTelemetry.
  func Client(opts ...Option) middleware.Middleware {
  	tracer := NewTracer(trace.SpanKindClient, opts...)
  	return func(handler middleware.Handler) middleware.Handler {
  		return func(ctx context.Context, req interface{}) (reply interface{}, err error) {
  			if tr, ok := transport.FromClientContext(ctx); ok {
  				var span trace.Span
  				ctx, span = tracer.Start(ctx, tr.Operation(), tr.RequestHeader())
  				setClientSpan(ctx, span, req)
  				defer func() { tracer.End(ctx, span, reply, err) }()
  			}
  			return handler(ctx, req)
  		}
  	}
  }
```

### 2.如何在项目中使用trace

#### 2.1.追踪平台

当我们完成trace provider初始化并且完成追踪平台endpoints配置后，trace provider即可将我们的trace自动发送到追踪器中(如jaeger)

以jaeger举例,我们将endpoints配置为jaeger的默认端口

```golang
  var endpoint = "localhost:4318"
```
向服务发送请求后，可以看到我们的请求已经被追踪平台追踪到了

![jaeger1](assets/pic/2025-03-21/jaeger1.png)

`这里看到jaeger ui 中的service除我们的服务trace_demo以外，还有一个jaeger-all-in-one环境，这是jaeger的集成Agent、Collector、Query和UI的一体化部署环境，只支持将追踪数据保存到内存中。生产环境中，通常会部署独立的Agent、Collector和Query Service，并使用持久化存储如ES,Cassandra`

点击被追踪到的trace，可以看到追踪的开始时间、持续时间、深度、span总数、以及kratos的trace中间件帮我们在span中新增的tag(htto.method,http.route等)

![jaeger2](assets/pic/2025-03-21/jaeger2.png)

#### 2.2.日志扩展traceID,方便日志排查

* 扩展本地log,如zap.Logger

```golang
  var traceFmt = "traceID: %s, spanID: %s,"

  func Debugf(ctx context.Context, format string, a ...interface{}) {
  	logger := LoggerFromContext(ctx)
  	traceID := trace.SpanContextFromContext(ctx).TraceID().String()
  	spanID := trace.SpanContextFromContext(ctx).SpanID().String()
  	logger.Log(zap.DebugLevel, fmt.Sprintf(fmt.Sprintf(traceFmt, traceID, spanID)+format, a...))
  }
```

`如果日志框架有钩子功能的话，也可以把traceID扩展放到钩子函数中，如logrus.Logger`

* gorm日志扩展

```golang
  // 重写gorm日志Trace方法
  // Trace 打印 SQL 执行日志
  func (l *ZapGormLogger) Trace(ctx context.Context, begin time.Time, fc func() (string, int64), err error) {
  	if l.Config.LogLevel <= logger.Silent {
  		return
  	}

  	traceID := trace.SpanContextFromContext(ctx).TraceID().String()
  	elapsed := time.Since(begin)
  	sql, rows := fc()

  	switch {
  	case err != nil && l.Config.LogLevel >= logger.Error:
  		l.ZapLogger.With(zap.String("traceID", traceID)).Sugar().Errorf("sql: %s, rows: %d, elapsed: %v, error: %v", sql, rows, elapsed, err)
  	case elapsed > l.Config.SlowThreshold && l.Config.SlowThreshold != 0 && l.Config.LogLevel >= logger.Warn:
  		l.ZapLogger.With(zap.String("traceID", traceID)).Sugar().Warnf("slow sql: %s, rows: %d, elapsed: %v", sql, rows, elapsed)
  	case l.Config.LogLevel >= logger.Info:
  		l.ZapLogger.With(zap.String("traceID", traceID)).Sugar().Infof("sql: %s, rows: %d, elapsed: %v", sql, rows, elapsed)
  	}
  }

```

*本质上，对于日志的扩充，就是将上下文中的span相关信息打印到日志中。kratos中间件就将span相关信息放到上下文中了*


### 3.补充

#### 3.1.传播器
* 定义：
  传播器是OpenTelemetry(或其他追踪框架)的一种机制，用于将追踪上下文从一个服务序列化(Inject,注入)到载体(如HTTP请求头)，并在另一个服务中反序列化(Extract,提取)出来。
* 作用：
  * 注入(Inject):将当前服务的追踪上下文(TraceID和SpanID)写入到请求头中,发送给下游服务。
  * 提取(Extract):下有服务从接收到的请求头中解析出上下文，恢复到当前服务的追踪系统中。
* 分类：
  传播器有很多，区别在于注入和提取的方式与在请求头中的字段不同。因此，在一套有多服务的系统中，系统间尽量使用相同的传播器，或使用组合传播器
  * TraceContext:基于W3C Trace Context 规范，在请求头中使用traceparent字段和tracestate(可选)字段(比较常用)
  * B3传播器:基于Zipkin的B3传播协议
  * Jaeger传播器
  * 复合传播器：
    * 注入时：将追踪上下文按所有指定传播器的格式写入请求头。
    * 提取时：尝试按照谁许解析头字段，直到成功提取上下文

#### 3.2.采样率
* 定义：
  * 采样率是一个概率值（通常在 0 到 1 之间），表示追踪系统中哪些请求会被完整记录（包括所有 Span），哪些会被忽略。
  例如，采样率 0.1 表示 10% 的请求会被追踪，90% 的请求不会生成完整的追踪数据。
* 作用：
  * 性能优化：追踪每个请求会产生大量数据(span,属性,事件等)，增加CPU、内存、网络开销
  * 数据聚焦：在高流量系统中，仅采样一部分代表性请求，而不是记录所有请求
  * 保留足够的数据以分析问题，避免过载追踪后端(如jaeger)

#### 3.3.语义约定版本
* 定义：
  语义约定是 OpenTelemetry 定义的一组标准化命名规则和属性（Attributes），用于描述追踪（Traces）、指标（Metrics）和日志（Logs）中的元数据。
这些约定确保不同服务、工具和语言生成的追踪数据具有一致的结构和语义，便于分析和可视化。
  简单来说，语义约定版本就是OpenTelemetry对于span元数据的解析不相同，因此我们尽量保持同一套多服务的系统中，使用相同版本的语义约定。并且采用当前追踪平台兼容程度最高的语义约定版本URL，这样的传播与追踪平台解析效果是最佳的。


### 4.总结
trace是一项赋予服务链路追踪与链路分析的工具
其中trace表示一个请求在分布式系统中的完整旅程，由多个span组成，而所有span又由通过个traceID关联
span表示跨度，是trace中的一个操作单元，表示某个服务的一次具体活动(如http请求、数据库处理)。他的核心标识包括TraceID,SpanID,ParentSpanID等。以及，span中有丰富的元数据，可以被发送到追踪平台，协助我们分析。

在分布式系统的传输过程中：
发送请求时，传输器会将追踪的上下文(TraceID,SpanID,采样状态等)注入到请求头。
服务收到请求后，传输器会将追踪的上下文提取出来，并生成一个新的Span(基于请求头中的SpanID即父SpanID)，并将其注入到服务的上下文中。

在服务执行过程中，也可以手动基于当前spanID生成一个子span

```golang
//获取tracer
ts := otel.Tracer("example")
//生成父span,并放到上下文ctx中
ctx, newSpan := ts.Start(context.Background(),"fatherOperation")
//基于上下文的父span，生成子span,并放到新的上下文中
newCtx, childSpan := ts.start(ctx,"childSpan")
```
