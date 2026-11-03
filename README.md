# qmc-decoder（fork）

本仓库是 [ownlight6/qmc-decoder](https://github.com/ownlight6/qmc-decoder) 的 fork。
**以下只列出与原版的差异**；功能介绍、使用方法、截图等其余内容见[原仓库](https://github.com/ownlight6/qmc-decoder)。

## 与原版的差异

### 1. 支持 STag 尾部（原版不支持）

原版 `detect_footer` 不认识新版 QQ 音乐客户端写入的 34 字节 STag 尾部
（仅含 `song_id` + `media_mid`，**不内嵌 ekey**），这类文件会报
`Cannot auto-fetch ekey: file does not have a musicEx footer`。

- `src/lib.rs`：新增 `FooterInfo::Stag` 变体；`detect_footer` 识别 STag；`decrypt_file` 新增 STag 分支，按尾部 meta 中的 size 正确裁尾后再解密
- `src-tauri/src/commands.rs`：文件信息展示补上 STag 分支（尾部类型、歌曲 ID、媒体 MID）

### 2. STag 文件的 ekey 自动获取（原版拿不到）

STag 文件本身不含 songmid，无法直接调 GetEVkey。

- `src/ekey_fetch.rs`：新增 `parse_stag_footer` 与 `resolve_stag_via_local_library` —— 在本机 QQ 音乐 `iQmc` 目录里按 song_id / media_mid 找到同曲的 musicex 副本，换回 API 所需的 songmid 与 filename
- 新增错误类型 `EkeyFetchError::StagUnresolved`（本机找不到副本时给出明确提示）

### 3. GetEVkey 前缀轮换 + 解密验证（原版固定单一 filename）

同一 `media_mid` 下，GetEVkey 会因 filename 前缀（`O4M0`/`O8M0`/`F0M0` 等质量档）不同返回**不同 ekey**。
原版固定用本地副本的前缀，若与待解文件码率档不一致就会解出乱码。

- `src/ekey_fetch.rs` 的 `fetch_ekey`：按候选前缀依次请求，用**解密前 4 字节的音频 magic**（fLaC/OggS/ID3/RIFF/MPEG sync）验证通过才采纳；全部不通过时降级返回首个 ekey（保持原版行为）

### 4. 解密输出校验（原版会静默写出乱码文件）

- `src/lib.rs`：`decrypt_file` 写盘前校验输出是否为合法音频；ekey 与密文不匹配时（典型场景：QQ 音乐服务端媒体轮换，旧媒体的 ekey 已从 API 消失）返回带根因说明的错误，而不是写出乱码
- 新增公开函数 `looks_like_audio` / `hex_head`

### 5. 新增命令行示例

- `examples/decrypt.rs`：`decrypt <input> <output> <ekey|--auto>`

### 6. 诊断开关

- 环境变量 `QMC_DEBUG_API=1` 时打印 GetEVkey 原始响应，便于排查 ekey 问题

## 测试范围

- **仅测试了 QQ 音乐加密文件**（QMC2 的 mflac / mflac0 / mflac1 / mgg2，含 STag 与 musicex 尾部）；未测试其他平台格式（如 NCM、KGM），也未覆盖 QMC1 各扩展名的批量实测
- **仅在 macOS 27.2（Apple Silicon）上测试**；Windows 端未测试
- `cargo test` 21 项通过；批量解密实测 134/134 成功（130 flac + 4 ogg，全部输出 magic 校验有效）

## 同步状态

- 改动分支 `feat/stag-footer`，已向上游提交 PR：[ownlight6/qmc-decoder#6](https://github.com/ownlight6/qmc-decoder/pull/6)
