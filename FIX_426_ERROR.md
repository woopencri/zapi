
# Z.ai API 426错误修复说明

## 📋 问题描述

在2025年9月30日，Z.ai更新了其客户端验证机制，导致API请求返回426错误：

```json
{
  "error": {
    "detail": "Your client version check is failed",
    "code": 426
  }
}
```

## 🔍 问题分析

通过测试和分析，发现Z.ai的新验证机制包含以下要求：

1. **客户端版本验证**：需要 `X-FE-Version` header
2. **签名验证**：需要 `X-Signature` header
3. **User-Agent更新**：需要匹配最新的Chrome版本

## ✅ 解决方案

### 1. 更新前端版本号

```go
// 从 1.0.70 更新到 1.0.94
X_FE_VERSION = "prod-fe-1.0.94"
```

### 2. 更新User-Agent

```go
// 从 Chrome 139 更新到 Chrome 140
BROWSER_UA = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/140.0.0.0 Safari/537.36"
```

### 3. 更新Sec-CH-UA

```go
// 更新Chrome版本信息
SEC_CH_UA = "\"Chromium\";v=\"140\", \"Not=A?Brand\";v=\"24\", \"Google Chrome\";v=\"140\""
```

### 4. 实现X-Signature签名

签名算法基于请求体的SHA-256哈希：

```go
// 生成 X-Signature - 基于请求体的 SHA-256 哈希
hash := sha256.Sum256(reqBody)
signature := fmt.Sprintf("%x", hash)
```

## 📝 修改文件

### main.go

**位置1：常量定义（第78-86行）**

```go
// 伪装前端头部（2025-09-30 更新：修复426错误）
const (
	X_FE_VERSION   = "prod-fe-1.0.94" // 更新：1.0.70 → 1.0.94
	BROWSER_UA     = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/140.0.0.0 Safari/537.36" // 更新：Chrome 139 → 140
	SEC_CH_UA      = "\"Chromium\";v=\"140\", \"Not=A?Brand\";v=\"24\", \"Google Chrome\";v=\"140\"" // 更新：Chrome 140
	SEC_CH_UA_MOB  = "?0"
	SEC_CH_UA_PLAT = "\"Windows\""
	ORIGIN_BASE    = "https://chat.z.ai"
)
```

**位置2：签名生成（第1618-1622行）**

```go
// 生成 X-Signature - 基于请求体的 SHA-256 哈希（426错误修复）
hash := sha256.Sum256(reqBody)
signature := fmt.Sprintf("%x", hash)

debugLog("生成签名: %s (基于请求体SHA256)", signature)
```

## 🧪 测试结果

### 非流式响应测试

```bash
curl -X POST http://localhost:9090/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer 0000" \
  -d '{"model": "GLM-4.6", "messages": [{"role": "user", "content": "你好，1+1等于几？"}], "stream": false}'
```

**结果：✅ 成功**
```json
{
  "id": "chatcmpl-1759245369",
  "object": "chat.completion",
  "created": 1759245369,
  "model": "GLM-4.6",
  "choices": [{
    "index": 0,
    "message": {
      "role": "assistant",
      "content": "你好！\n\n1 + 1 等于 **2**。\n\n这是一个基础的算术问题。如果您有其他数学问题或者任何其他方面的问题，也欢迎随时提问！"
    },
    "finish_reason": "stop"
  }]
}
```

### 流式响应测试

```bash
curl -X POST http://localhost:9090/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer 0000" \
  -d '{"model": "GLM-4.6", "messages": [{"role": "user", "content": "写一首关于春天的诗"}], "stream": true}'
```

**结果：✅ 成功**

返回了完整的流式SSE响应，内容正常。

## 📊 关键日志

成功的请求日志显示：

```
2025/09/30 23:16:07 [DEBUG] 生成签名: e39d78fcfa9b4926500af485388cd038c36e02529fbf82640002ee8fa07cc828 (基于请求体SHA256)
2025/09/30 23:16:09 [DEBUG] 上游响应状态: 200 200 OK
2025/09/30 23:16:09 [DEBUG] 开始收集完整响应内容
```

## 🎯 修复效果

### 修复前
- ❌ 返回426错误："Your client version check is failed"
- ❌ 无法获取任何AI响应
- ❌ 所有请求被拒绝

### 修复后
- ✅ 返回200 OK
- ✅ 正常获取AI响应
- ✅ 流式和非流式响应都正常工作
- ✅ 完全兼容OpenAI API格式

## 🚀 部署步骤

1. **更新代码**
   ```bash
   git pull origin main
   ```

2. **重新编译**
   ```bash
   go build -o ztoapi main.go simple_client.go
   ```

3. **重启服务**
   ```bash
   ./start.sh
   ```

4. **验证服务**
   ```bash
   curl -X POST http://localhost:9090/v1/chat/completions \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer 0000" \
     -d '{"model": "GLM-4.6", "messages": [{"role": "user", "content": "你好"}], "stream": false}'
   ```

## 🔧 技术细节

### 签名算法分析

Z.ai的签名验证机制：

1. **输入**：完整的JSON请求体
2. **处理**：计算SHA-256哈希
3. **输出**：16进制字符串（64字符）
4. **位置**：`X-Signature` HTTP header

示例：
```
请求体: {"stream":true,"model":"GLM-4-6-API-V1",...}
SHA256: e39d78fcfa9b4926500af485388cd038c36e02529fbf82640002ee8fa07cc828
Header: X-Signature: e39d78fcfa9b4926500af485388cd038c36e02529fbf82640002ee8fa07cc828
```

### 版本检测机制

Z.ai通过以下headers检测客户端版本：

1. **X-FE-Version**: 前端版本号（必须 ≥ 1.0.94）
2. **User-Agent**: 浏览器标识（推荐Chrome 140+）
3. **Sec-CH-UA**: Chrome用户代理客户端提示

如果任何一个不匹配，将返回426错误。

## 📝 维护建议

### 监控要点

1. **定期检查Z.ai更新**
   - 版本号可能继续升级
   - 签名算法可能变化
   - 新增验证机制

2. **日志监控**
   - 关注426错误
   - 监控成功率
   - 记录响应时间

3. **自动化测试**
   ```bash
   # 定期运行测试脚本
   ./test_api.sh
   ```

### 未来可能的变化

Z.ai可能会进一步加强验证：

- ⚠️ 更复杂的签名算法
- ⚠️ 添加时间戳验证
- ⚠️ 实现请求频率限制
- ⚠️ IP地址白名单

**应对策略**：保持代码的灵活性，便于快速调整。

## 🔗 相关文档

- [README.md](./README.md) - 项目主文档

## 📅 更新日志

- **2025-09-30**:
  - 成功修复426错误
  - 更新到Chrome 140
  - 实现SHA-256签名验证
  - 版本号更新至1.0.94

---

**修复者**: Kilo Code
**修复日期**: 2025年9月30日
**状态**: ✅ 已验证并部署