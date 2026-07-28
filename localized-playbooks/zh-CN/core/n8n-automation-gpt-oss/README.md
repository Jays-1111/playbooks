<!--
Copyright Advanced Micro Devices, Inc.

SPDX-License-Identifier: MIT
-->

<!-- @github-only -->
> [!IMPORTANT]
> 此 playbook 使用 GitHub 无法正确渲染的特殊标签。请访问 [amd.com/playbooks](https://amd.com/playbooks) 以正确预览内容。
<!-- @github-only:end -->

## 概览

n8n 是一个工作流自动化平台，可以让你通过可视化的节点编辑器连接应用和服务。

本 playbook 将教你如何搭建一个由 AI 驱动的财经新闻摘要器。它会抓取新华网财经频道，提取关键标题，并使用运行在本机系统上的本地 LLM 生成面向投资者的摘要。

## 你将学到什么

- 如何安装和启动 n8n
- 如何导入和配置预构建工作流
- 如何使用 n8n 原生集成连接 Lemonade
- 如何理解工作流节点和数据流

## 什么是 Lemonade？

[Lemonade](https://lemonade-server.ai) 是一个面向 AMD 硬件构建的本地 LLM 服务平台。它提供 OpenAI 兼容 API，并完全在你的本机运行，因此你的数据不会离开设备。

在本 playbook 中，我们使用 Lemonade 来服务一个本地 LLM，n8n 会连接到该模型来完成 AI 驱动的任务。

n8n 包含一个原生的 Lemonade 节点（`Lemonade Chat Model`），提供一等集成能力，不需要手动配置。这让你可以更直接地把本地 LLM 接入自动化工作流。

## 设置内存配置

<!-- @require:memory-config -->

## 安装软件前置条件

<!-- @os:windows -->
<!-- @require:lemonade,nodejs -->
<!-- @os:end -->

<!-- @device:halo -->
<!-- @var:id=lemonade_model value="Qwen3.5-35B-A3B" -->
<!-- @device:end -->

<!-- @test:id=lemonade-version timeout=60 hidden=True -->
```bash
lemonade --version
```
<!-- @test:end -->

<!-- @os:windows -->
<!-- @test:id=lemonade-chat-windows timeout=1200 hidden=True -->
```powershell
$ErrorActionPreference = "Stop"

# Wait for server to come up
$modelsJson = $null
for ($i=0; $i -lt 120; $i++) {
  $modelsJson = curl.exe -s --max-time 2 http://127.0.0.1:13305/api/v1/models
  if ($modelsJson) { break }
  Start-Sleep -Seconds 1
}
if (-not $modelsJson) { throw "Lemonade server not ready on http://127.0.0.1:13305" }
Write-Host "OK: Lemonade server is responding"

# Now that the server is responding, check if model is downloaded in Lemonade (robust JSON parse)
$parsed = $modelsJson | ConvertFrom-Json
$entry  = $parsed.data | Where-Object { $_.id -eq "${lemonade_model}" } | Select-Object -First 1
if (-not $entry) { throw "Model ${lemonade_model} is not present in Lemonade /api/v1/models." }
if (-not $entry.downloaded) { throw "Model ${lemonade_model} is present but not downloaded in Lemonade. Please download it." }
Write-Host "OK: ${lemonade_model} model is downloaded in Lemonade"

# Model chat test
$body = @{
  model = "${lemonade_model}"
  messages = @(@{ role = "user"; content = "Reply with exactly: OK" })
  temperature = 0
  max_tokens = 32
} | ConvertTo-Json -Depth 5

$tmpBody = Join-Path $env:TEMP "lemonade-chat-body.json"
[System.IO.File]::WriteAllText($tmpBody, $body, [System.Text.UTF8Encoding]::new($false))

try {
  $out = curl.exe -sS --fail-with-body --max-time 300 http://127.0.0.1:13305/api/v1/chat/completions `
  -H "Content-Type: application/json" `
  --data-binary "@$tmpBody"
  if (-not $out) { throw "Empty response from Lemonade chat/completions" }
}
finally {
  Remove-Item  $tmpBody -Force -ErrorAction SilentlyContinue
}
```
<!-- @test:end -->
<!-- @os:end -->

<!-- @test:id=node-npm-version timeout=60 hidden=True -->
```bash
node -v
npm -v
```
<!-- @test:end -->

## 安装 n8n

<!-- @os:windows -->
使用 npm 全局安装 n8n。

> **注意**：你可能会看到一些 npm 警告，这是正常现象。

```bash
npm install -g n8n
```

<!-- @test:id=n8n-version timeout=60 hidden=True -->
```bash
n8n --version
```
<!-- @test:end -->
<!-- @os:end -->

<!-- @os:windows -->
> **提示**：Windows 用户在运行某些 PowerShell 命令前，可能需要修改 PowerShell Execution Policy，例如设置为 `RemoteSigned` 或 `Unrestricted`。
<!-- @os:end -->

<!-- @os:windows -->
> **PATH 问题**：如果 `n8n --version` 显示 `command not found`，请确保 npm 全局 bin 目录在用户 `PATH` 中。常见路径是 `C:\Users\<username>\AppData\Roaming\npm`。可以将该路径添加到用户 Path（编辑系统环境变量 > 环境变量 > 编辑用户 Path），然后重新打开终端。
<!-- @os:end -->

<!-- @os:windows -->
## 启动 n8n

从终端启动 n8n：

```bash
n8n start
```

<!-- @test:id=n8n-start-windows timeout=300 hidden=True -->
```powershell
$N8N_CMD = "$env:APPDATA\npm\n8n.cmd"
$p = Start-Process -FilePath "cmd.exe" -ArgumentList "/c `"$N8N_CMD`" start" -NoNewWindow -PassThru
try {
  $ok = $false
  for ($i=0; $i -lt 120; $i++) {
    # Check HTTP status code only (body may be empty)
    $code = curl.exe -s -o NUL -w "%{http_code}" --max-time 2 http://127.0.0.1:5678/healthz
    if ($LASTEXITCODE -eq 0 -and $code -eq "200") { $ok = $true; break }
    Start-Sleep -Seconds 1
  }
  if (-not $ok) { throw "n8n not ready on http://127.0.0.1:5678/healthz" }
  Write-Host "OK: n8n server is responding"
} finally {
  # Kill the process actually listening on 5678
  $conn = Get-NetTCPConnection -LocalPort 5678 -State Listen -ErrorAction SilentlyContinue | Select-Object -First 1
  if ($conn) { Stop-Process -Id $conn.OwningProcess -Force -ErrorAction SilentlyContinue }
  # Also kill wrapper pid just in case
  if ($p -and -not $p.HasExited) { Stop-Process -Id $p.Id -Force -ErrorAction SilentlyContinue }
}
```
<!-- @test:end -->
<!-- @os:end -->

<!-- @os:windows -->
n8n 会启动一个本地 Web 服务器。按 `o`，或在浏览器中打开 `http://localhost:5678` 访问编辑器。
<!-- @os:end -->

> **提示**：使用 n8n 时请保持终端窗口打开。关闭终端可能会停止服务器。

## 启动 Lemonade

Lemonade 是将运行模型并连接到 n8n 的本地服务器。

<!-- @os:windows -->
打开 Lemonade GUI，或在 PowerShell 中运行：
<!-- @os:end -->

<!-- @device:halo -->
```bash
lemonade run ${lemonade_model} --llamacpp vulkan
```
<!-- @device:end -->

> **提示**：Lemonade 运行后，也可以通过 `http://localhost:13305` 访问 GUI。

你也可以运行 `lemonade list` 查看 Lemonade 设置中可用的模型。

### 可选：使用 ModelScope 下载并加载本地 GGUF 模型

如果通过 Lemonade 直接下载 `Qwen3.5-35B-A3B` 失败，可以先使用 ModelScope 下载对应的 GGUF 模型文件到本地，然后让 Lemonade 从本地目录加载模型。

安装 ModelScope：

```bash
pip install modelscope
```

下载 Qwen3.5-35B 的 GGUF 模型文件：

```bash
modelscope download --model unsloth/Qwen3.5-35B-A3B-GGUF Qwen3.5-35B-A3B-Q4_K_M.gguf --local_dir ./dir
```

将本地模型目录配置给 Lemonade。这里建议使用绝对路径：

```bash
lemonade config set extra_models_dir="<本地GGUF模型文件夹路径>"
```

查看并运行模型：

```bash
lemonade list
lemonade run Qwen3.5-35B-A3B-Q4_K_M --llamacpp vulkan
```

## 设置工作流

### 第 1 步：注册或登录 n8n

首次打开 n8n 时，系统会提示你创建账号或登录：

1. 在浏览器中打开 `http://localhost:5678`
2. 使用你的邮箱创建新的本地账号，或使用已有账号登录
3. 登录后，你会看到 n8n dashboard

> **提示**：如果账号被锁定，可以尝试 `n8n user-management:reset`。

### 第 2 步：导入工作流

我们提供了一个可以直接导入的预构建工作流：

1. 下载以下工作流文件：[financial-news-workflow.json](assets/financial-news-workflow.json)
2. 点击 **Start from Scratch** 打开工作流编辑器。或者点击左上角的 **+** 按钮，然后点击 **Add workflow**。
3. 点击右上角栏中的 **...** 菜单，也就是三个点，选择 **Import from file**。
4. 选择下载的 `financial-news-workflow.json` 文件。
5. 工作流会显示在画布上。

### 第 3 步：理解工作流

导入的工作流包含 9 个相连节点：

<p align="center">
  <img src="assets/workflow-overview.png" alt="n8n 财经新闻工作流" width="800"/>
</p>

| 节点 | 用途 |
|------|------|
| **When clicking 'Execute workflow'** | 手动触发以启动工作流。 |
| **Fetch Financial News Webpage** | 向 `https://www.news.cn/fortune/index.htm` 发送 HTTP GET 请求。 |
| **Delay to Ensure Page Load** | 等待节点，确保页面内容已完全加载。 |
| **Extract News Headlines & Text** | HTML 节点，使用 CSS 选择器提取财经新闻标题。 |
| **Clean Extracted News Data** | Set 节点，将所有提取的数据合并到单个文本字段中。 |
| **AI Financial News Summarizer** | AI Agent，使用金融分析师 system prompt 处理新闻。 |
| **Lemonade Chat Model** | 连接到运行 LLM 的本地 Lemonade server。 |
| **Structured Output Parser** | 将 AI 输出格式化为结构化 JSON。 |
| **Convert to File** | 将摘要转换为可下载文件。 |

### 第 4 步：配置 Lemonade 凭据

运行工作流之前，你需要将它连接到本地 Lemonade server：

1. 在 n8n 中双击 **Lemonade Chat Model** 节点。
2. 在 **Credential to connect with** 下拉菜单中选择 **Create New Credential**。
3. 输入下面的值，然后点击保存。
4. 选择你已经在 Lemonade Server 中加载的相关模型。

| 字段 | 值 |
|------|----|
| **Base URL** | `http://localhost:13305/api/v1` |
| **API Key** | `lemonade` |

> **注意**：测试前，请在终端中运行 `lemonade status`，确认 Lemonade server 正在运行。

### 第 5 步：测试工作流

1. 确保 Lemonade 正在运行并且模型已加载。
2. 点击画布底部中间的 **Execute workflow**。
3. 观察每个节点从左到右执行，成功后它们会变成绿色。
4. 双击 **AI Financial News Summarizer** 节点，在底部面板查看生成的摘要。
5. 双击 **Convert to File** 节点，在底部面板下载对应的文本文件。

## 理解 AI Agent

AI 财经新闻摘要器使用一个为金融分析设计的系统提示词：

```text
你是一名 AI 财经分析师。你的职责是阅读、理解并总结今天的重要财经新闻。目标是为投资者提供清晰、简洁的市场概览，以支持更好的投资决策。

投资者展望
今天的新闻指向[偏多/偏空/中性]的市场情绪。请关注明天的[经济事件/财报]，这可能会影响市场方向。
```

Agent 接收清洗后的新闻数据，并输出带有市场情绪的结构化摘要。

### 保存你的工作流

点击顶部的工作流名称，并按需要重命名。工作流会在你操作时自动保存。

## 下一步

- **定时自动化**：将 Manual Trigger 替换为 **Schedule Trigger**，以便每天运行。
- **发送通知**：添加 **Email** 节点来接收摘要。
- **尝试不同模型**：在 Lemonade Chat Model 节点中更改模型，以实验不同 LLM。
- **自定义提取**：修改 HTML Extract 节点的 CSS 选择器，以定位不同新闻版块。
- **尝试不同后端**：n8n 也支持 [Ollama](https://n8n.io/workflows/?integrations=Ollama+Chat+Model)、LM Studio 和其他本地 LLM 后端。

### 探索 n8n 模板

n8n 有数百个预构建工作流模板。可以在官方模板库浏览：

**[https://n8n.io/workflows/](https://n8n.io/workflows/)**

搜索 “AI”、“LLM” 或 “automation” 来查找可以导入和自定义的工作流。

如需更多信息，请查看 [n8n 文档](https://docs.n8n.io/)。

<!-- @os:windows -->
<!-- @test:id=lemonade-unload-windows timeout=60 hidden=True -->
```powershell
# CI cleanup: unload the model so the GPU pool is free
lemonade unload
exit 0
```
<!-- @test:end -->
<!-- @os:end -->
