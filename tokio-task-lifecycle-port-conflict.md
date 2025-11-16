title:

Tokio任务生命周期管理不当导致端口绑定冲突

Category:

Concurrency and Async Runtime

Description:

在使用 tokio::spawn 启动后台任务时，如果没有保存返回的 JoinHandle，会导致任务无法被停止和管理，形成 "fire-and-forget" 反模式。当需要重启服务时，旧任务仍在运行并占用资源（如端口），新任务启动失败但错误被后台吞掉，导致系统状态不一致和误导性的成功日志输出。

Background:

在开发异步 Rust 应用时，经常需要启动长期运行的后台任务（如 proxy bridge、HTTP server 等）。这些任务通常使用 tokio::spawn 在独立的异步任务中运行。在服务重启场景下，如果不正确管理任务生命周期，会导致资源泄漏和端口冲突问题。

在实际场景中，一个 ChromeLauncher 需要为每个浏览器实例启动独立的 proxy bridge。当浏览器重启时，launcher 会尝试启动新的 proxy bridge，但如果旧的 bridge 任务没有被停止，就会出现端口冲突。

Problem Code/Description:

```rust
// ❌ 错误模式：Fire-and-Forget
pub struct ChromeLauncher {
    chrome_path: String,
    chrome_port: u16,
    proxy_bridge_port: Option<u16>,
    // ❌ 缺少 JoinHandle 字段，无法管理后台任务
}

impl ChromeLauncher {
    pub async fn launch(&mut self, options: &LaunchOptions) -> Result<Child> {
        // 分配代理
        let proxy_url = self.allocate_proxy_if_needed().await?;

        // ❌ 每次 launch 都启动新任务，但没有停止旧任务
        self.start_proxy_bridge(&proxy_url).await?;

        // 启动浏览器...
        Ok(child)
    }

    async fn start_proxy_bridge(&mut self, upstream_proxy: &str) -> Result<u16> {
        let bridge_port = 8888;
        let listen_addr = format!("127.0.0.1:{}", bridge_port);

        let config = ProxyBridgeConfig {
            listen_addr: listen_addr.parse()?,
            upstream_proxy: upstream_proxy.to_string(),
        };

        // ❌ 启动后台任务但不保存 JoinHandle
        tokio::spawn(async move {
            if let Err(e) = start_proxy_bridge(config).await {
                // ❌ 错误在后台任务中被吞掉，主流程不知道
                tracing::error!("Proxy bridge error: {}", e);
            }
        });

        // ❌ 立即返回成功，但实际绑定可能失败
        tokio::time::sleep(Duration::from_millis(500)).await;
        tracing::info!("✅ Proxy bridge started successfully on port {}", bridge_port);
        Ok(bridge_port)
    }
}

// 模拟 proxy bridge 启动函数
async fn start_proxy_bridge(config: ProxyBridgeConfig) -> Result<()> {
    // 绑定端口（如果端口已被占用，这里会失败）
    let listener = TcpListener::bind(&config.listen_addr).await?;
    // ... 代理转发逻辑
    Ok(())
}
```

**问题场景**：
1. 第一次调用 `launch()` → proxy bridge 任务 A 启动，绑定端口 8888 ✓
2. 浏览器重启，再次调用 `launch()` → proxy bridge 任务 B 尝试启动
3. 任务 B 在后台尝试绑定端口 8888 → 失败（Address already in use）
4. 错误在任务 B 的 async block 中被捕获，仅记录日志
5. 主流程继续执行，输出 "✅ Proxy bridge started successfully"（误导）
6. 浏览器配置代理指向 127.0.0.1:8888
7. 实际连接到的是旧任务 A，使用过期的上游代理配置
8. 代理认证失败（407 Invalid Auth）

Error Message:

```
error binding to 127.0.0.1:8888: error creating server listener: Address already in use (os error 98)
```

但主流程日志仍显示：
```
✅ Proxy bridge started successfully on port 8888
```

导致用户看到：
```
ERR_TUNNEL_CONNECTION_FAILED
407 Invalid Auth
```

Solution:

实现完整的任务生命周期管理，保存 JoinHandle 并在需要时停止旧任务：

```rust
use tokio::task::JoinHandle;
use std::time::Duration;

// ✅ 正确模式：Managed Lifecycle
pub struct ChromeLauncher {
    chrome_path: String,
    chrome_port: u16,
    proxy_bridge_port: Option<u16>,
    /// ✅ 保存后台任务的 handle，用于停止和管理
    proxy_bridge_handle: Option<JoinHandle<()>>,
}

impl ChromeLauncher {
    pub fn new(chrome_path: String, chrome_port: u16) -> Self {
        Self {
            chrome_path,
            chrome_port,
            proxy_bridge_port: None,
            proxy_bridge_handle: None, // 初始化为 None
        }
    }

    pub async fn launch(&mut self, options: &LaunchOptions) -> Result<Child> {
        // ✅ 在启动新 bridge 前，先停止旧的
        self.stop_proxy_bridge().await;

        // 分配代理
        let proxy_url = self.allocate_proxy_if_needed().await?;

        // 启动新的 proxy bridge
        self.start_proxy_bridge(&proxy_url).await?;

        // 启动浏览器...
        Ok(child)
    }

    /// ✅ 停止 proxy bridge 任务并释放端口
    async fn stop_proxy_bridge(&mut self) {
        if let Some(handle) = self.proxy_bridge_handle.take() {
            tracing::info!("Stopping old proxy bridge task...");

            // 中止任务
            handle.abort();

            // 等待端口释放（操作系统需要时间）
            tokio::time::sleep(Duration::from_millis(100)).await;

            tracing::info!("Old proxy bridge stopped");
        }

        // 清除端口记录
        self.proxy_bridge_port = None;
    }

    async fn start_proxy_bridge(&mut self, upstream_proxy: &str) -> Result<u16> {
        let bridge_port = 8888 + (self.chrome_port - 9222);
        let listen_addr = format!("127.0.0.1:{}", bridge_port);

        let config = ProxyBridgeConfig {
            listen_addr: listen_addr.parse()
                .map_err(|e| Error::Config(format!("Invalid address: {}", e)))?,
            upstream_proxy: upstream_proxy.to_string(),
        };

        tracing::info!("Starting proxy bridge: {} -> {}", listen_addr, upstream_proxy);

        // ✅ 启动后台任务并保存 JoinHandle
        let handle = tokio::spawn(async move {
            if let Err(e) = start_proxy_bridge(config).await {
                tracing::error!("Proxy bridge error on port {}: {}", bridge_port, e);
            }
        });

        // ✅ 保存 handle，以便后续停止
        self.proxy_bridge_handle = Some(handle);

        // 等待绑定完成
        tokio::time::sleep(Duration::from_millis(500)).await;

        tracing::info!("✅ Proxy bridge started successfully on port {}", bridge_port);
        Ok(bridge_port)
    }
}

// ✅ 在 Drop 时确保清理资源
impl Drop for ChromeLauncher {
    fn drop(&mut self) {
        if let Some(handle) = self.proxy_bridge_handle.take() {
            handle.abort();
        }
    }
}
```

**改进方案的关键点**：

1. **保存 JoinHandle**：将 `tokio::spawn` 返回的 handle 存储在结构体中
2. **主动停止**：在启动新任务前调用 `handle.abort()` 停止旧任务
3. **等待释放**：使用 `tokio::time::sleep` 给操作系统时间释放端口
4. **清理状态**：停止任务后清除相关状态（port 记录）
5. **Drop 保证**：实现 Drop trait 确保对象销毁时清理资源

**进一步优化**：使用 channel 获取绑定结果

```rust
async fn start_proxy_bridge(&mut self, upstream_proxy: &str) -> Result<u16> {
    let bridge_port = 8888 + (self.chrome_port - 9222);
    let listen_addr = format!("127.0.0.1:{}", bridge_port);

    let config = ProxyBridgeConfig {
        listen_addr: listen_addr.parse()?,
        upstream_proxy: upstream_proxy.to_string(),
    };

    // ✅ 使用 channel 接收绑定结果
    let (tx, rx) = tokio::sync::oneshot::channel();

    let handle = tokio::spawn(async move {
        match start_proxy_bridge(config).await {
            Ok(_) => {
                let _ = tx.send(Ok(()));
                // 继续运行 proxy bridge...
            }
            Err(e) => {
                let _ = tx.send(Err(e));
            }
        }
    });

    self.proxy_bridge_handle = Some(handle);

    // ✅ 等待绑定结果，确保真正成功
    rx.await
        .map_err(|_| Error::Internal("Proxy bridge task died".into()))?
        .map_err(|e| Error::ProxyBridge(format!("Failed to bind: {}", e)))?;

    tracing::info!("✅ Proxy bridge started successfully on port {}", bridge_port);
    Ok(bridge_port)
}
```

Related Concepts:

- Tokio task management (`JoinHandle`, `spawn`, `abort`)
- Resource lifecycle (RAII pattern in async context)
- Port binding conflicts
- Fire-and-forget anti-pattern
- Error propagation in background tasks
- Graceful shutdown patterns

Meta-Question Mapping:

- **Meta-Question**: m07 (Concurrency Correctness)
- **Tech Category**: 250 (Tokio Runtime)
- **Cognitive Level**: L2 (Application - understanding when and how to manage task lifecycle)
- **Error Codes**: OS Error 98 (EADDRINUSE)

Learning Objectives:

1. Understand the fire-and-forget anti-pattern and its consequences
2. Learn how to properly manage tokio::spawn task lifecycles
3. Master resource cleanup in async contexts
4. Implement graceful shutdown for background tasks
5. Debug port binding conflicts in concurrent scenarios
6. Use channels for error propagation from background tasks

Real-World Scenarios:

- HTTP/proxy servers that need restart without service interruption
- Background workers that need cleanup on application shutdown
- Long-running async tasks that hold system resources
- Container/VM lifecycle management with network services
- Service discovery agents with health check endpoints

Attachments:

None
