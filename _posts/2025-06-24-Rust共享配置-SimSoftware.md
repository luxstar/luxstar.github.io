---
title: Rust共享配置-SimSoftware
date: 2025-06-24 05:49:00 +0800
categories: [Rust]
tags: [Rust,Web]
---

在 Rust 中，你需要在多个线程之间共享和同步配置数据。最常用的解决方案是使用 `Arc<RwLock<T>>` 或 `Arc<Mutex<T>>` 来实现线程安全的配置共享。

```rust
use std::sync::{Arc, RwLock};
use std::time::Duration;
use tokio::time::sleep;
use serde::{Deserialize, Serialize};
use warp::Filter;

// 配置结构体
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Config {
    pub interval_ms: u64,
    pub message: String,
    pub multiplier: f64,
    pub enabled: bool,
}

impl Default for Config {
    fn default() -> Self {
        Self {
            interval_ms: 1000,
            message: "Default message".to_string(),
            multiplier: 1.0,
            enabled: true,
        }
    }
}

// 共享配置类型
type SharedConfig = Arc<RwLock<Config>>;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 创建共享配置
    let config = SharedConfig::new(RwLock::new(Config::default()));
    
    // 启动数据输出任务
    let data_config = Arc::clone(&config);
    let data_task = tokio::spawn(async move {
        data_output_loop(data_config).await;
    });
    
    // 启动 HTTP 服务器
    let server_config = Arc::clone(&config);
    let server_task = tokio::spawn(async move {
        start_http_server(server_config).await;
    });
    
    // 等待任务完成
    tokio::try_join!(data_task, server_task)?;
    
    Ok(())
}

// 数据输出循环
async fn data_output_loop(config: SharedConfig) {
    let mut counter = 0u64;
    
    loop {
        // 读取当前配置
        let current_config = {
            let cfg = config.read().unwrap();
            cfg.clone()
        };
        
        // 如果启用，则输出数据
        if current_config.enabled {
            let output_data = generate_data(&current_config, counter);
            println!("[{}] {}", 
                chrono::Utc::now().format("%Y-%m-%d %H:%M:%S%.3f"),
                output_data
            );
        }
        
        // 根据配置的间隔时间休眠
        sleep(Duration::from_millis(current_config.interval_ms)).await;
        counter += 1;
    }
}

// 根据配置生成数据
fn generate_data(config: &Config, counter: u64) -> String {
    let value = (counter as f64) * config.multiplier;
    format!("{} - Counter: {}, Value: {:.2}", config.message, counter, value)
}

// 启动 HTTP 服务器
async fn start_http_server(config: SharedConfig) {
    // GET /config - 获取当前配置
    let get_config = warp::path("config")
        .and(warp::get())
        .and(with_config(Arc::clone(&config)))
        .and_then(get_config_handler);
    
    // POST /config - 更新配置
    let update_config = warp::path("config")
        .and(warp::post())
        .and(warp::body::json())
        .and(with_config(Arc::clone(&config)))
        .and_then(update_config_handler);
    
    // PUT /config/field - 更新单个字段
    let update_field = warp::path!("config" / String)
        .and(warp::put())
        .and(warp::body::json())
        .and(with_config(Arc::clone(&config)))
        .and_then(update_field_handler);
    
    let routes = get_config
        .or(update_config)
        .or(update_field)
        .with(warp::cors().allow_any_origin());
    
    println!("HTTP server starting on http://0.0.0.0:3030");
    println!("Available endpoints:");
    println!("  GET  /config          - 获取当前配置");
    println!("  POST /config          - 更新完整配置");
    println!("  PUT  /config/interval - 更新间隔时间");
    println!("  PUT  /config/message  - 更新消息内容");
    println!("  PUT  /config/enabled  - 启用/禁用输出");
    
    warp::serve(routes)
        .run(([0, 0, 0, 0], 3030))
        .await;
}

// Warp filter 辅助函数
fn with_config(config: SharedConfig) -> impl Filter<Extract = (SharedConfig,), Error = std::convert::Infallible> + Clone {
    warp::any().map(move || Arc::clone(&config))
}

// 获取配置的处理函数
async fn get_config_handler(config: SharedConfig) -> Result<impl warp::Reply, warp::Rejection> {
    let cfg = config.read().unwrap();
    Ok(warp::reply::json(&*cfg))
}

// 更新完整配置的处理函数
async fn update_config_handler(
    new_config: Config,
    config: SharedConfig,
) -> Result<impl warp::Reply, warp::Rejection> {
    {
        let mut cfg = config.write().unwrap();
        *cfg = new_config;
        println!("Configuration updated: {:?}", *cfg);
    }
    
    Ok(warp::reply::json(&serde_json::json!({
        "status": "success",
        "message": "Configuration updated successfully"
    })))
}

// 更新单个字段的处理函数
async fn update_field_handler(
    field: String,
    value: serde_json::Value,
    config: SharedConfig,
) -> Result<impl warp::Reply, warp::Rejection> {
    {
        let mut cfg = config.write().unwrap();
        
        match field.as_str() {
            "interval" => {
                if let Some(interval) = value.as_u64() {
                    cfg.interval_ms = interval;
                } else {
                    return Ok(warp::reply::with_status(
                        warp::reply::json(&serde_json::json!({
                            "error": "Invalid interval value"
                        })),
                        warp::http::StatusCode::BAD_REQUEST,
                    ));
                }
            }
            "message" => {
                if let Some(msg) = value.as_str() {
                    cfg.message = msg.to_string();
                } else {
                    return Ok(warp::reply::with_status(
                        warp::reply::json(&serde_json::json!({
                            "error": "Invalid message value"
                        })),
                        warp::http::StatusCode::BAD_REQUEST,
                    ));
                }
            }
            "multiplier" => {
                if let Some(mult) = value.as_f64() {
                    cfg.multiplier = mult;
                } else {
                    return Ok(warp::reply::with_status(
                        warp::reply::json(&serde_json::json!({
                            "error": "Invalid multiplier value"
                        })),
                        warp::http::StatusCode::BAD_REQUEST,
                    ));
                }
            }
            "enabled" => {
                if let Some(enabled) = value.as_bool() {
                    cfg.enabled = enabled;
                } else {
                    return Ok(warp::reply::with_status(
                        warp::reply::json(&serde_json::json!({
                            "error": "Invalid enabled value"
                        })),
                        warp::http::StatusCode::BAD_REQUEST,
                    ));
                }
            }
            _ => {
                return Ok(warp::reply::with_status(
                    warp::reply::json(&serde_json::json!({
                        "error": format!("Unknown field: {}", field)
                    })),
                    warp::http::StatusCode::BAD_REQUEST,
                ));
            }
        }
        
        println!("Field '{}' updated. New config: {:?}", field, *cfg);
    }
    
    Ok(warp::reply::with_status(
        warp::reply::json(&serde_json::json!({
            "status": "success",
            "message": format!("Field '{}' updated successfully", field)
        })),
        warp::http::StatusCode::OK,
    ))
}
```

以下是一个完整的解决方案示例：同时需要在 `Cargo.toml` 中添加依赖：## 核心解决方案说明：

```Toml
[package]
name = "realtime-config-service"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio = { version = "1.0", features = ["full"] }
warp = "0.3"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
chrono = { version = "0.4", features = ["serde"] }
```

### 1. **线程安全的配置共享**

- 使用 `Arc<RwLock<Config>>` 在多个异步任务之间共享配置
- `Arc` 允许多个所有者
- `RwLock` 允许多个读者或一个写者，提供读写锁机制

### 2. **实时数据输出**

- `data_output_loop` 函数持续运行，每次循环都读取最新配置
- 根据配置的 `interval_ms` 调整输出频率
- 配置更新后立即生效（下一次循环）

### 3. **HTTP API 端点**

- `GET /config` - 获取当前配置
- `POST /config` - 更新完整配置
- `PUT /config/{field}` - 更新单个字段

## 使用示例

```bash
# 获取当前配置
curl http://localhost:3030/config

# 更新完整配置
curl -X POST http://localhost:3030/config \
  -H "Content-Type: application/json" \
  -d '{"interval_ms": 500, "message": "Updated message", "multiplier": 2.5, "enabled": true}'

# 更新单个字段
curl -X PUT http://localhost:3030/config/interval \
  -H "Content-Type: application/json" \
  -d '2000'

# 暂停输出
curl -X PUT http://localhost:3030/config/enabled \
  -H "Content-Type: application/json" \
  -d 'false'
```

## 关键特性

1. **零延迟配置更新** - 配置更改立即生效
2. **线程安全** - 多个线程可以安全地读写配置
3. **灵活的API** - 支持完整更新和字段级更新
4. **非阻塞** - 数据输出不会阻塞HTTP请求处理
5. **类型安全** - 使用强类型确保配置的正确性

这个解决方案可以确保你的服务在接收到HTTP请求后，输出数据能够立即根据新配置进行调整。
