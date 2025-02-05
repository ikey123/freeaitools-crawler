# 使用 curl 调用本地 API 的示例

## 1. 基本信息

在本地运行的 API 服务器地址为 `http://127.0.0.1:8040`。您可以使用 curl 命令向该服务器发送 POST 请求，以抓取指定网站的信息。

## 2. curl 命令示例

### 2.1 Windows 命令行版本

```bash
curl -X POST ^
  -H "Content-Type: application/json" ^
  -H "Authorization: Bearer 4487f197tap4ai8Zh42Ufi6mAHWGdy" ^
  -d "{\"url\": \"https://tap4.ai\", \"tags\": [\"ai-detector\", \"chatbot\", \"text-writing\", \"image\", \"code-it\"]}" ^
  http://127.0.0.1:8040/site/crawl
```

### 2.2 PowerShell 版本

```powershell
Invoke-RestMethod `
  -Method Post `
  -Uri "http://127.0.0.1:8040/site/crawl" `
  -Headers @{
    "Content-Type" = "application/json"
    "Authorization" = "Bearer 4487f197tap4ai8Zh42Ufi6mAHWGdy"
  } `
  -Body '{"url": "https://tap4.ai", "tags": ["ai-detector", "chatbot", "text-writing", "image", "code-it"]}'
```

### 2.3 一行版本的 curl 命令

```bash
curl -X POST -H "Content-Type: application/json" -H "Authorization: Bearer 4487f197tap4ai8Zh42Ufi6mAHWGdy" -d "{\"url\": \"https://tap4.ai\", \"tags\": [\"ai-detector\", \"chatbot\", \"text-writing\", \"image\", \"code-it\"]}" http://127.0.0.1:8040/site/crawl
```

## 3. 注意事项

- 确保 API 服务器正在运行（`http://127.0.0.1:8040`）。
- 确保 Authorization token 是正确的。
- 确保端口号 8040 没有被其他程序占用。

## 4. 错误处理

如果您在使用 curl 命令时遇到以下错误：

- `curl: (3) URL rejected: Bad hostname`
- `curl: (3) URL rejected: Port number was not a decimal number between 0 and 65535`
- `curl: (3) bad range specification in URL position 2`

请检查以下内容：

1. 确保 URL 格式正确。
2. 确保没有多余的字符或空格。
3. 确保端口号在有效范围内（0-65535）。
