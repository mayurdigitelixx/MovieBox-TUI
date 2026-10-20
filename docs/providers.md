# Streaming Providers

MovieBox-TUI searches and streams media across multiple independent providers.

## Available Providers

| Provider | Description |
| :--- | :--- |
| **MovieBox** | Primary streaming catalog with multi-language audio, seasons, and subtitles. |
| **4KHDHub** | High-bitrate 4K UHD and 1080p releases with fast CDN mirrors. |
| **Dramachi** | Asian dramas and series catalog. |
| **CircleFTP** | High-speed local BDIX mirror (Bangladesh ISPs). |
| **DhakaFlix** | Local BDIX media indexer (Bangladesh ISPs). |
| **Addons** | Community Stremio HTTP addons (Cinemeta catalog and streams). |

## Switching Providers

- **Cycle Provider**: Press **`Ctrl+P`** on the home screen to switch to the next provider.
- **Provider Menu**: Click the provider badge in the search bar (or press `Enter` on it) to open the provider selection menu.
- **Enable / Disable Providers**: Open `/settings` → **Content Modes** → **Streaming Sources** to toggle providers on or off.

## BDIX Network Detection

If you are connected through a Bangladeshi ISP supporting BDIX:
- MovieBox-TUI automatically tests local BDIX mirrors on startup and enables them if reachable.
- You can manually re-test your connection anytime via `/settings` → **Maintenance** → **Re-check BDIX Network**.

## Provider Architecture (For Developers)

All providers implement a shared Rust trait in `src/providers/`:

```rust
pub trait Provider: Send + Sync {
    fn id(&self) -> ProviderKind;
    fn capabilities(&self) -> ProviderCapabilities;
    async fn search(&self, query: &str, page: usize) -> Result<Vec<CatalogItem>, ProviderError>;
    async fn details(&self, id: &str) -> Result<MediaDetails, ProviderError>;
}
```

New providers implement `Provider` (and optionally `ReleaseProvider` for multi-mirror streams), then register in `src/service.rs`. Responses are automatically cached as MessagePack envelopes.

### MovieBox CDN Manifest Resolution

MovieBox CDN stream entries parse signed `CloudFront-Policy` and `Edge-Cache-Cookie` tokens to find stream roots. The manifest resolver accepts only `/dash/` resources. Unhosted legacy `/hls/` cookie targets are dropped to prevent 404 failures during playback or downloads.

### MovieBox Stream Resolution (detector links)

As of 2026-10 the MovieBox `play-info` endpoint answers with the **site trailer**
(a ~23 second MP4, byte-identical for every title) instead of the media, and the
`resource` endpoint points `resourceLink` at the same file. Those placeholder URLs are
rejected by `is_trailer_or_notice_url()` and never offered as playable mirrors.

The real media is still served by the **subject details** endpoint:
`subject-api/get` returns `resourceDetectors[].resolutionList[]`, where every entry
carries a signed CDN link on the `*.hakunaymatata.com` CDN:

```text
https://bcdn.hakunaymatata.com/resource/h265/<hash>.mp4?sign=<token>&t=<expiry>
```

`moviebox_detector_json_to_releases()` turns those entries into releases with the real
file size, duration, codec and per-resolution variants, and `episode_streams()` prefers
them, keeping the `play-info`/`resource` path as a fallback in case upstream restores
signed manifests.

Two consequences worth knowing:

- **The links are time limited.** The `t=` query is an expiry timestamp, so the URL is
  refreshed right before playback (`MovieBoxService::refresh_moviebox_source`, used by
  the terminal player path and by the Android bridge). Listed links are never cached.
- **Series episodes are usually not resolved upstream.** Their detector points at an
  index site (`fzmovies.cms etc.`, `kickasstorrents.to etc.`) and carries an empty
  `resolutionList`, so no direct link exists. MovieBox then reports that it has no
  direct source for the title; use **4KHDHub** or **Dramachi** for series.

