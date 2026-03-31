# Fork 改动说明

这份文件不是为了记录“我改过什么”，而是为了保证以后再次同步上游新版本代码时，仍然可以按照这里的说明，重新做出同样的改动。

当前这个 fork 的核心目标有两个：

- 移除录音链路中的 VAD 功能
- 让 fork 可以在没有上游签名密钥的情况下，通过 GitHub Actions 产出可直接使用的 Windows `handy.exe`

如果以后我从作者仓库拉取了新版本代码，只要按这份文件逐项检查并重新修改，就应该能恢复同样的行为。

## 目标一：移除 VAD

### 修改目标

录音流程不要再依赖 Silero VAD 或 Smoothed VAD。录音器应当直接创建，不再把 VAD 挂到 `AudioRecorder` 上。

### 需要检查的文件

- `src-tauri/src/managers/audio.rs`

### 需要实现的结果

在 `create_audio_recorder(...)` 这类负责创建录音器的函数里，最终状态应该是：

- 不再导入 `SileroVad`
- 不再导入 `SmoothedVad`
- 不再创建 `SileroVad::new(...)`
- 不再创建 `SmoothedVad::new(...)`
- 不再调用 `.with_vad(...)`
- 保留音量/频谱回调，例如 `.with_level_callback(...)`
- 如果函数签名里仍保留 `vad_path`，可以把它改成 `_vad_path`，表示兼容保留但不使用

### 参考思路

目标是把类似这样的链路去掉：

```rust
let silero = SileroVad::new(...)?;
let smoothed_vad = SmoothedVad::new(...);

AudioRecorder::new()?
    .with_vad(Box::new(smoothed_vad))
    .with_level_callback(...)
```

改成类似这样：

```rust
AudioRecorder::new()?
    .with_level_callback(...)
```

### 修改完成后的预期行为

- 录音不再经过 VAD 门控
- 不再依赖 VAD 初始化是否成功
- 录音行为更直接，便于测试和持续维护

## 目标二：确保 fork 能持续产出 Windows `handy.exe`

这部分不是上游主功能修改，而是为了让 fork 在没有签名密钥的情况下，仍然可以稳定构建出 Windows 可执行文件。

### 需要检查的工作流文件

- `.github/workflows/build.yml`
- `.github/workflows/build-test.yml`
- `.github/workflows/main-build.yml`
- `.github/workflows/pr-test-build.yml`
- `.github/workflows/release.yml`

### 核心原则

以后如果上游更新了 workflow，这几条原则仍然要重新确认：

- 只保留 Windows x64 的构建流程
- 不依赖上游的签名密钥
- 构建完成后立刻上传产物
- 上传的产物里必须包含 `handy.exe`

## 工作流入口应该是什么样

上层 workflow 最终应当只调用 Windows x64 构建，而不是继续跑全平台矩阵。

### 目标状态

在这些入口 workflow 里，应当看到类似这样的参数：

```yml
with:
  platform: "windows-latest"
  target: "x86_64-pc-windows-msvc"
  build-args: ""
  sign-binaries: false
  upload-artifacts: true
```

### 检查重点

- 不要再保留 macOS / Linux / Windows ARM 的 matrix
- `sign-binaries` 必须为 `false`
- `upload-artifacts` 应当开启，至少在测试构建里开启

## build.yml 中必须保留的行为

### 1. unsigned Windows build 时移除 signCommand

即使 workflow 传了 `sign-binaries: false`，Tauri 配置文件里如果还保留：

```json
"windows": {
  "signCommand": "trusted-signing-cli ..."
}
```

那么构建时仍然会尝试执行签名命令。

因此在 Windows 且 `sign-binaries: false` 时，必须在构建前动态修改 `src-tauri/tauri.conf.json`，移除：

- `bundle.windows.signCommand`

## 2. unsigned Windows build 时关闭 updater artifacts

如果 `src-tauri/tauri.conf.json` 里保留：

```json
"createUpdaterArtifacts": true
```

那么 Tauri 在打完安装包后，仍可能继续执行 updater 相关签名流程，从而因为缺少 updater 私钥而失败。

因此在 Windows 且 `sign-binaries: false` 时，也必须在构建前把：

- `bundle.createUpdaterArtifacts`

改成：

```json
false
```

## 3. 上传步骤必须紧跟在构建步骤之后

Windows 的 artifact 上传，不要拖到 workflow 很后面再做。正确方向是：

1. `Build with Tauri`
2. 立刻解析产物路径
3. 立刻上传 Windows artifact

这样即使后面有别的平台步骤或者其他非关键步骤，也不会影响拿到 Windows 产物。

## 4. Windows artifact 必须包含 handy.exe

这里最重要。以后如果你再次重做 workflow，必须确保上传的不只是安装器，而是包含程序本体。

### 必须上传的内容

- `handy.exe`

### 可以顺便上传的内容

- `bundle/nsis/*.exe`
- `bundle/msi/*.msi`

### 原因

你需要的是可以直接替换当前使用中的程序本体，而不是安装器。

安装器：

- `setup.exe`
- `.msi`

都不是主要目标。

真正需要的是：

- `handy.exe`

这样在保留原有配置文件和使用环境的前提下，可以直接替换程序本体继续使用。

## 建议的核对清单

以后同步上游新版本代码后，按下面顺序检查：

1. 打开 `src-tauri/src/managers/audio.rs`
2. 确认录音器创建逻辑里没有 `SileroVad`
3. 确认录音器创建逻辑里没有 `SmoothedVad`
4. 确认没有 `.with_vad(...)`
5. 打开 `.github/workflows/build-test.yml`
6. 确认只构建 `windows-latest` + `x86_64-pc-windows-msvc`
7. 确认 `sign-binaries: false`
8. 打开 `.github/workflows/build.yml`
9. 确认 unsigned Windows build 会移除 `bundle.windows.signCommand`
10. 确认 unsigned Windows build 会关闭 `bundle.createUpdaterArtifacts`
11. 确认 `Build with Tauri` 后面立刻就是 Windows artifact 上传
12. 确认 artifact 上传路径里包含 `handy.exe`
13. 在 GitHub Actions 上手动跑一次 `Build Test`
14. 下载 artifact，确认里面确实有 `handy.exe`

## 这份文件的用途

以后如果我再次从作者仓库同步代码，这份文件就是“重复修改指南”。

判断标准不是代码是否和这次完全一样，而是最终是否满足这几个结果：

- VAD 已被移除
- Windows fork 构建不依赖签名密钥
- GitHub Actions 能产出 artifact
- artifact 中包含可直接替换使用的 `handy.exe`
