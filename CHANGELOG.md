# 更新日志

## v1.1.0 (2024-xx-xx)

### 新增功能

#### 1. 新增负载均衡策略

`primary_sort_key` 现支持5种策略：

| 值 | 说明 |
|---|---|
| `upload_speed` | 上传速度优先（默认，速度低的优先） |
| `download_speed` | 下载速度优先（速度低的优先） |
| `active_downloads` | 活跃下载数优先（数量少的优先） |
| `free_space` | **新增** 剩余空间优先（空间大的优先） |
| `round_robin` | **新增** 轮流推送（按配置顺序循环分配） |

**配置示例：**
```json
{
    "primary_sort_key": "free_space"
}
```

#### 2. 新增故障转移功能

当种子添加到某个实例失败时，程序会自动尝试下一个可用实例，直到成功或所有实例都尝试过。

日志示例：
```
实例 qBittorrent-1 添加失败，尝试故障转移到下一个实例
```

#### 3. 新增状态更新间隔配置

新增 `status_update_interval` 参数，控制程序更新所有 qBittorrent 客户端状态的频率。

**适用场景：** 当您有大量客户端（如20+个）时，可以适当增加此间隔以减少 API 调用。

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `status_update_interval` | `30` | 状态更新间隔（秒），最小5秒 |

**配置示例：**
```json
{
    "status_update_interval": 30
}
```

### 架构优化

#### 快速汇报机制重构

将原来的单线程改为三个独立线程：

| 线程 | 功能 | 间隔 |
|------|------|------|
| 状态更新线程 | 更新所有客户端状态（速度、空间等） | `status_update_interval` |
| 快速汇报线程 | 只处理新添加的种子 | `fast_announce_interval` |
| 任务处理线程 | 分配待推送的种子 | 1秒 |

**优势：**
- 状态更新和快速汇报解耦，互不影响
- 大幅降低 API 调用频率
- 精准监控：只对新添加的种子进行快速汇报，不遍历所有种子

### 配置参数汇总

#### 完整配置示例

```json
{
    "qbittorrent_instances": [
        {
            "name": "qBittorrent-1",
            "url": "http://192.168.1.100:8080",
            "username": "admin",
            "password": "your_password",
            "traffic_check_url": "",
            "traffic_limit": 0,
            "reserved_space": 21504
        }
    ],
    "webhook_port": 5000,
    "webhook_path": "/webhook/secure-random-string",
    "max_new_tasks_per_instance": 2,
    "max_announce_retries": 30,
    "fast_announce_interval": 3,
    "status_update_interval": 30,
    "reconnect_interval": 120,
    "max_reconnect_attempts": 1,
    "connection_timeout": 6,
    "primary_sort_key": "upload_speed",
    "log_dir": "./logs",
    "debug_add_stopped": false,
    "fast_announce_enabled": false,
    "fast_announce_category_blacklist": []
}
```

#### 新增/修改的参数说明

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `primary_sort_key` | string | `upload_speed` | 负载均衡策略，新增 `free_space` 和 `round_robin` 选项 |
| `status_update_interval` | number | `30` | 状态更新间隔（秒），控制多久更新一次所有客户端状态 |

### 兼容性

- **Docker 部署**：无需修改 Dockerfile，直接使用原有镜像构建方式
- **本地运行**：无需安装新依赖，`requirements.txt` 未变更
- **配置文件**：新参数均有默认值，旧配置文件无需修改即可运行

### 升级指南

1. 替换 `main.py` 文件
2. （可选）更新 `config.json`，添加新参数
3. 重启服务

```bash
# Docker 部署
./docker-start.sh restart

# 本地运行
# 停止旧进程后重新启动
python run.py
```
