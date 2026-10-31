# ChaoInput（吵吵拼音）

自训 AI 拼音输入法 · Windows TSF · 本地推理 · 28.7M 参数

ChaoInput 是一个从零实现的 Windows 拼音输入法：TSF 输入法框架 + 常驻主进程 + 拼音引擎 + CLMG v2 上下文重排模型。模型在本地 CPU 推理，不联网、不做生成，只对词典候选重排。

> 演示视频见 B 站《自训开源 AI 拼音输入法》。

## 亮点

- **真 TSF 输入法**：Chrome / Office / 记事本 / 微信等任意 TSF 宿主；x64 + x86 双 DLL，32 位 Office 也能用
- **本地 AI 重排**：6 层 Transformer，28,730,368 参数，上下文 64 字，权重 115 MB，CPU 推理
- **只重排不生成**：候选来自 15 万条词典，不会出现词典外的怪词；模型失效自动退回词频序
- **低延迟**：按键经命名管道同步 IPC 送到主进程，一次往返 < 2 ms
- **绿色便携**：单个目录，词库 / 配置 / 模型都在 exe 旁 `data\`，不写 AppData

## 效果

| 口径 | 第一候选命中率 |
|---|---|
| 独立验证集 4,277 行（ctx 64） | **92.2%** |
| 同上，ctx 32 基线 | 90.4% |
| 真实用户日志 2,638 条 | **87.2%**（纯词频 85.2%） |

口径说明：候选池内重排 top-1，不等于整句输入准确率；验证集与训练集按行随机切分。

## 架构

```
宿主应用（Chrome / Office / 记事本 / 微信 …）
        │  TSF
ChaoTsf.dll (x64)   ChaoTsf32.dll (x86)
        │  命名管道：按键帧 / 位置帧 / 提交帧（同步一问一答）
ChaoInput.exe  ──  按键路由 · 候选窗 · 词库 · 用户学习 · 语音输入
        └─  拼音引擎（DP 音节切分 + 候选池）
            └─  CLMG v2 重排（本地 CPU，6 层 / 512 维）
```

- **DLL 不含 AI**：只做 TSF 接口与按键转发；模型、词典、候选窗都在主进程，宿主崩溃不影响引擎。
- **同步 IPC**：按键帧等待主进程回复（含上屏文本），避免异步提交在 Word 等宿主上的竞态。
- **语境**：主进程维护每个宿主的最近 64 字上下文，焦点丢失 / 换文档 / 宿主终止时断开。

## 目录结构

```
src/ChaoInput/                 主程序：输入引擎、候选窗、IPC 服务端、词库（VC++ Win32）
src/ChaoInput.Tsf/             TSF 输入法 DLL（x64 + x86 双架构）
```

运行所需的 `dict.db` 与 `lmt.bin` 随 Releases 运行包提供，不放在源码仓库。

## 快速开始（使用）

1. 到 **Releases** 下载 `ChaoInput-v3.3.15-win64.zip`，解压到任意目录（路径可含中文）。
2. **右键 `ChaoInput.exe` → 以管理员身份运行**（首次注册 TSF 需要写 HKLM；之后普通启动即可）。
3. 任务栏出现悬浮窗后，在任意输入框打拼音：
   - `空格` / `回车` 上屏第一候选；`1`–`6` 选词；`-` `=` 或 `PgUp` / `PgDn` 翻页
   - 轻按 `Shift` 或 `CapsLock` 切换中 / 英
   - `Esc` 取消组合；退格删拼音
4. 退出：悬浮窗右键退出。程序退出时会注销 TSF，删除目录即可卸载。

> 系统要求：Windows 10 / 11 x64。词典与模型都在 `data\`，不依赖 .NET / VC 运行时（`/MT` 静态链接）。

## 构建

需要 Visual Studio 2022（C++ 桌面开发）+ Windows SDK，使用 **PowerShell 7**：

```powershell
# 先构建 DLL（exe 启动时要按目录注册它们）
pwsh -NoProfile -File src\ChaoInput.Tsf\build.ps1
# 再构建主程序
pwsh -NoProfile -File src\ChaoInput\build.ps1
```

产物在 `src\ChaoInput\bin\Release\`（exe）与 `src\ChaoInput.Tsf\bin\Release\`（DLL）。
运行所需文件：`ChaoInput.exe`、`ChaoTsf.dll`、`ChaoTsf32.dll`、`appsettings.json`、`data\dict.db`、`data\lmt.bin`。

## 模型

发布包内含训练好的 CLMG v2 权重 `data/lmt.bin`：6 层 Transformer、512 维、8 头、上下文 64 字，共 28,730,368 参数，CPU 推理。

本仓库只包含输入法运行时源码；PyTorch 训练脚本与语料预处理流水线未随仓库发布。

## 配置

`appsettings.json`：

```json
{
  "Asr": { "Endpoint": "", "ApiKey": "", "Model": "Qwen3-ASR-1.7B", "SampleRate": 16000, "TimeoutSeconds": 60 },
  "Model": "lmt",
  "Log": false
}
```

- `Model`：`lmt` = CLMG v2 重排（默认）；`lm` = 旧线性模型（可选，缺 `lm.bin` 时退化为纯词频）。
- `Asr`：可选语音输入，指向任意 OpenAI 兼容 `/v1/chat/completions` 的 ASR 服务；留空则语音按钮不可用。
- `Log`：调试日志开关（默认关闭）。

## 已知限制

- 仅 Windows 10/11；TSF 需按宿主位数注册，仓库已提供 x64/x86 双 DLL。
- 词典 15 万条，未登录词不在候选池；专业术语可通过「＋」手动加词。
- 公开包仅清除输入日志（`userlog` 行），词库保留作者积累的常用词（含少量个人词）。
- 语音输入需要自备 ASR 服务，不随包提供模型。

## 许可

代码以 MIT 许可发布（见 `LICENSE`）。词典与训练语料可能包含第三方数据，二次分发前请自行确认来源许可。
