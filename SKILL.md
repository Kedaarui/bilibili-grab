---
name: bilibili-grab
description: Download Bilibili videos and series at full premium quality (1080P) using yt-dlp + ffmpeg with cookie-based authentication. Handles single videos, multi-part series (分P), and season collections. Includes workarounds for B站 412 errors, Chrome/Edge v127+ app-bound encryption, and yt-dlp auto-merge failures.
---

# bilibili-grab

Battle-tested workflow for downloading Bilibili videos on Windows. Built on `yt-dlp` + `ffmpeg`.

## 你需要 1080P 以上画质吗？

**不需要** → 直接跳到[下载命令](#download-commands)，无需额外配置，会自动选可用最高画质。

**需要**（1080P 60fps / 1080P 高清） → 必须满足以下条件：

1. **B站大会员** — 非大会员账号无法下载 1080P 及以上格式
2. **浏览器 cookie** — 需要用 "Get cookies.txt LOCALLY" 扩展导出 cookie（见下方说明）
3. **每次下载前重新导出 cookie** — B站 cookie 有效期约 30 天，旧 cookie 会导致 1080P 不可用

### 获取 cookie（仅 1080P+ 需要）

Chrome/Edge v127+ 使用了 app-bound 加密，yt-dlp 自带的 `--cookies-from-browser` 无法解密。需要用浏览器扩展：

1. 在 Chrome 中登录 bilibili.com（确认是大会员账号）
2. 安装扩展 **"Get cookies.txt LOCALLY"**（Chrome Web Store）
3. 访问任意 bilibili.com 页面（保持登录状态）
4. 点击扩展图标 → Export → **Netscape** 格式
5. 保存到 `D:\Agent\bilibili_cookies.txt`

> cookie 文件包含有效登录会话，请勿提交到 git 或分享给他人。

## Prerequisites

### 1. Install yt-dlp + ffmpeg (Windows)

```bash
winget install yt-dlp.yt-dlp --accept-package-agreements --accept-source-agreements
winget install yt-dlp.FFmpeg --accept-package-agreements --accept-source-agreements
```

To use from any shell, prepend to PATH:
```bash
export PATH="/c/Users/$USER/AppData/Local/Microsoft/WinGet/Links:/c/Users/$USER/AppData/Local/Microsoft/WinGet/Packages/yt-dlp.FFmpeg_Microsoft.Winget.Source_8wekyb3d8bbwe/ffmpeg-N-124279-g0f6ba39122-win64-gpl/bin:$PATH"
```

(For a permanent fix, add both to Windows System PATH via `sysdm.cpl` → Environment Variables.)

### 2. Cookie file（仅 1080P+ 需要）

参考上方「获取 cookie」说明。不需要 1080P 的可以跳过此步。

## Download Commands

> 所有命令都需要 PATH 配置和（如果需要 1080P）有效的 cookie 文件。

### Single video

```bash
yt-dlp \
  --extractor-args "bilibili:use_api=true" \
  --user-agent "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36" \
  --cookies "D:/Agent/bilibili_cookies.txt" \
  -S "res:1080,ext:mp4:m4a" \
  --merge-output-format mp4 \
  --concurrent-fragments 4 \
  --retries 5 \
  -o "D:/Agent/%(title)s [%(id)s].%(ext)s" \
  "https://www.bilibili.com/video/BVxxxx"
```

如果 cookie 有效且是大会员，`-S "res:1080"` 会自动选 1080P。如果要强制指定编码：

- `-f "30080+30280"` — 1080P H.264 + 音频（最兼容）
- `-f "30077+30280"` — 1080P HEVC + 音频（文件更小）
- `-f "100026+30280"` — 1080P AV1 + 音频（最小文件）

### Full series / season collection

```bash
yt-dlp \
  --extractor-args "bilibili:use_api=true" \
  --user-agent "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36" \
  --cookies "D:/Agent/bilibili_cookies.txt" \
  -S "res:1080,ext:mp4:m4a" \
  --merge-output-format mp4 \
  --concurrent-fragments 4 \
  --retries 5 \
  -o "D:/Agent/%(playlist_index)02d - %(title)s [%(id)s].%(ext)s" \
  "https://space.bilibili.com/UID/lists/SID?type=season"
```

### Multi-part video (分P within one BV)

Same as single video — yt-dlp auto-detects `anthology` and iterates. The `%(playlist_index)` template also works for 分P.

## Critical Flags Explained

| Flag | Why |
|------|-----|
| `--extractor-args "bilibili:use_api=true"` | Forces API-based metadata extraction, bypasses some 412 errors |
| `--user-agent "..."` | B站 returns 412 (Precondition Failed) for default Python UA |
| `--cookies <file>` | Premium-only formats (1080P) require authenticated session |
| `-S "res:1080,ext:mp4:m4a"` | Sort selector: prefer 1080P, then mp4/m4a containers for max compatibility |
| `--merge-output-format mp4` | Output container; ffmpeg merges separate video/audio streams |
| `--concurrent-fragments 4` | Parallel download (4× faster on B站 CDN) |
| `--retries 5` | B站 CDN is flaky; auto-retry is essential |
| `--no-overwrites` | Optional: skip files already on disk (for re-runs) |
| `--download-archive <file>` | Optional: persist download history, skip already-downloaded items |

## Post-Download: Fix Broken Merges

yt-dlp occasionally fails to auto-merge and leaves `.f<vid>.mp4` + `.f<aid>.m4a` files. Fix manually with ffmpeg:

```bash
cd /d/Agent
ffmpeg -y -i "VIDEO.f30080.mp4" -i "VIDEO.f30280.m4a" \
  -c copy -map 0:v:0 -map 1:a:0 "VIDEO.mp4"
rm "VIDEO.f30080.mp4" "VIDEO.f30280.m4a"
```

## Troubleshooting

### `HTTP Error 412: Precondition Failed`
Add the `--user-agent` flag. B站 rejects default Python UA.

### `Could not copy Chrome cookie database` (issue #7271)
Chrome/Edge v127+ app-bound encryption. Use the **Get cookies.txt LOCALLY** extension instead of `--cookies-from-browser`.

### `Formats 1080P 高清, 720P 准高清 are missing; you have to become a premium member`
Cookies missing, expired, or account is not premium. Re-export cookies or downgrade expectation (remove `-S "res:1080"`).

### `WARNING: ffmpeg is not installed`
ffmpeg not in PATH. Install via winget (see Prerequisites) and export PATH.

### Download stalls at high percentage with `Read timed out`
B站 CDN throttles. `--retries 5` handles it automatically — just let it finish.

### File is only 480P despite premium cookies
Some videos are uploaded at 480P max. Check `-F` output for available formats before downloading.

## Verified Working Stack

- yt-dlp 2026.03.17
- ffmpeg N-124279 (from yt-dlp.FFmpeg winget package)
- Windows 10/11
- B站 premium account
- Chrome 131+ (with Get cookies.txt LOCALLY extension)
