# Design notes — Omega WinUI

The full design lives in the SDLC workspace: **`WINDOWS_APP_DESIGN.md`** (authoritative for
this app), grounded in the `microsoft/win-dev-skills` v0.7.1 study (`WIN_DEV_SKILLS_STUDY.md`).
The upstream data design is shared with Android Omega:

- `../../sdlc/api/UPSTREAM_SPEC.md` — how the original `jiosaavn-api` repo builds its calls
  against JioSaavn upstream (the spec this client ports 1:1).
- `../../sdlc/api/UPSTREAM_VALIDATION.md` — live-probe results; **overrides the spec wherever
  they differ**. Summary below.

## Upstream design (summary)

- **One endpoint, no REST paths:** `GET https://www.jiosaavn.com/api.php`, multiplexed by the
  `__call` parameter, with fixed params `_format=json&_marker=0&api_version=4&ctx=web6dot0`
  (`ctx=android` only for the two radio calls). No auth, no cookies, no API key.
- **A random real-browser User-Agent per request** (pool ported from the repo's UA constant).
- **Errors are HTTP 200** with error/empty bodies — parse first, then check expected keys;
  absence of a key means "no result", not an exception.
- **Stringly-typed payloads:** `duration`, `play_count`, `year`, `list_count`, `song_count`
  arrive as strings (DTOs keep them as strings / flexible-int converters; mappers parse).
  Other fields are real numbers/booleans — modelled per-field, never generalized.
- **Media decryption:** `encrypted_media_url` is standard Base64 of **DES-ECB/PKCS7**
  ciphertext, key = the 8 ASCII bytes `38346591`. Decrypt → base CDN URL ending `_96.mp4`;
  the quality ladder synthesizes `_12/_48/_96/_160/_320` by FIRST-occurrence replacement of
  `_96` (`.NET string.Replace` replaces all — the implementation splices by index).
  Image ladder swaps the `150x150`/`50x50` token for `50x50`/`150x150`/`500x500`, forces https.
- **Endpoint override** (Settings → Advanced) exists only as an escape hatch if the upstream
  host changes; there is deliberately no "wrapper base URL" concept (hosted instances such as
  saavn.dev died — that failure mode is designed out).

## Validation corrections (applied in Core — these override the spec)

1. **Lyrics:** call `lyrics.getLyrics` with `lyrics_id` = **the song's own id**.
   `more_info.lyrics_id` is `""` in search payloads and absent in details; never gate on
   `song.getDetails.has_lyrics` (always `"false"` there, even for lyric'd songs). The
   *search-payload* `has_lyrics` flag was accurate in every probe and is what the domain
   model's `HasLyrics` carries. A body with no `lyrics` key = no lyrics. Cleaning: `<br>` → `\n`.
2. **Suggestions/radio are broken upstream:** `webradio.createEntityStation` returns a
   `stationid`, but `webradio.getSong` answers `{"stationid": …, "error": "No new song found
   for current radio."}` for every probed station. The client returns an **empty list** for
   that body — a normal result, never an exception — and no feature may depend on it.
3. **Paging:** `search.getResults`' `start` is unreliable (observed `total=4675, start=-4`).
   Page from `total` + accumulated result count; terminate on accumulated ≥ total or short page.
4. **Browse modules:** top-level keys are `radio`, `browse_discover`, `new_albums`, `charts`,
   `top_shows`, `new_trending`, `top_playlists`. Five are **direct arrays**; only
   `radio.featured_stations` and `top_shows.shows` are wrapped objects. Entity `type` inside
   sections is unreliable → parsers branch on **shape** (a `list` of songs ⇒ album-shaped).
   The payload is walked as `JsonElement`; typed DTOs are used per item.
5. **Totals:** playlist/album `list_count` is a **string** and is the true total (page until
   accumulated ≥ it). Global search also returns `episodes`/`shows` sections (unused).
6. **Decryption verified live:** range GETs on `_96`/`_160`/`_320` rungs returned HTTP 206.
   The `320kbps` flag gates whether the `_320` rung is offered. Descending fallback on HTTP
   errors is mandatory in playback/downloads because the ladder is synthesized.

## Component name mapping (design doc → this repo)

| Design doc | This repo |
|---|---|
| `Api/JioSaavnClient`, DTOs, contexts | `Upstream/JioSaavnClient.cs`, `Upstream/Dtos/`, `Upstream/UpstreamJsonContext.cs` |
| `Crypto/MediaUrlFactory` | `Upstream/MediaDecryptor.cs` (decrypt + quality/image ladders) |
| `Domain/*` models | `Models/` |
| DI composition root | `App.xaml.cs` (explicit registrations only) |

## Deferred to later phases (decided, not forgotten)

- **Persistence: Microsoft.Data.Sqlite** + hand-written SQL/row mapping is the chosen library
  (design §6.1). EF Core was rejected (reflection-heavy model building/query translation under
  AOT), sqlite-net rejected (reflection-based attribute mapping), Dapper rejected (IL-emit
  materialization is not Native-AOT compatible). Schema mirrors the Android Room entities
  (favorites, local playlists, history, downloads, recent searches, media ladder cache);
  settings go to a JSON file through a second source-generated context (`AppJsonContext`).
  `Microsoft.Data.Sqlite` is **not referenced yet** — it lands with the persistence phase.
- **Playback:** `Windows.Media.Playback.MediaPlayer` in a singleton `PlayerService`, SMTC
  integration, descending quality fallback, sleep timer (design §7). Honest constraint:
  closing the window ends the process — no background execution promises.
- **Downloads:** Core `DownloadService` (HttpClient streaming, Range resume, ≤3 concurrent),
  `AppNotification` on completion (design §8).
- **Transport policy:** the client already retries 5xx/timeout/IO (max 2, exponential
  backoff); the in-memory song-details cache and ≤3-parallel-call limiter land with the
  repository layer.
- **Window sizing refinement:** DPI-scaled sizing via `GetDpiForWindow` through the
  `Microsoft.Windows.CsWin32` source generator (design §3.5/§9.1); the scaffold uses a
  fixed 1280×800 `AppWindow.Resize`.
- **UI tests:** `ui-tests/ui-tests.ps1` (AutomationId-driven, Windows Sandbox) per design §11
  lands once the first real pages exist.
