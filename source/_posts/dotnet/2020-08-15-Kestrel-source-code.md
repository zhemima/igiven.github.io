---
title : "Kestrel源码分析"
---

Kestrel是[http://ASP.NET](https://link.zhihu.com/?target=http%3A//ASP.NET) Core框架内置的默认Web Server 什么是Web Server? 根据[维基百科](https://link.zhihu.com/?target=https%3A//en.wikipedia.org/wiki/Web_server)的定义: Web Server是可以处理来自客户端的HTTP协议请求并返回网页的软件或硬件。 因此Kestrel的主要功能就是接收来自网络客户端的HTTP请求，并根据请求返回对应的网页（数据也是一种网页）。

## 定义 - `IServer`、`IHttpApplication<TContext>`

------

[http://ASP.NET](https://link.zhihu.com/?target=http%3A//ASP.NET) Core定义了两个基本的接口`IServer`，及`IHttpApplication<TContext>`，`IServer`接口定义了Web Server的基本功能，`IHttpApplication<TContext>`则定义了处理HTTP协议的应用程序的基本功能，我们首先来看下这两个定义:

## Web 服务器 - `IServer`

```text
namespace Microsoft.AspNetCore.Hosting.Server
{
    /// <summary>
    /// Represents a server.
    /// </summary>
    public interface IServer : IDisposable
    {
        /// <summary>
        /// A collection of HTTP features of the server.
        /// </summary>
        IFeatureCollection Features { get; }

        /// <summary>
        /// Start the server with an application.
        /// </summary>
        /// <param name="application">An instance of <see cref="IHttpApplication{TContext}"/>.</param>
        /// <typeparam name="TContext">The context associated with the application.</typeparam>
        /// <param name="cancellationToken">Indicates if the server startup should be aborted.</param>
        Task StartAsync<TContext>(IHttpApplication<TContext> application, CancellationToken cancellationToken);

        /// <summary>
        /// Stop processing requests and shut down the server, gracefully if possible.
        /// </summary>
        /// <param name="cancellationToken">Indicates if the graceful shutdown should be aborted.</param>
        Task StopAsync(CancellationToken cancellationToken);
    }
}
```

`Features` 是一个功能集合，其中可以包含所有应用程序需要的，用以处理HTTP协议各个阶段和组成部分的功能集，以接口的形式注入到`Features`中。

`StartAsync`方法可以启动IServer对象，用来接受用户请求。包含两个参数：`IHttpApplication<TContext>`和`CancellationToken`。 `IHttpApplicatoin<TContext>`是最终处理HTTP请求的应用程序入口点，在ASP.NET Core应用程序中，默认的`IHttpApplication<TContext>`实现是：`HostingApplication`，我们会在稍后的部分进行详细的介绍。 而`CancellationToken`用来响应中断应用程序启动的请求。

`StopAsync`方法用来处理停止服务的请求，接受一个参数`CancellationToken`，用来响应中断停止应用程序的请求。

## Http应用程序 - `IHttpApplication<TContext>`

```text
namespace Microsoft.AspNetCore.Hosting.Server
{
    /// <summary>
    /// Represents an application.
    /// </summary>
    /// <typeparam name="TContext">The context associated with the application.</typeparam>
    public interface IHttpApplication<TContext>
    {
        /// <summary>
        /// Create a TContext given a collection of HTTP features.
        /// </summary>
        /// <param name="contextFeatures">A collection of HTTP features to be used for creating the TContext.</param>
        /// <returns>The created TContext.</returns>
        TContext CreateContext(IFeatureCollection contextFeatures);

        /// <summary>
        /// Asynchronously processes an TContext.
        /// </summary>
        /// <param name="context">The TContext that the operation will process.</param>
        Task ProcessRequestAsync(TContext context);

        /// <summary>
        /// Dispose a given TContext.
        /// </summary>
        /// <param name="context">The TContext to be disposed.</param>
        /// <param name="exception">The Exception thrown when processing did not complete successfully, otherwise null.</param>
        void DisposeContext(TContext context, Exception exception);
    }
}
```

`IHttpApplication<TContext>`接口的定义包含了三个方法： `CreateContext`方法用来创建处理请求的上下文中所需要的所有相关数据，组成`Context`对象，由接口的实现自己定义类型， `ProcessRequestAsync`方法使用`CreateContext`方法创建的`Context`对象处理本次请求。 `DisposeContext`方法在完成请求的处理后，负责释放`Context`对象。

## 实现 - `KestrelServer`

[http://ASP.NET](https://link.zhihu.com/?target=http%3A//ASP.NET) Core提供了默认的`IServer`：`KestrelServer`，下面我们就来看看`KestrelServer`具体都做了些什么。

> `KestrelServer` 定义在dotnet/aspnetcore项目中（[GITHUB REPO](https://link.zhihu.com/?target=https%3A//github.com/dotnet/aspnetcore)）。 项目名称为：Microsoft.AspNetCore.Server.Kestrel.Core 名称空间:Microsoft.AspNetCore.Server.Kestrel.Core [源代码](https://link.zhihu.com/?target=https%3A//github.com/dotnet/aspnetcore/blob/master/src/Servers/Kestrel/Core/src/KestrelServer.cs)

## 服务器启动：端口监听，协议解析及请求处理。

我们先看一下`KestrelServer`.`StartAsync()`方法的代码实现：

```text
public async Task StartAsync<TContext>(IHttpApplication<TContext> application, CancellationToken cancellationToken)
        {
            try
            {
                if (!BitConverter.IsLittleEndian)
                {
                    throw new PlatformNotSupportedException(CoreStrings.BigEndianNotSupported);
                }

                ValidateOptions();

                if (_hasStarted)
                {
                    // The server has already started and/or has not been cleaned up yet
                    throw new InvalidOperationException(CoreStrings.ServerAlreadyStarted);
                }
                _hasStarted = true;

                ServiceContext.Heartbeat?.Start();

                async Task OnBind(ListenOptions options)
                {
                    // Add the HTTP middleware as the terminal connection middleware
                    options.UseHttpServer(ServiceContext, application, options.Protocols);

                    var connectionDelegate = options.Build();

                    // Add the connection limit middleware
                    if (Options.Limits.MaxConcurrentConnections.HasValue)
                    {
                        connectionDelegate = new ConnectionLimitMiddleware(connectionDelegate, Options.Limits.MaxConcurrentConnections.Value, Trace).OnConnectionAsync;
                    }

                    var connectionDispatcher = new ConnectionDispatcher(ServiceContext, connectionDelegate);
                    var transport = await _transportFactory.BindAsync(options.EndPoint).ConfigureAwait(false);

                    // Update the endpoint
                    options.EndPoint = transport.EndPoint;
                    var acceptLoopTask = connectionDispatcher.StartAcceptingConnections(transport);

                    _transports.Add((transport, acceptLoopTask));
                }

                await AddressBinder.BindAsync(_serverAddresses, Options, Trace, OnBind).ConfigureAwait(false);
            }
            catch (Exception ex)
            {
                Trace.LogCritical(0, ex, "Unable to start Kestrel.");
                Dispose();
                throw;
            }
        }
```

`Kestrel`首先会检查服务器的[字节序](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/zh-hans/%E5%AD%97%E8%8A%82%E5%BA%8F)，目前是不支持大端序的。 然后检查最大请求长度限制的设置项，以及服务器是否已经启动。

最后，通过`AddressBinder`对预先配置的IP地址或终结点(EndPoint)名称进行监听，开始接受客户端的请求。

当每有一个新的HTTP请求通过TCP协议或其他协议和服务器成功简历连接后，AddressBinder使用`ThreadPool.UnsafeQueueUserWorkItem()`方法将`OnBind()`方法添加到线程池中，等待线程池的调度。

如果此时进程有可用的线程，就会调用`OnBind()`方法，处理用户的HTTP请求。

`OnBind()`方法默认使用`HttpConnectionMiddleware<ServiceContext>`中间件，处理新接入的用户请求，当设置了`MaxConcurrentConnections`值为`True`时，则会默认使用`ConnectionLimitMiddleware`中间件，限制最大可用连接数，如果当前请求数已经达到最大可接受连接数，则拒绝用户的请求并断开连接，否则调用`HttpConnectionMiddleware<ServiceContext>`中间件，继续处理用户的请求。

## 处理HTTP请求 - `HttpConnectionMiddleware<ServiceContext>`、`HttpConnection`

`HttpConnectionMiddleware<ServiceContext>`中间件负责组装连接相关的上下文数据`HttpConnectionContext`，并使用`HttpConnection`类处理用户请求。

```text
internal class HttpConnectionMiddleware<TContext>
    {
        private readonly ServiceContext _serviceContext;
        private readonly IHttpApplication<TContext> _application;
        private readonly HttpProtocols _protocols;

        public HttpConnectionMiddleware(ServiceContext serviceContext, IHttpApplication<TContext> application, HttpProtocols protocols)
        {
            _serviceContext = serviceContext;
            _application = application;
            _protocols = protocols;
        }

        public Task OnConnectionAsync(ConnectionContext connectionContext)
        {
            var memoryPoolFeature = connectionContext.Features.Get<IMemoryPoolFeature>();

            var httpConnectionContext = new HttpConnectionContext
            {
                ConnectionId = connectionContext.ConnectionId,
                ConnectionContext = connectionContext,
                Protocols = _protocols,
                ServiceContext = _serviceContext,
                ConnectionFeatures = connectionContext.Features,
                MemoryPool = memoryPoolFeature.MemoryPool,
                Transport = connectionContext.Transport,
                LocalEndPoint = connectionContext.LocalEndPoint as IPEndPoint,
                RemoteEndPoint = connectionContext.RemoteEndPoint as IPEndPoint
            };

            var connection = new HttpConnection(httpConnectionContext);

            return connection.ProcessRequestsAsync(_application);
        }
    }
```

### HTTP版本控制 - `HttpConnection`

当用户创建`HttpConnection`类时，在初始化过程中，会根据用户请求声明的HTTP协议版本，分别创建对应版本的Connection类，并使用该类处理用户请求：

```text
public async Task ProcessRequestsAsync<TContext>(IHttpApplication<TContext> httpApplication)
        {
            try
            {
                // Ensure TimeoutControl._lastTimestamp is initialized before anything that could set timeouts runs.
                _timeoutControl.Initialize(_systemClock.UtcNowTicks);

                IRequestProcessor requestProcessor = null;

                switch (SelectProtocol())
                {
                    case HttpProtocols.Http1:
                        // _http1Connection must be initialized before adding the connection to the connection manager
                        requestProcessor = _http1Connection = new Http1Connection<TContext>(_context);
                        _protocolSelectionState = ProtocolSelectionState.Selected;
                        break;
                    case HttpProtocols.Http2:
                        // _http2Connection must be initialized before yielding control to the transport thread,
                        // to prevent a race condition where _http2Connection.Abort() is called just as
                        // _http2Connection is about to be initialized.
                        requestProcessor = new Http2Connection(_context);
                        _protocolSelectionState = ProtocolSelectionState.Selected;
                        break;
                    case HttpProtocols.None:
                        // An error was already logged in SelectProtocol(), but we should close the connection.
                        break;
                    default:
                        // SelectProtocol() only returns Http1, Http2 or None.
                        throw new NotSupportedException($"{nameof(SelectProtocol)} returned something other than Http1, Http2 or None.");
                }

                _requestProcessor = requestProcessor;

                if (requestProcessor != null)
                {
                    var connectionHeartbeatFeature = _context.ConnectionFeatures.Get<IConnectionHeartbeatFeature>();
                    var connectionLifetimeNotificationFeature = _context.ConnectionFeatures.Get<IConnectionLifetimeNotificationFeature>();

                    // These features should never be null in Kestrel itself, if this middleware is ever refactored to run outside of kestrel,
                    // we'll need to handle these missing.
                    Debug.Assert(connectionHeartbeatFeature != null, nameof(IConnectionHeartbeatFeature) + " is missing!");
                    Debug.Assert(connectionLifetimeNotificationFeature != null, nameof(IConnectionLifetimeNotificationFeature) + " is missing!");

                    // Register the various callbacks once we're going to start processing requests

                    // The heart beat for various timeouts
                    connectionHeartbeatFeature?.OnHeartbeat(state => ((HttpConnection)state).Tick(), this);

                    // Register for graceful shutdown of the server
                    using var shutdownRegistration = connectionLifetimeNotificationFeature?.ConnectionClosedRequested.Register(state => ((HttpConnection)state).StopProcessingNextRequest(), this);

                    // Register for connection close
                    using var closedRegistration = _context.ConnectionContext.ConnectionClosed.Register(state => ((HttpConnection)state).OnConnectionClosed(), this);

                    await requestProcessor.ProcessRequestsAsync(httpApplication);
                }
            }
            catch (Exception ex)
            {
                Log.LogCritical(0, ex, $"Unexpected exception in {nameof(HttpConnection)}.{nameof(ProcessRequestsAsync)}.");
            }
            finally
            {
                if (_http1Connection?.IsUpgraded == true)
                {
                    _context.ServiceContext.ConnectionManager.UpgradedConnectionCount.ReleaseOne();
                }
            }
        }
```

HTTP1和HTTP2处理HTTP协议的方式有所不同，HTTP1协议解析完成后，会立即调用`IHttpApplication<TContext>`处理请求，HTTP2协议解析完成后，会再次调用`ThreadPool.UnsafeQueueUserWorkItem()`方法等待线程池可用线程。

## 结束语

`Kestrel`服务的代码量并不下，其中主要是辅助接受用户请求和解析HTTP协议的代码，在这里不做详细的介绍，各位读者有兴趣的，可以详细阅读源代码。

我们看到，`Kestrel`服务在接受和处理请求时，都用到了线程池，可以极大的提高服务器的吞吐量。

后面，我们还会详细介绍系统默认的`IHttpApplication<TContext>`实现，看看ASP.NET Core是如何将HTTP转发到Controller和Action，其中又有哪些精妙的代码呢。