# Cloudflare R2 存储配置指南

## 问题描述
当图片上传到 Cloudflare R2 后无法通过 URL 直接访问，通常是因为存储桶的公共访问权限未开启。

## 1. 基础配置

### 1.1 开启公共访问
1. 登录 Cloudflare 控制台
2. 进入 R2 存储
3. 选择您的存储桶（例如：`toolsaifree`）
4. 点击 "设置"
5. 找到 "公共 URL 访问"，将其改为 "允许"

### 1.2 配置 CORS 策略
在存储桶设置中添加以下 CORS 配置：
```json
[
  {
    "AllowedOrigins": ["*"],
    "AllowedMethods": ["GET", "POST", "PUT", "DELETE", "HEAD"],
    "AllowedHeaders": ["*"],
    "ExposeHeaders": ["ETag"],
    "MaxAgeSeconds": 3600
  }
]
```

### 1.3 存储桶权限设置
添加以下权限策略：
```json
{
    "Version": "2012-10-17",  // IAM 策略的版本，通常使用 2012-10-17
    "Statement": [
        {
            "Sid": "PublicRead",  // 语句的唯一标识符，可以自定义
            "Effect": "Allow",  // 允许的效果，这里是允许
            "Principal": "*",  // 适用的主体，这里是所有用户（即公共访问）
            "Action": [
                "s3:GetObject"  // 允许的操作，这里是获取对象（即读取文件）
            ],
            "Resource": [
                "arn:aws:s3:::your-bucket-name/*"  // 资源 ARN，指定允许访问的存储桶和对象
            ]
        }
    ]
}
```

## 2. 自定义域名设置（可选但推荐）

### 2.1 添加自定义域名
1. 在存储桶设置中找到 "域" 设置
2. 添加您的自定义域名（例如：`img.yourdomain.com`）
3. 配置 DNS 记录

### 2.2 更新环境变量
在 `.env` 文件中添加或更新以下配置：
```bash
S3_CUSTOM_DOMAIN=img.yourdomain.com
```

## 3. URL 格式说明

### 3.1 默认 R2 URL 格式
```
https://[account-id].r2.cloudflarestorage.com/[bucket-name]/path/to/image.png
```

### 3.2 自定义域名 URL 格式
```
https://img.yourdomain.com/path/to/image.png
```

## 4. 验证步骤

### 4.1 测试图片访问
使用 curl 命令测试图片访问：
```bash
curl -I https://your-domain/path/to/image.png
```

### 4.2 检查响应头
正常响应应该包含：
```
HTTP/2 200 
content-type: image/png
content-length: [size]
```

## 5. 常见问题排查

### 5.1 图片访问 403 错误
- 检查存储桶的公共访问设置
- 验证 CORS 配置是否正确
- 确认权限策略已正确设置

### 5.2 图片访问 404 错误
- 确认图片路径是否正确
- 检查图片是否已成功上传
- 验证存储桶名称是否正确

### 5.3 跨域访问问题
- 检查 CORS 配置是否包含您的域名
- 确认 AllowedMethods 中包含需要的请求方法
- 验证 AllowedHeaders 配置是否正确

## 6. 安全建议
1. 限制允许的文件类型
2. 设置适当的文件大小限制
3. 考虑使用签名 URL 进行访问控制
4. 定期审查访问日志
5. 配置适当的缓存策略

## 7. 监控和维护
1. 定期检查存储使用情况
2. 监控访问流量和成本
3. 及时清理未使用的文件
4. 保持安全配置的更新

## 相关文档
- [Cloudflare R2 官方文档](https://developers.cloudflare.com/r2/)
- [R2 定价说明](https://www.cloudflare.com/products/r2/) 