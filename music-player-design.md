# Web-Based Music Player: Architecture and Implementation Guide

## Architecture overview

Use **React + TypeScript** for the web app, **Fastify + TypeScript** for APIs, **PostgreSQL** for catalog and rights data, private **S3-compatible object storage** for masters and renditions, and a CDN for signed playback and download URLs. Use **Redis + BullMQ** for ingest and export jobs. Keep third-party playback separate from your own player: platform tracks are metadata and links to official players, never audio sources for your streaming pipeline.

**Request flow:** browser → API for authentication, catalog metadata, and short-lived URLs → CDN for your own renditions. Platform searches and playback use each provider’s official APIs and SDKs. Uploads go to private storage → metadata/rights validation → authorized encodes → CDN.

**Important boundary:** there is no public official YouTube Music API/SDK for a third-party music catalog player. You can offer YouTube discovery via the **YouTube Data API** and playback in the **YouTube IFrame Player**, but describe it as YouTube—not YouTube Music.

## 1. Architecture and data model

Suggested layout:

```text
apps/
  web/src/
    player/Player.tsx
    media/capabilities.ts
    platforms/apple.ts
    platforms/spotify.ts
    platforms/youtube.ts
  api/src/
    auth/
    routes/downloads.ts
    routes/platforms.ts
    jobs/transcode.ts
    ingest/
packages/
  shared/src/types.ts
infra/
  migrations/
  deploy/
```

Keep original masters private. Store generated, authorized renditions separately, with codec, protocol, bitrate, channel layout, and Atmos/JOC metadata. A `download_enabled` flag is necessary, but it is not sufficient by itself: also record the rights holder, license evidence, territory, and any expiry or restrictions.

Example PostgreSQL fields:

```sql
CREATE TABLE tracks (
  id uuid PRIMARY KEY,
  title text NOT NULL,
  artist_id uuid NOT NULL,
  master_key text NOT NULL,
  rights_holder text NOT NULL,
  license_type text NOT NULL,
  license_reference text,
  territory_codes text[],
  rights_expires_at timestamptz,
  download_enabled boolean NOT NULL DEFAULT false,
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE renditions (
  id uuid PRIMARY KEY,
  track_id uuid NOT NULL REFERENCES tracks(id),
  protocol text NOT NULL CHECK (protocol IN ('hls', 'dash', 'file')),
  codec text NOT NULL,
  mime_type text NOT NULL,
  channels smallint NOT NULL,
  bitrate integer NOT NULL,
  is_lossless boolean NOT NULL DEFAULT false,
  is_eac3_joc boolean NOT NULL DEFAULT false,
  manifest_key text NOT NULL,
  UNIQUE(track_id, protocol, codec, bitrate)
);
```

## 2. Own-catalog streaming

### Browser capability detection

Do not infer Atmos from a codec string alone. `MediaCapabilities` and `canPlayType()` can indicate likely decode support; they cannot prove that a device renders Dolby Atmos spatially. Mark a rendition as Atmos only when trusted catalog metadata says it contains E-AC-3 JOC, and enable it only for tested browser/device combinations.

Lossless ALAC/FLAC in fMP4 is not broadly supported by browser MSE implementations. Probe it and only offer it when the browser reports support. Keep a high-bitrate AAC/Opus rendition available as the compatibility path.

```ts
// apps/web/src/media/capabilities.ts

export type Tier = "atmos" | "lossless" | "standard";

export type Rendition = {
  id: string;
  url: string;
  protocol: "hls" | "dash";
  mimeType: string;           // Include codecs, e.g. audio/mp4; codecs="alac"
  tier: Tier;
  channels: number;
  bitrate: number;
  isEac3Joc?: boolean;        // Set from trusted ingest metadata, never guessed.
};

async function canDecode(r: Rendition): Promise<boolean> {
  const media = document.createElement("audio");
  const browserHint = media.canPlayType(r.mimeType) !== "";

  if (!navigator.mediaCapabilities) return browserHint;

  try {
    const info = await navigator.mediaCapabilities.decodingInfo({
      type: "media-source",
      audio: {
        contentType: r.mimeType,
        channels: String(r.channels),
        bitrate: r.bitrate,
        samplerate: 48000,
      },
    });
    return info.supported;
  } catch {
    // Older implementations may reject valid but uncommon codec strings.
    return browserHint;
  }
}

export async function chooseRendition(
  renditions: Rendition[],
  preferAtmos: boolean,
  atmosDeviceValidated: boolean,
): Promise<Rendition | undefined> {
  const ordered = [...renditions].sort((a, b) => {
    const rank = (r: Rendition) => {
      if (preferAtmos && atmosDeviceValidated && r.isEac3Joc) return 0;
      if (r.tier === "lossless") return 1;
      return 2;
    };
    return rank(a) - rank(b) || b.bitrate - a.bitrate;
  });

  for (const rendition of ordered) {
    if (rendition.isEac3Joc && (!preferAtmos || !atmosDeviceValidated)) continue;
    if (await canDecode(rendition)) return rendition;
  }
  return undefined;
}
```

### HLS/DASH packaging and player

Use HLS as the broad delivery path, with `hls.js` for MSE browsers and native HLS where available. Use `dash.js` for DASH manifests where desired. The browser player must fail visibly when no rendition is playable; do not silently label a fallback as lossless or Atmos.

```tsx
// apps/web/src/player/Player.tsx
import { useEffect, useRef, useState } from "react";
import Hls from "hls.js";
import dashjs from "dashjs";
import { chooseRendition, type Rendition } from "../media/capabilities";

type Track = {
  id: string;
  title: string;
  artist: string;
  renditions: Rendition[];
};

type Props = {
  track: Track;
  queue: Track[];
  atmosDeviceValidated: boolean; // Set only by your tested device policy.
};

export function Player({ track, queue, atmosDeviceValidated }: Props) {
  const audioRef = useRef<HTMLAudioElement>(null);
  const [preferAtmos, setPreferAtmos] = useState(false);
  const [volume, setVolume] = useState(0.8);
  const [normalizationDb, setNormalizationDb] = useState(0);
  const [selected, setSelected] = useState<Rendition>();
  const [error, setError] = useState("");
  const [queueIndex, setQueueIndex] = useState(0);

  useEffect(() => {
    let hls: Hls | undefined;
    let dash: ReturnType<ReturnType<typeof dashjs.MediaPlayer>> | undefined;
    let cancelled = false;

    async function load() {
      setError("");
      const rendition = await chooseRendition(
        track.renditions,
        preferAtmos,
        atmosDeviceValidated,
      );
      if (cancelled) return;
      if (!rendition) {
        setError("This browser cannot play an available rendition.");
        return;
      }
      setSelected(rendition);

      const audio = audioRef.current;
      if (!audio) return;
      audio.pause();
      audio.removeAttribute("src");
      audio.load();

      if (rendition.protocol === "hls" && Hls.isSupported()) {
        hls = new Hls();
        hls.loadSource(rendition.url);
        hls.attachMedia(audio);
        hls.on(Hls.Events.ERROR, (_event, data) => {
          if (data.fatal) setError(`HLS playback failed: ${data.type}`);
        });
      } else if (rendition.protocol === "dash" && dashjs.MediaPlayer().isSupported()) {
        dash = dashjs.MediaPlayer().create();
        dash.initialize(audio, rendition.url, false);
        dash.on(dashjs.MediaPlayer.events.ERROR, () => {
          setError("DASH playback failed.");
        });
      } else if (rendition.protocol === "hls" && audio.canPlayType("application/vnd.apple.mpegurl")) {
        audio.src = rendition.url; // Native HLS, commonly Safari.
      } else {
        setError("No compatible HLS/DASH playback path is available.");
      }
    }

    void load();
    return () => {
      cancelled = true;
      hls?.destroy();
      dash?.reset();
    };
  }, [track, preferAtmos, atmosDeviceValidated]);

  useEffect(() => {
    // Apply replay-gain/EBU R128 metadata separately from the user's volume.
    const gain = Math.pow(10, normalizationDb / 20);
    if (audioRef.current) audioRef.current.volume = Math.min(1, volume * gain);
  }, [volume, normalizationDb]);

  const badge = selected?.isEac3Joc
    ? "Atmos"
    : selected?.tier === "lossless"
      ? "Lossless"
      : "Standard";

  const next = () => {
    if (queueIndex + 1 < queue.length) setQueueIndex(queueIndex + 1);
  };

  return (
    <section aria-label="Music player">
      <audio
        ref={audioRef}
        controls
        preload="auto"
        onEnded={next}
        aria-label={`Playback for ${track.title} by ${track.artist}`}
      />
      <p aria-live="polite">
        {track.title} — {track.artist} · <strong>{badge}</strong>
      </p>
      {error && <p role="alert">{error}</p>}

      <label>
        Volume
        <input
          type="range" min="0" max="1" step="0.01" value={volume}
          onChange={(e) => setVolume(Number(e.target.value))}
        />
      </label>
      <label>
        Normalization
        <input
          type="range" min="-12" max="6" step="0.5" value={normalizationDb}
          onChange={(e) => setNormalizationDb(Number(e.target.value))}
        />
      </label>
      <label>
        Prefer Atmos
        <input
          type="checkbox" checked={preferAtmos}
          onChange={(e) => setPreferAtmos(e.target.checked)}
        />
      </label>

      <button type="button" onClick={next} aria-label="Play next track">
        Next
      </button>
      <ol aria-label="Queue">
        {queue.map((item, index) => (
          <li key={item.id} aria-current={index === queueIndex ? "true" : undefined}>
            {item.title} — {item.artist}
          </li>
        ))}
      </ol>
    </section>
  );
}
```

This player supports adaptive manifests, a queue, volume, normalization, and a quality badge. **A per-track source change is not guaranteed sample-accurate gapless.** For true album gaplessness, package an album as a continuous, codec-consistent manifest with chapter/track markers, or use a tested dual-deck Web Audio implementation. Do not promise gapless across arbitrary codec or manifest switches.

For a standard AAC HLS rendition, FFmpeg can produce fMP4 segments:

```bash
ffmpeg -i authorized-master.wav \
  -c:a aac -b:a 256k -ar 48000 \
  -f hls -hls_playlist_type vod \
  -hls_segment_type fmp4 -hls_time 6 \
  -hls_fmp4_init_filename init.mp4 \
  -hls_segment_filename 'segment-%05d.m4s' playlist.m3u8
```

Generate ALAC/FLAC fMP4 and DASH renditions only with a packager/container/browser combination validated in your target matrix. E-AC-3 JOC authoring requires appropriate Dolby-licensed encoding and packaging; a generic FFmpeg build is not a substitute for that licensing or validation.

A binaural fallback should be a rights-cleared, authored stereo binaural mix, or an explicitly designed spatial downmix from source material you are licensed to process. Do not imply that ordinary stereo or an E-AC-3 stream automatically provides binaural output.

## 3. Official platform integrations

The quality controls exposed by these browser SDKs do **not** make them lossless or Atmos players:

- **Apple MusicKit JS:** official subscriber playback and metadata; do not promise web lossless/Atmos. The public web SDK does not provide a general quality-tier selector you can rely on.
- **Spotify Web Playback SDK:** official browser playback and current playback metadata; Premium is required. The SDK does not expose a lossless/Atmos mode or a dependable user-selectable browser bitrate.
- **YouTube IFrame Player:** official YouTube video playback; it does not expose an audio-only quality tier. The Data API is for discovery/metadata, not media delivery. There is no public official YouTube Music player API/SDK.

Use OAuth Authorization Code with PKCE where supported, validate `state`, use exact redirect URIs, and keep refresh tokens server-side. Give SDKs only short-lived access tokens when they require them; keep those in memory, not local storage. Protect cookie-authenticated API mutations with CSRF defenses.

```ts
// apps/web/src/platforms/apple.ts
export async function connectAppleMusic() {
  const developerToken = await fetch("/api/apple/developer-token", {
    credentials: "same-origin",
  }).then(async (r) => {
    if (!r.ok) throw new Error("Could not obtain Apple developer token");
    return (await r.json()).token as string;
  });

  const music = await window.MusicKit.configure({
    developerToken,
    app: { name: "My Player", build: "1.0.0" },
  });

  // User interaction should initiate authorization.
  const musicUserToken = await music.authorize();
  return { music, musicUserToken };
}

export async function searchApple(music: MusicKit.MusicKitInstance, term: string) {
  return music.api.search(term, {
    types: ["songs", "albums", "artists", "playlists"],
    limit: 25,
  });
}
```

```ts
// apps/web/src/platforms/spotify.ts
type SpotifyPlayer = {
  connect(): Promise<boolean>;
  addListener(event: string, callback: (data: any) => void): boolean;
  getCurrentState(): Promise<any>;
  resume(): Promise<void>;
  pause(): Promise<void>;
};

export async function connectSpotify() {
  // Backend performs PKCE login/callback and returns a short-lived access token.
  const playerToken = async () => {
    const r = await fetch("/api/spotify/player-token", { credentials: "same-origin" });
    if (!r.ok) throw new Error("Spotify session expired");
    return (await r.json()).accessToken as string;
  };

  const sdk = window.Spotify;
  if (!sdk) throw new Error("Spotify Web Playback SDK did not load");

  const player: SpotifyPlayer = new sdk.Player({
    name: "My Player",
    getOAuthToken: (callback: (token: string) => void) => {
      void playerToken().then(callback).catch((e) => console.error(e));
    },
    volume: 0.8,
  });

  player.addListener("player_state_changed", (state) => {
    // Render Spotify's official playback metadata; never fetch or capture audio.
    window.dispatchEvent(new CustomEvent("spotify-now-playing", { detail: state }));
  });

  await player.connect();
  return player;
}

export async function searchSpotify(query: string) {
  // Proxy this through your API, which attaches the user's authorized token.
  const r = await fetch(`/api/spotify/search?q=${encodeURIComponent(query)}`);
  if (!r.ok) throw new Error("Spotify search failed");
  return r.json();
}
```

```ts
// apps/web/src/platforms/youtube.ts
export async function searchYouTube(query: string) {
  // API key stays on your server; this endpoint uses the official YouTube Data API.
  const r = await fetch(`/api/youtube/search?q=${encodeURIComponent(query)}`);
  if (!r.ok) throw new Error("YouTube search failed");
  return r.json(); // Return videoId/title/channel; embed the official player.
}

export function createYouTubePlayer(
  elementId: string,
  videoId: string,
  onNowPlaying: (metadata: unknown) => void,
) {
  return new YT.Player(elementId, {
    videoId,
    playerVars: { playsinline: 1 },
    events: {
      onReady: (event: YT.PlayerEvent) => {
        onNowPlaying(event.target.getVideoData());
      },
      onStateChange: (event: YT.OnStateChangeEvent) => {
        onNowPlaying(event.target.getVideoData());
      },
      onError: () => console.error("YouTube embedded playback failed"),
    },
  });
}
```

For platform playlists and browsing, call each platform’s documented API using the user’s authorized scopes and permissions. Do not create a “unified playlist” by copying platform audio; store provider IDs and metadata references, then launch playback in the corresponding official player. Review API policies and commercial-use approval requirements before release.

## 4. Rights-gated downloads and transcoding

The API—not the browser—must decide whether a track can be downloaded. Check the authenticated user, rights flag, territory/expiry, and rate limit on every request. Return a short-lived signed URL only for a completed export. Apply CSRF protection to the session-authenticated POST route.

```ts
// apps/api/src/routes/downloads.ts
import type { FastifyInstance } from "fastify";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";
import { GetObjectCommand } from "@aws-sdk/client-s3";
import { exportQueue } from "../jobs/queue";
import { prisma } from "../db";
import { s3 } from "../storage";

const formats = {
  wav: { extension: "wav", contentType: "audio/wav" },
  flac: { extension: "flac", contentType: "audio/flac" },
  alac: { extension: "m4a", contentType: "audio/mp4" },
  mp3: { extension: "mp3", contentType: "audio/mpeg" },
  aac: { extension: "m4a", contentType: "audio/mp4" },
} as const;

export async function registerDownloadRoutes(app: FastifyInstance) {
  app.post<{ Params: { trackId: string }; Body: { format: string } }>(
    "/tracks/:trackId/download",
    { preHandler: [app.authenticate, app.csrfProtection, app.downloadRateLimit] },
    async (request, reply) => {
      const format = request.body.format as keyof typeof formats;
      if (!(format in formats)) return reply.code(400).send({ error: "Unsupported format" });

      const track = await prisma.track.findUnique({
        where: { id: request.params.trackId },
        select: {
          id: true,
          downloadEnabled: true,
          rightsExpiresAt: true,
          territoryCodes: true,
          artist: { select: { ownerUserId: true } },
        },
      });

      if (!track || !track.downloadEnabled) {
        return reply.code(403).send({ error: "Downloads are not enabled for this track" });
      }
      if (track.rightsExpiresAt && track.rightsExpiresAt <= new Date()) {
        return reply.code(403).send({ error: "Download rights have expired" });
      }
      await app.assertTrackTerritory(request.user, track.territoryCodes);

      const existing = await prisma.trackExport.findUnique({
        where: { trackId_format: { trackId: track.id, format } },
      });

      if (!existing || existing.status !== "ready" || !existing.objectKey) {
        await exportQueue.add(
          "track-export",
          { trackId: track.id, format, requestedBy: request.user.id },
          { jobId: `${track.id}:${format}`, removeOnComplete: true },
        );
        return reply.code(202).send({ status: "processing" });
      }

      const url = await getSignedUrl(
        s3,
        new GetObjectCommand({
          Bucket: process.env.MEDIA_BUCKET!,
          Key: existing.objectKey,
          ResponseContentType: formats[format].contentType,
          ResponseContentDisposition: `attachment; filename="${track.id}.${formats[format].extension}"`,
        }),
        { expiresIn: 300 },
      );
      return reply.send({ status: "ready", url, expiresInSeconds: 300 });
    },
  );
}
```

`authenticate`, `csrfProtection`, `downloadRateLimit`, and `assertTrackTerritory` above are required application middleware/helpers, not optional stubs to omit. Also ensure the user is allowed to access the catalog item under your product’s policy.

The worker must re-check rights before processing, use argument arrays rather than shell interpolation, upload only to private storage, and clean up temporary files:

```ts
// apps/api/src/jobs/transcode.ts
import { Worker } from "bullmq";
import { spawn } from "node:child_process";
import { mkdtemp, rm } from "node:fs/promises";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { prisma } from "../db";
import { downloadMaster, uploadExport } from "../storage";

const encoders = {
  wav: ["-c:a", "pcm_s24le"],
  flac: ["-c:a", "flac"],
  alac: ["-c:a", "alac", "-f", "ipod"],
  mp3: ["-c:a", "libmp3lame", "-b:a", "320k"],
  aac: ["-c:a", "aac", "-b:a", "256k", "-f", "ipod"],
} as const;

function runFfmpeg(args: string[]) {
  return new Promise<void>((resolve, reject) => {
    const child = spawn("ffmpeg", ["-nostdin", "-v", "error", ...args], {
      stdio: ["ignore", "ignore", "pipe"],
    });
    let stderr = "";
    child.stderr.on("data", (chunk: Buffer) => { stderr += chunk.toString(); });
    child.on("error", reject);
    child.on("close", (code) => {
      if (code === 0) resolve();
      else reject(new Error(`FFmpeg exited ${code}: ${stderr.slice(-2000)}`));
    });
  });
}

new Worker("track-export", async (job) => {
  const { trackId, format } = job.data as {
    trackId: string;
    format: keyof typeof encoders;
  };
  if (!(format in encoders)) throw new Error("Unsupported export format");

  const track = await prisma.track.findUnique({ where: { id: trackId } });
  if (!track?.downloadEnabled ||
      (track.rightsExpiresAt && track.rightsExpiresAt <= new Date())) {
    throw new Error("Track is not currently authorized for download");
  }

  const dir = await mkdtemp(join(tmpdir(), "music-export-"));
  const input = join(dir, "master");
  const output = join(dir, format === "alac" || format === "aac" ? "export.m4a" : `export.${format}`);

  try {
    await downloadMaster(track.masterKey, input);
    await runFfmpeg(["-i", input, ...encoders[format], output]);

    const objectKey = `exports/${trackId}/${format}/${crypto.randomUUID()}`;
    await uploadExport(objectKey, output);

    await prisma.trackExport.upsert({
      where: { trackId_format: { trackId, format } },
      create: { trackId, format, objectKey, status: "ready" },
      update: { objectKey, status: "ready" },
    });
  } catch (error) {
    await prisma.trackExport.updateMany({
      where: { trackId, format, status: "processing" },
      data: { status: "failed" },
    });
    throw error; // Let BullMQ record and retry/report the failure.
  } finally {
    await rm(dir, { recursive: true, force: true });
  }
});
```

Use a dedicated unprivileged worker/container with CPU, memory, and duration limits. Validate uploaded file types and metadata; do not trust extensions or client-provided codec labels. When rights are revoked, stop new exports, invalidate stored exports as appropriate, and use short URL expirations to limit already-issued access.

## 5. Player UX and accessibility

The example provides keyboard-operable native controls, labels, live status, a quality badge, queue display, volume, and normalization. For a responsive interface, keep the player docked on small screens and place the queue beside it on wider screens. Add explicit loading and error states, disable unavailable quality options, and preserve focus when queue items change.

Compute normalization gain from ingest-time loudness analysis (for example, EBU R128) and store the result with track metadata. Apply a bounded gain; do not normalize by altering the source file at playback time.

## 6. Backend and operations

- **Authentication:** secure, HttpOnly, SameSite cookies; CSRF protection for state-changing requests; short-lived access sessions.
- **OAuth:** validate `state` and PKCE verifier; exact redirect URIs; keep refresh tokens encrypted server-side; never log tokens.
- **Uploads:** presigned upload to a quarantine bucket, virus/file validation, metadata extraction, rights-holder attestation, then controlled packaging.
- **Rights:** persist rights holder, license reference, allowed territories, expiry, download permission, and audit history. Require human review before enabling downloads or Atmos distribution.
- **Abuse prevention:** per-user/IP limits, export concurrency and duration limits, signed URLs, private bucket origin access, and rate-limited search proxies.
- **Observability:** structured logs with request/job IDs; never log credentials, signed URLs, or raw OAuth payloads. Track ingest/transcode failures and CDN errors.

## 7. Roadmap

1. **MVP:** own catalog, AAC HLS, private uploads, rights metadata, authentication, queue, standard player, download gating for one format.
2. **Beta:** lossless renditions behind capability probes, export worker and formats, artist upload flow, playlists, Apple/Spotify integrations.
3. **Full release:** validated Atmos pipeline and device matrix, DASH where needed, continuous album manifests for gapless playback, territory-aware rights enforcement, operations dashboards, and commercial/API-policy approvals.

## Setup guide

1. Create PostgreSQL, Redis, private object storage, and a CDN with private-origin access.
2. Configure Apple Music, Spotify, and Google developer projects; register exact OAuth redirect URIs and permitted origins.
3. Install FFmpeg in the isolated worker image. Add a licensed Dolby-authoring toolchain only if you have the required rights and agreements.
4. Apply migrations; configure API and worker; upload a rights-cleared test track.
5. Verify the generated manifest and renditions with the actual supported browsers and devices before enabling a quality option in production.
6. Deploy API, web app, and workers separately; use TLS, secret manager, backups, CDN cache policy, and monitoring.

### Environment variables

```text
DATABASE_URL=
REDIS_URL=
SESSION_SECRET=
TOKEN_ENCRYPTION_KEY=
WEB_ORIGIN=
API_ORIGIN=

MEDIA_BUCKET=
MEDIA_REGION=
MEDIA_ENDPOINT=
MEDIA_ACCESS_KEY_ID=
MEDIA_SECRET_ACCESS_KEY=
CDN_BASE_URL=
SIGNED_URL_TTL_SECONDS=300

APPLE_TEAM_ID=
APPLE_KEY_ID=
APPLE_PRIVATE_KEY=
APPLE_BUNDLE_ID=

SPOTIFY_CLIENT_ID=
SPOTIFY_CLIENT_SECRET=
SPOTIFY_REDIRECT_URI=

GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GOOGLE_REDIRECT_URI=
YOUTUBE_API_KEY=
```

Keep secrets in a managed secret store, not source control. The Apple private key and provider refresh tokens belong on the server.

### Licensing

You need the relevant master and composition rights for your catalog, explicit download/export permissions, and rights to process and distribute each format in each territory. Dolby Atmos/E-AC-3 JOC creation and distribution may require Dolby licensing and approved tools. Apple, Spotify, and Google each impose developer terms, playback requirements, regional restrictions, branding rules, and possibly commercial approval; confirm the current terms before launch.

### Testing checklist

- Verify a track with downloads disabled cannot be exported by calling the API directly.
- Verify expired and territory-restricted rights are rejected; confirm URLs expire and buckets remain private.
- Test each output format by decoding the result and checking codec/container metadata.
- Confirm unsupported renditions are not offered and the badge matches the selected rendition.
- Test autoplay restrictions, queue transitions, keyboard-only operation, screen-reader labels, and narrow layouts.
- Test OAuth callback state/PKCE, token expiry, revocation, and absence of tokens in logs/storage.
- Test Apple/Spotify playback only through their official SDKs; test YouTube playback only in the official IFrame.
- **Atmos:** test a known E-AC-3 JOC asset on the actual supported Safari, Edge, and Chrome versions and hardware/output routes. Confirm the stream is JOC and that the device actually renders spatial audio. A positive `canPlayType()` or `MediaCapabilities` result alone is not an Atmos pass.

## Official references

- [MusicKit JS](https://developer.apple.com/documentation/musickitjs)
- [Spotify Web Playback SDK](https://developer.spotify.com/documentation/web-playback-sdk)
- [YouTube IFrame Player API](https://developers.google.com/youtube/iframe_api_reference)
- [YouTube Data API search](https://developers.google.com/youtube/v3/docs/search/list)
- [MediaCapabilities](https://developer.mozilla.org/en-US/docs/Web/API/MediaCapabilities/decodingInfo)
- [Apple HLS authoring specification](https://developer.apple.com/streaming/hls-authoring-specification-for-apple-devices.html)
