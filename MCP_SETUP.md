# RAGFlow MCP Server 使用说明

## 启动 MCP Server

```bash
cd /home/hbrens/Desktop/Project/coding/ragflow
python3 mcp/server/server.py --host=127.0.0.1 --port=9382 --base-url=http://127.0.0.1:59380 --api-key=<你的API_KEY>
```

### 参数说明

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `--host` | 监听地址 | 127.0.0.1 |
| `--port` | 监听端口 | 9382 |
| `--base-url` | RAGFlow 后端地址 | http://127.0.0.1:9380 |
| `--api-key` | RAGFlow API Key | 必填（self-host 模式） |
| `--mode` | 启动模式 | self-host |

### 获取 API Key

1. 打开 RAGFlow 界面 `http://127.0.0.1:50080`
2. 点击右上角头像
3. 点击 **API** 选项
4. 复制 API Key

## OpenCode 配置

编辑 `~/.config/opencode/opencode.json`，添加 `mcp` 字段：

```json
{
  "mcp": {
    "ragflow": {
      "type": "remote",
      "url": "http://127.0.0.1:9382/mcp",
      "headers": {
        "Authorization": "Bearer <你的API_KEY>"
      }
    }
  }
}
```

## 验证

启动后访问 `http://127.0.0.1:9382/mcp` 应返回 MCP 相关信息。
