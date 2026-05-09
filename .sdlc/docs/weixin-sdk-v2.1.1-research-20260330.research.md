# Research: WeChat SDK v1.0.2 → v2.1.1 Upgrade Analysis

## Summary

The `@tencent-weixin-openclaw-weixin` SDK has been updated from v1.0.2 to v2.1.1. This is a **moderate** update — no types were removed, but several new fields and behavioral changes require attention. The Go implementation (`github.com/tingly-dev/weixin`) needs targeted updates in types, API client, CDN layer, and message handling.

## Background

Our Go module implements the WeChat ilink protocol based on the official TypeScript SDK. The SDK is used as the **reference implementation** for API types, request/response formats, and CDN media handling. The Go code does NOT import the TypeScript SDK — it re-implements the protocol independently.

## Changes by Category

### 1. Type Changes (LOW impact)

#### New fields added to `CDNMedia`:
```typescript
full_url?: string  // Server-returned complete download URL (no client-side stitching needed)
```
**Go impact**: Add `FullURL string` field to `CDNMedia` in `types.go`.

#### New field added to `GetUploadUrlResp`:
```typescript
upload_full_url?: string  // Server-returned complete upload URL
```
**Go impact**: Add `UploadFullURL string` field to `GetUploadURLResponse` in `api/types.go`.

#### New fields added to `ImageItem`:
```typescript
thumb_height?: number
thumb_width?: number
```
**Go impact**: Add `ThumbHeight int` and `ThumbWidth int` to `ImageItem` in `types.go`.

### 2. API Client Changes (MEDIUM impact)

#### New HTTP headers on every request:
- `iLink-App-Id`: from `ilink_appid` field in package.json (value: `"bot"`)
- `iLink-App-ClientVersion`: uint32 encoding of SDK version (e.g. `2.1.1` → `0x00020101`)

**Go impact**: Add these two headers to `buildHeaders()` in `api/client.go`. The version should be hardcoded or derived from a constant matching our Go module version.

#### New exported function `apiGetFetch`:
- Generic GET request with timeout + abort support
- Used for QR code login endpoints (`get_bot_qrcode`, `get_qrcode_status`)

**Go impact**: Currently our QR login in `api/qrcode.go` may already use GET. Verify and align.

### 3. CDN Layer Changes (MEDIUM impact)

#### `full_url` support — server-returned URLs preferred over client-side construction:

**Upload**: `uploadBufferToCdn()` now accepts optional `uploadFullUrl` parameter.
- Priority: `uploadFullUrl` (server-returned) > `uploadParam` + `buildCdnUploadUrl()` (client-side) > error

**Download**: `downloadAndDecryptBuffer()` and `downloadPlainCdnBuffer()` now accept optional `fullUrl` parameter.
- Priority: `fullUrl` (server-returned) > `buildCdnDownloadUrl()` (client-side)
- Fallback controlled by `ENABLE_CDN_URL_FALLBACK = true`

**Go impact**:
- `cdn/upload.go`: `UploadBufferToCdn()` — add optional `uploadFullURL string` parameter; use it when non-empty.
- `cdn/download.go`: `DownloadAndDecryptBuffer()` and `DownloadPlainBuffer()` — add optional `fullURL string` parameter; use it when non-empty.

### 4. Behavioral Changes (MEDIUM impact)

#### contextToken no longer required for sending:
- v1.0.2: All send functions throw `Error("contextToken is required")` when missing
- v2.1.1: Log warning, send without context

**Go impact**: In `message/outbound.go` or wherever contextToken validation occurs — relax to warning-only.

#### Media download checks both `encrypt_query_param` and `full_url`:
- v1.0.2: Only checked `encrypt_query_param`
- v2.1.1: New helper `hasDownloadableMedia(m)` checks both

**Go impact**: In `message/inbound.go` or media download code — also check `full_url` field on media items to determine downloadability.

#### Block streaming support:
- New capability: `blockStreaming: true`
- New streaming config: `{ minChars: 200, idleMs: 3000 }`

**Go impact**: Add block streaming capability to `channel/types.go` or plugin capabilities. This is a channel framework change.

### 5. Infrastructure Changes (LOW impact for Go, INFO only)

These changes are specific to the OpenClaw plugin framework and don't affect our standalone Go implementation:

- **Host version compatibility guard** (`compat.ts`) — OpenClaw-specific
- **Context token disk persistence** — We already handle this in `contexttoken/`
- **Stale account cleanup on re-login** — Nice-to-have feature
- **Smart outbound account resolution** via context token matching — Nice-to-have
- **Enhanced account cleanup** (removes sync buf + context tokens + allowFrom) — Nice-to-have
- **Route tag caching** — Already handled differently in Go
- **Temp directory via framework** — Go handles this independently
- **Config reload trigger** — OpenClaw-specific
- **Uninstall CLI command** — OpenClaw-specific
- **Log upload removed** — OpenClaw-specific

## Change Impact Matrix

| Change | Go Files Affected | Effort | Priority |
|--------|-------------------|--------|----------|
| `CDNMedia.FullURL` field | `types.go` | Trivial | High |
| `GetUploadURLResponse.UploadFullURL` field | `api/types.go` | Trivial | High |
| `ImageItem.ThumbHeight/Width` fields | `types.go` | Trivial | Medium |
| `iLink-App-Id` / `iLink-App-ClientVersion` headers | `api/client.go` | Low | High |
| CDN `full_url` upload support | `cdn/upload.go` | Low | High |
| CDN `full_url` download support | `cdn/download.go` | Low | High |
| contextToken relaxed to warning | send/outbound code | Low | Medium |
| Media download checks `full_url` | `message/inbound.go` or media download | Low | Medium |
| Block streaming capability | `channel/types.go` | Medium | Low |

## Recommendation

**Proceed with targeted updates.** The SDK changes are backward-compatible — no types were removed, only new optional fields added and behavioral relaxations. The update should be done in one pass:

1. **Types first**: Add new fields to `types.go` and `api/types.go`
2. **API client**: Add new headers to `buildHeaders()`
3. **CDN layer**: Add `full_url` support to upload and download
4. **Behavioral**: Relax contextToken requirement, update media download checks
5. **Capability**: Add block streaming flag (optional, low priority)

## Next Steps

1. If approved, transition to **feature workflow** for implementation
2. Run existing tests after each change group
3. Test against live WeChat API to verify new headers and full_url behavior
