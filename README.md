# StratoCast

> A client/server music and audio streaming platform for DCS World's Simple Radio Standalone (SRS).

StratoCast lets authenticated remote clients stream music and audio into a DCS mission as SRS radio transmissions. A lightweight client app handles source selection, playlists, and encoding; a server-side relay — co-located with your DCS-SRS server — handles authentication, quotas, and injection of audio onto in-game frequencies.

Think in-universe radio stations, mission-themed background music, GCI/AWACS roleplay ambience, or a "tower ATIS" frequency — all controllable remotely by authorised users without giving any of them the ability to flood the server.

## Acknowledgements

This project is directly inspired by **[SR-Music](https://github.com/ace747/SR-Music)** by [ace747](https://github.com/ace747) — an open-source C# music client that pioneered the idea of broadcasting music into DCS over SRS. SR-Music works brilliantly as a local, SRS-admin-controlled music player with a fixed set of radio stations.

StratoCast extends that core idea in a few key directions:

- **Client/server model** — the audio source and the SRS injection point no longer have to live on the same machine. authorised users can stream from their own PCs.
- **Authentication and quotas** — remote connections are explicitly supported, but protected behind per-user tokens, rate limits, and an admin kill switch, so the feature can't be weaponized as "musical DDoS."
- **Broader source support** — local files across multiple formats (MP3, FLAC, OGG, WAV), with a path toward streaming services via `ytmusicapi` and the Spotify Web API.
- **Unit-bound transmissions** (stretch goal) — read in-mission unit state via DCS Lua export hooks, the same approach [LotATC](https://www.lotatc.com/) uses, so transmissions can inherit line-of-sight and range propagation from a specific in-game unit.

SR-Music remains the simpler, local-only option for a single operator — if that matches your needs, please check it out and give ace747 a star.

## Architecture

<!-- Replace the path below with your diagram file once added to the repo.
     Suggested: save the architecture SVG as docs/architecture.svg -->

![StratoCast architecture diagram](docs/dcs_srs_streaming_architecture.svg)

The system has three logical zones:

**The user machine** runs the **client app** — a UI for selecting audio sources (local files, streaming APIs), managing playlists, and assigning streams to frequencies. The client authenticates to the relay server, decodes and resamples audio, and Opus-encodes it for transmission.

**The server host** runs the **relay server** and a standard **DCS-SRS server** side by side. The relay is the trust boundary: it terminates client connections, authenticates users against issued tokens, enforces quotas and frequency allowlists, transcodes audio as needed, and speaks the SRS client protocol to inject transmissions into the SRS server over loopback. The DCS-SRS install is unmodified — the relay simply appears to it as one or more SRS clients.

**The DCS game** — pilots tune to the assigned frequency on their aircraft radio and hear the broadcast via SRS's existing UDP delivery mechanism.

## Features

### MVP

- Authenticated client connecting to the relay over TLS
- One active stream per authenticated user
- Local file playback: MP3, FLAC, OGG, WAV
- Direct injection into a single SRS frequency
- Basic admin: list authorised users, revoke tokens

### V1

- Multiple concurrent streams on different frequencies
- Playlist management, queueing, and crossfade
- Admin web UI on the server: who's streaming what, where, kill switches
- Audit logging of broadcast history
- Per-user frequency allowlists and concurrent-stream caps

### V2 — external sources

- YouTube Music via `ytmusicapi` (bring your own credentials)
- Spotify Web API integration (subject to ToS review — see notes below)

### Stretch — unit binding

- Optional DCS Lua export hook to surface unit positions to the relay
- Transmissions tied to a specific coalition/unit so SRS's line-of-sight and range attenuation apply naturally
- Compatible with SRS servers running with LOS and Distance Limit enabled (unlike a frequency-only broadcast)

## How it works

1. The client authenticates to the relay with a user-scoped token over TLS.
2. The user selects a source (local file, playlist, or streaming URL) and assigns it to a frequency.
3. The client decodes the source, resamples to 16 kHz mono, Opus-encodes, and streams to the relay.
4. The relay validates the stream against the user's quotas and frequency allowlist, then hands the audio to its SRS adapter.
5. The SRS adapter presents itself as an SRS client over loopback and transmits on the requested frequency.
6. Any DCS pilot tuned to that frequency hears the broadcast through their normal SRS client, with whatever propagation rules the SRS server is configured for.

## Security and abuse prevention

Because remote clients can push audio onto radio frequencies that in-game pilots will hear, the relay is built with hostile-user assumptions from the start:

- **Authentication** — all clients present a token issued by an admin; no anonymous access.
- **Per-user concurrent stream caps** — a compromised credential can't saturate every frequency at once.
- **Frequency allowlists** — a user may be permitted to broadcast on 251.0 MHz but not on an AWACS or ATC frequency.
- **Rate limiting** — connection and bitrate limits per user and globally.
- **Admin kill switch** — any active stream can be terminated immediately from the admin UI.
- **Audit log** — who broadcast what, where, and when, retained for review.
- **TLS everywhere** — no cleartext audio or credentials in transit.

## Roadmap

| Milestone | Scope | Status |
|-----------|-------|--------|
| MVP       | Auth, local files, single frequency, CLI admin   | Planned |
| V1        | Multiple streams, playlists, admin UI, logging   | Planned |
| V2        | Streaming source integrations                    | Planned |
| Stretch   | Unit-bound transmissions via DCS Lua             | Exploratory |

## Getting started

> Setup instructions will be added once the MVP is runnable. In the meantime, the sections below describe the intended layout.

### Prerequisites

- A running [DCS-SimpleRadioStandalone](https://github.com/ciribob/DCS-SimpleRadioStandalone) server
- A host machine with network access from the relay to the SRS server (loopback recommended)
- TLS certificates for the relay's public endpoint

### Server install

```bash
# placeholder
```

### Client install

```bash
# placeholder
```

### First run

```bash
# placeholder
```

## Configuration

Configuration lives in a TOML/YAML file alongside the relay binary. Expected sections include:

- `[server]` — bind address, TLS certificates, SRS server endpoint
- `[auth]` — token store location, token TTL, issuer settings
- `[quotas]` — global and per-user stream/bitrate limits
- `[frequencies]` — allowlist definitions and per-user/per-role mappings
- `[admin]` — admin UI bind address and credentials

A fully annotated example config will ship with the MVP release.

## Notes on streaming service sources

Integrating external streaming sources carries licensing and ToS considerations:

- **Local files** are the intended primary source and carry no platform-level restrictions (use content you have rights to).
- **`ytmusicapi`** sits in a grey area with respect to YouTube's Terms of Service. Use at your own risk; credentials are user-provided and not bundled with the project.
- **Spotify Web API** explicitly prohibits redistribution of audio. Any Spotify integration will be scoped carefully (e.g. metadata/playlist browsing only, with playback from a legally-sourced copy) or may be dropped entirely depending on the results of a ToS review.

If you're running a public server, lean on local files and user-provided URLs.

## Contributing

Contributions are welcome! Before opening a large PR, please open an issue to discuss scope — especially for anything touching the auth layer, the SRS protocol adapter, or the quotas system.

Good first issues, architecture decision records, and a development setup guide will be added under `CONTRIBUTING.md` once the MVP lands.

## Credits

- **[SR-Music](https://github.com/ace747/SR-Music)** by ace747 — the direct inspiration for this project
- **[DCS-SimpleRadioStandalone](https://github.com/ciribob/DCS-SimpleRadioStandalone)** by Ciribob — the underlying SRS system everything here plugs into
- **[LotATC](https://www.lotatc.com/)** — for demonstrating how to bind SRS transmissions to in-game units
- The DCS World community for keeping mission-side radio interesting

## License

TBD — likely GPL-3.0 to align with SR-Music and DCS-SimpleRadioStandalone's licensing lineage.

## Disclaimer

StratoCast is not affiliated with Eagle Dynamics, Ciribob, or ace747. "DCS World" is a trademark of Eagle Dynamics SA.
