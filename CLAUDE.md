# tencent-weixin-openclaw-weixin

This directory contains downloaded and extracted npm packages for analysis. Packages are obtained via `npm pack` or direct registry download, then extracted locally.

## Directory Structure

```
.
├── CLAUDE.md                                    # This file
├── README.md                                    # Architecture analysis (v1.0.2)
├── docs/                                        # Additional documentation
│
├── tencent-weixin-openclaw-weixin-1.0.2/        # Extracted package (old version)
├── tencent-weixin-openclaw-weixin-1.0.2.tgz    # Original tarball
│
├── tencent-weixin-openclaw-weixin-2.1.1/        # Extracted package (current version)
├── tencent-weixin-openclaw-weixin-2.1.1.tgz    # Original tarball
│
├── tencent-weixin-openclaw-weixin-cli-1.0.2/   # CLI installer package
└── tencent-weixin-openclaw-weixin-cli-1.0.2.tgz
```

## Packages

### `@tencent-weixin/openclaw-weixin`

- **npm registry name**: `@tencent-weixin/openclaw-weixin`
- **Purpose**: OpenClaw framework channel plugin — adapts WeChat's private protocol to OpenClaw's standard HTTP JSON API
- **Versions present**: 1.0.2 (old), 2.1.1 (current)

The `2.1.1/` directory is the primary analysis target. Key files:
- `index.ts` — plugin entry point
- `src/channel.ts` — ChannelPlugin interface implementation
- `src/api/` — HTTP communication layer
- `src/auth/` — QR code login, multi-account management
- `src/messaging/` — inbound/outbound message handling
- `src/monitor/monitor.ts` — long-polling loop
- `src/cdn/` — AES-128-ECB encryption, CDN upload
- `src/media/` — media transcoding (SILK audio)
- `src/storage/` — sync cursor persistence
- `openclaw.plugin.json` — plugin metadata

### `@tencent-weixin/openclaw-weixin-cli`

- **npm registry name**: `@tencent-weixin/openclaw-weixin-cli`
- **Purpose**: Lightweight installer CLI (`weixin-installer` binary)
- **Version present**: 1.0.2
- Single file: `cli.mjs`

## How to Download New Versions

```bash
# Download latest version of main package
npm pack @tencent-weixin/openclaw-weixin
tar xzf tencent-weixin-openclaw-weixin-<version>.tgz -C tencent-weixin-openclaw-weixin-<version>/

# Or download a specific version
npm pack @tencent-weixin/openclaw-weixin@2.2.0
tar xzf tencent-weixin-openclaw-weixin-2.2.0.tgz -C tencent-weixin-openclaw-weixin-2.2.0/

# Download CLI package
npm pack @tencent-weixin/openclaw-weixin-cli
tar xzf tencent-weixin-openclaw-weixin-cli-<version>.tgz -C tencent-weixin-openclaw-weixin-cli-<version>/
```

Note: `npm pack` downloads to a `.tgz` and extracts into a `package/` subdirectory. Rename/move accordingly.

## Analysis Notes

See [README.md](README.md) for in-depth architecture analysis of v1.0.2.

Key architectural concepts:
- **Long polling**: `ilink/bot/getupdates` with 35s timeout, cursor-based incremental sync
- **Auth**: QR code scan → `bot_token` → Bearer token in all API requests
- **Context token**: Per-message token that must be echoed back when replying
- **Media**: AES-128-ECB encrypted uploads to CDN (`https://novac2c.cdn.weixin.qq.com/c2c`)
- **API base URL**: `https://ilinkai.weixin.qq.com`
- **Session expiry**: errcode `-14` → 30-minute pause before retrying
- **Account storage**: `~/.openclaw/openclaw-weixin/accounts/{accountId}.json`

## Version Comparison

When analyzing a new version, compare against the previous by:

```bash
diff -rq --exclude='*.json' \
  tencent-weixin-openclaw-weixin-2.1.1/src \
  tencent-weixin-openclaw-weixin-<new>/src
```

Also check `CHANGELOG.md` and `CHANGELOG.zh_CN.md` in the new package root.
