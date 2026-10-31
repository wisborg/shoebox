# Shoebox — plan

Shoebox is a self-hosted photo album: albums and collections, photo indexes, slideshows, users
and groups, passkey and password sign-in, four kinds of album visibility, and photo metadata
stripped from the images and shown only to whoever is allowed to see it. It is named after
where most people's photos actually lived for decades.

Its slideshows are [Clack](https://github.com/wisborg/clack), a separate library any website
can embed, named after the sound a slide projector made between slides. The two are built to
the same standard, and this plan holds what they share: the principles, the dependency policy
and the review process. Clack's own design is in its plan (locally, `../clack/docs/plan.md`).

Nothing is built yet. This document records the decisions that shape everything else, with
the alternatives that were rejected and why, and the order the work is done in. Decisions
taken are listed in [Decided](#decided). Every other choice here is a recommendation that
stands until someone argues it down.

The first deployment is local, for testing, with at most a handful of users later. The
collection it is built for is large — more than 125,000 photos going back to 1999, mostly JPEG
and HEIC, in topic folders and Apple Photos — though it starts with a few albums of around a
hundred photos each. Scale and HEIC shape more of this plan than the user count does; see
[A collection that starts in 1999](#a-collection-that-starts-in-1999) and
[HEIC, and Apple Photos](#heic-and-apple-photos).

---

## Principles

These decide ties between otherwise reasonable options.

- **Look and quality over speed of delivery.** There is no deadline. A feature is finished
  when it looks right on a large calibrated screen and on a phone, not when it works.
- **Our own code over someone else's**, unless the someone else is a large, well-maintained
  project whose bugs are found quickly by many users. Every dependency is a decision, written
  down with its reason (see [Dependencies](#dependencies)).
- **Where a photo was taken is personal data.** So is when, and so is the device. The site
  never gives any of it away by accident: not in the image bytes, not in a file name, not in a
  map image, not in a URL.
- **Standalone first.** One binary and one data directory must be a complete, production-grade
  deployment. Anything bigger (a database server, object storage, a CDN) is an option added
  later behind an interface, never a requirement.
- **No third party at view time.** No web fonts from a font CDN, no analytics, no map tiles
  from a tile host, no script from anywhere but the site itself.

---

## Decision 1: languages

### Clack: TypeScript compiled to plain JavaScript, no framework

Clack runs in the visitor's browser, so it is JavaScript in the end. It is written in
TypeScript, ships as one ES module and one CSS file, and has no runtime dependencies; the
reasoning, and the frameworks rejected, are in Clack's plan.

### Shoebox: Go

| | Go | Rust | TypeScript (Node/Deno/Bun) | Python | Java / Kotlin / C# |
|---|---|---|---|---|---|
| memory safety | yes (GC) | yes (ownership) | yes | yes | yes |
| standalone deploy | one static binary, assets embedded | one static binary | runtime + `node_modules` (Deno/Bun can compile, with caveats) | interpreter + packages | runtime |
| standard library for a web app | HTTP/1.1+2, TLS, `html/template` with contextual auto-escaping, crypto, image decoding, SQL interface, fuzzing | small; HTTP, TLS, templates and async are crates | large ecosystem, small stdlib | rich | rich |
| typical dependency tree for this app | a handful | dozens to hundreds (tokio, hyper, serde…) | hundreds | dozens | dozens |
| supply-chain tooling | checksum database, `go.sum`, minimal version selection (no silent upgrades), `govulncheck` with reachability | `cargo audit`, `cargo vet` | npm: the ecosystem with the worst recent record | pip: weak | Maven: decent |
| reuse of osmbase, fitactivity, course | direct import | FFI or subprocess | subprocess | subprocess | subprocess |
| performance for this workload | ample | best | ample | adequate | ample |

**Go**, for three reasons, any one of which would nearly decide it:

1. The map and activity features are imports of libraries that already exist and are
   maintained in this workspace, not a second implementation or a subprocess protocol.
2. The standard library covers almost the whole app — and `html/template` escapes by context
   (HTML, attribute, URL, JS, CSS), which removes the most common web vulnerability by
   construction rather than by care.
3. One static binary with Clack, themes, templates and migrations embedded is exactly
   the "completely standalone" deployment asked for.

Rust would be the choice if the server were parsing hostile input at high volume with no
garbage collector budget. It isn't: uploads come from admins, and the work is dominated by
image processing done once at ingest. Rust's async web stack would cost more dependencies than
the rest of the app combined.

### Shoebox's pages: server-rendered HTML, small TypeScript islands

Pages are rendered by Go with `html/template` and work without JavaScript. Script adds what
only script can: the slideshow (Clack), passkey ceremonies, drag-and-drop upload with
progress, drag-to-reorder. No single-page-app framework, for the same reasons as Clack,
plus: a strict Content-Security-Policy is far easier to hold, first paint is fast, and there
is no client-side router to secure.

---

## Decision 2: build and test tooling — no Node

Both repositories build and test their front-end without Node: esbuild through its Go API for
bundling, `tsgo` (TypeScript's Go port) for type checking, and our own standard-library
WebDriver client to run tests in Safari, Chrome and Firefox. The details are in Clack's plan;
Shoebox's small TypeScript islands use the same tools.

---

## Dependencies

### Policy

- Every dependency is listed in `DEPENDENCIES.md` with what it is for, why we don't write it,
  who maintains it, and its licence; licences are reproduced in `NOTICE`, as in the other
  repos.
- Runtime dependencies of the internet-facing binary are held to a stricter bar than build
  tools, and the two are kept apart: tools live under `tools/` with their own `go.mod`, so the
  server's module graph shows only what the server links.
- `go mod verify` and `govulncheck` in CI; toolchain pinned in `go.mod`; release builds with
  `-trimpath` and reproducible flags; an SBOM with each release.
- Parsers of untrusted bytes that we write ourselves (EXIF, CBOR, ICC, theme files, multipart
  uploads beyond the stdlib) get Go fuzz tests from the first commit.

### Anticipated

| dependency | for | runtime? | verdict |
|---|---|---|---|
| Go standard library | almost everything | yes | — |
| `golang.org/x/crypto` (argon2, acme/autocert) | password hashing, Let's Encrypt | yes | accept: Go team |
| `golang.org/x/image` | WebP/TIFF decoding, resampling kernels | yes | accept: Go team, already used by osmbase |
| a SQLite driver | the standalone database | yes | accept; which one is [Decision 3](#decision-3-storage) |
| `jackc/pgx` | PostgreSQL, if and when | yes | accept later: the standard driver |
| `spf13/cobra` + `wisborg/cobrakit` | the CLI | yes | accept: already used across the workspace |
| `wisborg/secrets` | server secrets (env/file in server mode) | yes | accept: ours |
| `evanw/esbuild` | bundling | build only | accept |
| `tsgo` or `typescript` | type checking | build only | accept |
| `go-webauthn/webauthn` | passkeys | — | **write our own** (see [Authentication](#authentication)) |
| EXIF libraries (`goexif`, `go-exif`) | metadata | — | **write our own**: the first is unmaintained, the second heavy; EXIF is a well-specified TIFF structure |
| libvips / ImageMagick | image processing | — | **not linked**; see [The media pipeline](#the-media-pipeline) |
| ORMs, routers, web frameworks | — | — | **no**: `database/sql` and Go's pattern-matching `http.ServeMux` suffice |

---

## Decision 3: storage

### Data: SQLite by default, PostgreSQL later, both behind one interface

Users, groups, grants, sessions, albums, photo records and their metadata are relational and
queried by permission ("the albums this user may see, newest first, with cover images"). That
is a database's job.

- **SQLite is the default** and is not a toy setting: WAL mode, one writer and many readers,
  comfortably handles a photo site's traffic (which is overwhelmingly reads of static image
  files that never touch the database). The whole site is one file plus the media directory;
  backup is `VACUUM INTO` and a copy.
- **PostgreSQL** is added only when a deployment needs more than one app instance. The SQL is
  kept to the portable subset from the start, migrations are written for both dialects, and
  once Postgres exists the full test suite runs against both. Until then, the storage layer is
  an interface in Go, and nothing outside it writes SQL.
- **Rejected: plain files (JSON/YAML) as the database.** Attractive for hand-editing, but
  concurrent writes, transactions (a user deleted mid-grant), and permission queries would mean
  reimplementing a database badly. Instead, an **export/import to plain files** gives the
  readable, diffable, git-friendly copy without being the source of truth.
- **Rejected: a key-value store (bbolt, Badger).** No query language, so every index is
  hand-maintained; SQLite is a smaller risk than our own indexes.

**The SQLite driver** is a genuine trade-off:

| driver | how | for | against |
|---|---|---|---|
| `modernc.org/sqlite` | SQLite's C machine-translated to Go | pure Go, no cgo; widely used (PocketBase among many) | one principal maintainer; a very large generated codebase |
| `mattn/go-sqlite3` | cgo, the real SQLite amalgamation | the most used; SQLite itself is among the best-tested software there is | cgo: needs a C toolchain, harder cross-compiling; the workspace so far is pure Go |
| `ncruces/go-sqlite3` | SQLite compiled to WebAssembly, run by wazero | pure Go, and the C code runs sandboxed | smaller user base |

Recommendation: start with **`modernc.org/sqlite`**, keep it behind `database/sql` so
switching is a one-line change, and settle it in phase 2 with a benchmark and a look at each
project's health at that time.

### Media: the filesystem, behind an interface

Images live in the data directory under opaque random IDs — never the uploaded file name, which
can itself carry a date or a place. All access goes through `os.Root` (Go 1.24+), so a bug
cannot walk out of the directory. S3-compatible storage is a later backend behind the same
interface; the AWS SDK is enormous, and SigV4 signing for the handful of calls needed is small
enough to write.

```
data/
  shoebox.db
  media/ab/cd/<id>/          # one photo
    full.jpg                 # the full-size image, metadata stripped (see below)
    w320.jpg … w3840.jpg     # display renditions, never wider than full.jpg
  maps/…                     # pre-rendered maps (later); protected like location metadata
  themes/*.json              # site-installed themes
```

---

## Clack, and what Shoebox needs from it

Clack's design — embedding, progression, sizing and the full-size view, image loading,
metadata display, accessibility, and the theme format with its built-in themes — is in
[Clack's plan](https://github.com/wisborg/clack/blob/main/docs/plan.md) (locally,
`../clack/docs/plan.md`). Shoebox relies on four things from it:

- **The manifest format**, which Shoebox produces per album *and per visitor*: it contains only
  the photos and the metadata classes that visitor's grants allow.
- **Metadata hidden by default** and taken only from the manifest — Clack never reads metadata
  from an image file, which is what makes Shoebox's stripping meaningful.
- **Works under Shoebox's strict Content-Security-Policy** with no exceptions.
- **The theme format**, which Shoebox extends rather than duplicates.

### Shoebox extends the theme format

Shoebox's site themes are Clack's format with an extra `site` section (page canvas, header,
album card style, grid spacing, typography scale for page text). One file styles both the
pages and the slideshow on them, so they always match. Site themes live in `data/themes/`,
are validated when loaded, and can be chosen per site and overridden per album. Shoebox ships
site versions of Clack's five built-in themes.

---

## Shoebox

### Pages

- **Home:** the albums this visitor may see, as cards — cover photo, title, date range,
  photo count — in a responsive grid. Unlisted albums are never listed (except to admins).
- **Album index:** every photo in a **justified grid** (rows of equal height, each photo in its
  own aspect ratio, filling the width — the layout serious photo sites use). It is done in CSS
  alone (`flex-grow` proportional to aspect ratio), so it needs no script and never reflows,
  with lazy loading, `srcset`, and ThumbHash placeholders. Title, description, and a
  "Slideshow" entry point.
- **Photo page:** each photo has a permanent URL showing it large, with caption, the metadata
  this visitor may see, and previous/next — complete without script. The slideshow enhances
  this page rather than replacing it, and keeps the URL in step.
- **Slideshow:** Clack, fed Shoebox's manifest for that album and visitor.
- **Map** (later), **sign-in**, **account** (passkeys, sessions), and **admin**.

### The media pipeline

Processing happens **once, at ingest**, never at view time: the web host serves files. Ingest
runs in the server (for web uploads) or in **`shoebox import`** on the admin's own machine,
which processes locally and uploads finished renditions — so a small host never decodes a
50-megapixel image, and the same command can later pre-render maps.

For each photo:

1. **Validate by decoding, not by extension or MIME type.** Read the header first
   (`image.DecodeConfig`) and refuse anything over the pixel limit before allocating for it, so
   a small file claiming to be 100,000 × 100,000 pixels is rejected cheaply.
2. **Extract metadata** with our own parser — EXIF (a TIFF structure, including GPS and the
   offset-time tags without which "taken at 09:00" has no time zone), XMP (XML, via
   `encoding/xml`; where Lightroom and others put title, caption, keywords, rating), and IPTC.
   Into the database, as structured fields plus the raw blocks for later.
3. **Apply orientation.** Browsers rotate by the EXIF orientation tag; once that tag is
   stripped, an unrotated portrait shows sideways. The pixels are rotated before anything else.
4. **Keep the colour profile.** The ICC profile is the one piece of metadata that must survive:
   without it, a Display P3 or Adobe RGB photo shows washed out or oversaturated. Renditions
   carry the source profile; resizing is done in linear light using that profile's tone curve
   (gamma-correct downscaling — the difference between crisp and muddy fine detail).
5. **Make renditions** at a ladder of widths (320 … 3840) with a high-quality Lanczos filter we
   write, and a light post-resize sharpen. Every rendition is a newly encoded image, so it
   carries no metadata by construction.
6. **Make the full-size file by stripping, not re-encoding.** The JPEG is rewritten segment by
   segment, keeping only an allow-list (image data, tables, the ICC profile, the Adobe colour
   transform marker) and dropping everything else — EXIF, GPS, XMP, IPTC, maker notes (which
   hold serial numbers), embedded thumbnails (which can show the photo *before* it was cropped),
   C2PA manifests, Photoshop blocks. An allow-list rather than a block-list, so a metadata
   format nobody has heard of yet is dropped too. The pixels are untouched. PNG and WebP get
   the same treatment at their chunk level (keep image data, transparency and colour chunks;
   drop `eXIf`, `tEXt`/`iTXt`, `EXIF`, `XMP `). If the photo needs rotating, the stripped file
   gets a **new** EXIF block that we write, holding the orientation tag and nothing else — it
   reveals nothing, and browsers honour it. (Rotating the JPEG losslessly instead, as
   `jpegtran` does, means rewriting its compressed blocks; worth doing only if we write our own
   JPEG codec anyway.)
7. **The original is not kept.** The stripped full-size file holds every pixel of the original
   and its colour profile, and the metadata is in the database, so nothing Shoebox needs is lost:
   renditions can be regenerated from it later with a better resampler or encoder. What *is*
   lost is the original byte stream, so **Shoebox is not a backup** of the collection; the
   originals stay in the photographer's own archive, and the docs say so plainly.
8. **Compute** the ThumbHash and the dominant colour, and a content hash of the decoded pixels,
   so re-importing a photo (or the same photo from two folders) is recognised as the same one.

**Formats.** JPEG, HEIC, TIFF, PNG and WebP in from the start — JPEG, TIFF, PNG and WebP
decoded by Go and `x/image`, HEIC as described [below](#heic-and-apple-photos). Out: JPEG for
everything. Go's encoder is baseline 4:2:0 with standard Huffman tables; it is used
first and measured, and if the measurements justify it we write our own encoder with 4:4:4
chroma, optimised Huffman tables and progressive scans — well-specified work, and the largest
single quality lever on file size.

**Not yet:** AVIF input, AVIF/WebP output (pure-Go encoders don't exist at quality), HDR gain
maps (lost on re-encoding today; worth doing properly later), the video half of Live Photos,
RAW files (out of scope).

**Rejected: linking libvips or ImageMagick.** Best-in-class speed and format coverage, but a
large C dependency tree whose purpose is parsing untrusted binary formats, and the end of the
single static binary. Running them as external programs, as fitdash runs ffmpeg, stays
possible for `shoebox import` on the admin's machine.

### HEIC, and Apple Photos

Most of the collection is HEIC, so it is a first-class input, not a later format.

**What a HEIC file is.** A HEIF container — ISO base media boxes, simple and well specified —
holding an image coded with HEVC (H.265), on iPhones as a grid of 512 × 512 tiles, plus an EXIF
block, often XMP, a colour profile (usually Display P3), a thumbnail, and sometimes a depth map,
a portrait matte, and an HDR gain map.

- **The container is ours.** Reading the boxes gives the metadata, the colour profile, the
  dimensions and the rotation without decoding a pixel, and goes through the same fuzzed-parser
  regime as EXIF. In HEIF the rotation and mirroring are container properties (`irot`, `imir`)
  that take precedence over the EXIF orientation tag; getting this wrong is the classic cause
  of sideways iPhone photos.
- **The decoding is the operating system's.** HEVC is a very large codec, and it is covered by
  patent pools; writing a decoder, or bundling one (libde265, via libheif), would put licence
  questions into an Apache-2.0 project that we cannot answer cheaply. macOS ships a licensed
  decoder. `shoebox import`, run on a Mac, asks the system to decode each HEIC losslessly into a
  temporary TIFF with its colour profile — first with Apple's `sips`, and if that loses bit depth
  or the profile, through a small Swift helper using ImageIO directly. Both are Apple's own, not
  a third party's. From the decoded pixels on, the photo goes through the same pipeline as any
  other.
- **The full-size file is a high-quality JPEG** (4:4:4, high quality, the source's profile).
  Only Safari displays HEIC, so serving the stripped HEIC would leave everyone else without a
  full-size view. This is the one case where the full-size file is not the original bytes
  stripped, and each photo records which kind it has. With originals not kept, the HEIC itself
  stays in Apple Photos or the folder it came from.
- **Through the web**, a HEIC upload to a server that is not a Mac is refused with a pointer to
  `shoebox import`, until a server-side decoder is chosen (an external `heif-convert`, configured
  by the operator, who then owns its licence questions).

**Apple Photos** is read through an export, not its library:

- **Not the library database.** A Photos library is a package around an undocumented SQLite
  database that changes between macOS releases; reading it would be fragile and would give an
  importer access to the entire library when it needs one album.
- **Now: export an album from Photos as TIFF** ("Export Photos", with title, keywords, caption
  and location included), and import the folder. That carries Photos' edits and crops — what
  the album should show — losslessly, with the profile, and with the metadata in TIFF's own
  EXIF, XMP and IPTC tags, which our EXIF parser reads natively (EXIF *is* TIFF). Location is
  deliberately included: Shoebox needs it to be able to grant it, and it is stripped from every
  file Shoebox serves either way. "Export Unmodified Original" also works, but loses the edits.
- **Later: a PhotoKit helper** (Swift, with the user's permission prompt) that imports a named
  album straight from the library, with no export step.

**Topic folders** map naturally to albums: `shoebox import` proposes one album per folder,
titled after it, and nested folders as collections. Proposals are reviewed before anything is
published, because a folder name can say more than its owner meant to show ("Home — 12 Elm
St"), and an album title is shown to every visitor.

**The same photo twice**, once in a folder and once in Photos, won't have the same bytes or even
the same pixels after Photos re-encodes it. Besides the exact pixel hash, the importer keeps a
perceptual hash and offers likely duplicates for review rather than deciding by itself.

### Metadata, and who may see it

Metadata is grouped into **classes**, because "may see the camera settings" and "may see where
I was" are different questions:

| class | contains | default |
|---|---|---|
| `description` | title, caption, keywords | visible to anyone who can see the photo |
| `camera` | make, model, lens, focal length, aperture, shutter, ISO | hidden |
| `time` | date and time taken | hidden |
| `location` | coordinates, altitude, direction, place names | hidden; the most sensitive |
| `technical` | dimensions, colour profile, file size | hidden |

Each grant on an album says which classes its holders see. Location can be granted
**approximate** (rounded to a few kilometres, or named only to the town) or **precise**. Every
derived artefact carries the class of what it reveals: a map of where an album's photos were
taken *is* location data, and is protected as such, not as an image.

The admin can edit metadata (fix a wrong clock, add a caption) without touching the image, and
can choose per album to hide a class even from people otherwise allowed to see it.

### Access

Four visibility modes, as asked:

- **Public** — anyone; listed on the home page.
- **Unlisted** — anyone with the link. The link carries a 128-bit random token, can be revoked
  and regenerated, and the album's pages send `noindex` and a `Referrer-Policy` that keeps the
  token from leaking to other sites.
- **Password** — anyone with the album's password. Hashed with Argon2id, attempts throttled
  per album and per address, a successful entry grants access for that browser session, and
  changing the password revokes every grant.
- **Restricted** — only the users and groups assigned to it.

Underneath, the four are **presets of one model**: an album has a list of *grants*, each with a
*principal* (everyone, link holders, password holders, a user, a group) and *capabilities*
(view, each metadata class, download full size). That keeps metadata permissions uniform across
modes, and leaves room for combinations later (restricted, plus a link for one guest) without
a new mode.

**A photo can be in several albums** ("Italy 2019" and "Best of the decade"), so photos form
a library and albums refer to them. Access is always decided *through an album*: a photo page's
URL names the album, and that album's grants decide what is shown. A photo's image files may be
fetched by anyone who can see it through any album; its metadata is shown only by the grants of
the album it is being viewed in, so adding a photo to a public album never quietly exposes the
location granted in a private one.

**Collections:** albums can be grouped (a collection per year, or "Travels"), and collections
nested, so the home page of a collection spanning decades is a few dozen entries rather than
a few hundred.

**Users and groups:** many-to-many. A user is a site admin or not; finer roles (an editor who
can upload but not manage users) can be added later as capabilities on the same model.

**Image URLs** are as protected as the pages: opaque IDs, checked against the visitor's grants
on every request for anything not public. Public images are served with long-lived cache
headers; everything else with `Cache-Control: private`. Signed, expiring URLs come with object
storage or a CDN, if ever.

### Authentication

**Passkeys (WebAuthn) are the primary and recommended sign-in.** They are phishing-resistant by
design — the browser will not use a credential on the wrong site — there is no shared secret on
the server to leak, and they sync across a user's devices. Sign-in offers the passkey autofill
UI (conditional mediation), so most sign-ins are one tap.

The hard part of passkeys is not sign-in but **getting an account and getting back into one**,
so that is designed first:

- **Invitations instead of sign-up.** An admin creates a user and Shoebox produces a single-use,
  short-lived enrolment link, which the admin sends however they like — a message, in person,
  an email of their own. Opening it registers the user's passkey. This needs no email or SMS
  infrastructure at all, and it is also how an admin restores access for someone who lost
  their devices.
- **Several passkeys per user**, with a nudge to register a second.
- **Recovery codes**, issued at enrolment, stored hashed, single use.
- **The first admin** is created with `shoebox admin create`, which prints an enrolment link
  on the server's console.
- **Passwords are supported** alongside passkeys, for users and devices that can't use one:
  Argon2id, a minimum length rather than composition rules, and a check against a local
  (offline) list of breached and common passwords. A **TOTP** second factor (RFC 6238, HMAC
  over a time counter, written here) can be added to any password account. Default policy, a
  site setting: TOTP optional for viewers; **required for an admin signing in with a password**
  — an admin account is the one an attacker wants, and a passkey needs no second factor
  because it already is two (the device, and the biometric or PIN that unlocks it).
- **Email and SMS are deferred**, and when they come they are a "notifier" interface the site
  operator configures with their own service (SMTP server, a webhook) — never one Shoebox picks.
  Following your reasoning, they would only ever deliver invitations and notices, never act as
  a second factor by default.
- **OIDC** ("sign in with" an identity provider the operator already runs, such as Authentik
  or Keycloak) is a later option for people with one.

**Writing WebAuthn ourselves.** The verification a relying party does is protocol plumbing over
primitives the standard library already has (ECDSA P-256, Ed25519, RSA, SHA-256): decode a
small subset of CBOR, parse a COSE key, check the challenge, origin, RP ID hash, flags and
counter, verify a signature. With attestation set to `none` — the recommended choice for a
consumer site, and the only one we need — that is on the order of a thousand lines.
`go-webauthn/webauthn` is the alternative, but it carries CBOR, JWT, TPM and other dependencies
for attestation and metadata-service features we would not use. **Decided: our own**, on the
conditions that it is tested against the specification's test vectors and other
implementations' test suites, fuzzed, and reviewed as described in [Reviews](#reviews) before
Shoebox faces the internet. It is security-critical code that fewer people have looked at than a
library's, and the review process is what makes up for that.

**Sessions:** server-side, an opaque 256-bit token in a `__Host-` cookie (`Secure`, `HttpOnly`,
`SameSite=Lax`), idle and absolute expiry, rotated on sign-in and privilege change, listed on
the account page and revocable. **CSRF:** Go's `http.CrossOriginProtection` (Fetch Metadata and
`Origin` checks) on every state-changing request. **Step-up:** sensitive admin actions (managing
users, changing site settings, deleting albums) require a fresh passkey check if the last one is
older than a few minutes. **Throttling** on every sign-in and password path, by account and by
address. **Audit log** of sign-ins, grants, and admin actions.

### Admin

Albums (create, edit, cover, order, visibility and grants, per-album theme), upload
(drag-and-drop, many at once, resumable chunks, progress, processing status), reorder by drag,
bulk caption editing, metadata review before publishing (a "this album contains location data"
warning with a one-click hide), users, groups, invitations, site settings (title, theme,
footer, defaults), theme management, the audit log, storage use, and backup.

Site configuration lives in the database and is edited in the admin UI; only what is needed to
start — listen address, data directory, public URL, TLS mode — is in a small config file or
the environment.

### Hardening

- Headers on every response: a strict CSP (`default-src 'self'`, no `unsafe-inline`, no
  `unsafe-eval`, `object-src 'none'`, `frame-ancestors 'none'`), HSTS, `nosniff`,
  `Referrer-Policy`, `Permissions-Policy`, COOP/CORP.
- Captions and descriptions are plain text, or a small Markdown subset rendered by our own code
  to a fixed set of elements; never HTML from the database.
- Uploads: admin only, size-limited, decoded under limits and a timeout in a bounded worker
  pool; nothing is ever served in the form it was uploaded with a client-chosen content type;
  SVG is not accepted.
- TLS built in with ACME (`autocert`) for the standalone case, or behind a reverse proxy with
  an explicit list of trusted proxies for client addresses.
- A written threat model in `docs/security.md`, reviewed at the end of each phase.

### Deployment

One binary (`shoebox serve`), one data directory, one small config file. Example `systemd` unit;
an optional minimal container image (static binary on an empty base). `shoebox backup` writes a
consistent snapshot of the database and media; restore is tested, not assumed.

**First, locally.** Development and testing run `shoebox serve` on `localhost`. Passkeys work
there without TLS (browsers treat `localhost` as a secure context), and a development mode
relaxes only what `localhost` forces: HSTS is not sent, and for a browser that won't accept
`Secure` cookies over plain HTTP on `localhost`, Shoebox can serve HTTPS with a certificate it
generates for itself, so the cookie rules are never weakened to make testing easier.
Development mode refuses to start on any address other than loopback, so it cannot end up
exposed by accident.

### A collection that starts in 1999

A personal collection that spans more than 25 years is a different problem from a few albums
of recent phone photos, and the plan takes it as the design case, not an edge case:

- **Scale.** The first albums hold about a hundred photos each, but the collection is over
  125,000 and the design assumes it may all be offered one day: test with a million synthetic
  records. SQLite handles that comfortably with the right indexes; the risk is in pages, not
  the database. Album indexes are paged (a few hundred photos at a time, with the justified
  grid continuing seamlessly as the visitor scrolls), and nothing ever loads "all photos".
- **Import in bulk, and resumably.** `shoebox import` walks a directory tree, runs on every core
  of the admin's machine, records progress so an interrupted import resumes where it stopped,
  and recognises photos it has seen before by their pixel hash. A first import of the whole
  collection will take hours, so it says how far along it is and how long is left. Albums can
  be proposed from the folder structure and reviewed before anything is published.
- **Storage.** The rendition ladder is configurable, and small photos get only the renditions
  smaller than themselves — a 1999 photo of 1024 × 768 is never upscaled to 3840.
- **Early digital cameras are a test corpus.** Photos from 1999–2005 have the oddest metadata:
  malformed EXIF, vendor quirks, no GPS, no time zone, and camera clocks that were never set.
  The parser must be lenient about everything except memory safety. Two tools follow from this:
  - **Running the parser over the whole collection** as a local check (it reports files it
    couldn't read or understood oddly). It runs on your machine only; nothing from it is
    committed.
  - **Batch time correction** for an album ("this camera was 1 h 12 min slow"), and a per-album
    time zone for photos whose metadata has none. Without a zone, a time is stored as local
    "wall-clock" time, never silently guessed into UTC.
- **Scanned prints** have no capture date; a date can be set by hand (to the year or month if
  that's all that is known) and is displayed with its precision ("summer 1987", not
  "1 June 1987 00:00").
- **Browsing by time** across the whole library — a year, then a month — for admins, besides
  browsing by album.

---

## Maps and activities (later)

Both are designed for now so that nothing built earlier gets in their way.

**A map of an album's photos.** osmbase draws a static map for a rectangle from data on local
disk, so the map is rendered **when the album is published**, not when it is viewed — on the
admin's machine through `shoebox import`/`shoebox maps`, or on the server if it holds an osmbase
store. Shoebox keeps the map image (a few sizes) with its projection — bounds and zoom — and the
page places photo markers and clusters over it as an HTML/SVG layer, so it feels interactive
with no tile server. A small pyramid of pre-rendered zoom levels for the album's own extent can
add zooming later. Every map is location data and protected as such; every map carries the
OpenStreetMap (ODbL) credit, and the elevation credits when terrain is drawn, as osmbase's
NOTICE requires of its output.

**An activity as an album.** A FIT, GPX or TCX file attached to an album (read by
fitactivity) gives the route map (course's `routemap`), the place summary ("Richmond → Kew →
Putney"), distance, time, and elevation — and positions photos without GPS along the route by
matching their capture time to the track, after correcting the camera's clock offset. fitdash
videos can become slides once Clack supports video. Activity data adds a sensitive class
of its own — heart rate and other physiological data — hidden by default.

**Keeping the server lean.** Importing the activity and map libraries into the server would
add their dependencies (FIT decoding, keychain access) to the internet-facing binary. Rendering
belongs in `shoebox import`/`shoebox maps`, which can be a separate binary; the server only stores
and serves the results.

---

## Repositories

Two, matching how the rest of the workspace is split by what can be used on its own:

- **Clack** (`github.com/wisborg/clack`) — its own repository and versioned releases, because
  it is meant for any website. Go module for the build tools only; TypeScript and CSS source;
  the theme format, its schema, the built-in themes, and the manifest schema (Clack consumes
  both, so they are defined there).
- **Shoebox** (`github.com/wisborg/shoebox`, this repository) — the Go application. It embeds a
  pinned release of Clack, copied in and checked against the release's hashes by a `tools/`
  command, so updating Clack is a visible, reviewable commit.

Both are Apache-2.0, with third-party licences reproduced in `NOTICE`.

Test photographs in either repository are synthetic (generated test charts with orientation
markers and colour patches, metadata written by our own test EXIF writer) or explicitly
licence-clean. **No personal photo, and nothing with real coordinates, ever enters a commit**,
as in fitdash.

---

## Reviews

Reviews are part of the work, not a step after it, in both repositories. Each level adds to
the one before:

1. **Every topic branch.** The gates pass (format, vet, tests, govulncheck; type check,
   browser tests, size budget), then the branch's diff gets a code review for correctness
   before it is reported ready to merge. Findings are fixed on the branch, or written down
   with the reason they weren't.
2. **Security-sensitive branches** — anything touching authentication, sessions, access
   decisions, parsers of untrusted bytes, uploads, cryptography, or response headers — also get
   a dedicated security review against `docs/security.md`, and you run the deepest
   (multi-agent) review available on them before they merge. They must also add the tests that
   show each relevant attack is refused, not only that the happy path works.
3. **Access control is tested as a matrix, and deny is the default.** Every route is
   registered with an access policy, and registering one without a policy fails the build. A
   generated test walks every route × every kind of visitor (anonymous, link holder, password
   holder, user with and without a grant, group member, admin) × every album mode, and checks
   both the response and that no metadata of a class not granted appears in it.
4. **Parsers are fuzzed continuously**: EXIF, XMP, IPTC, JPEG/PNG/WebP segment walking, ICC,
   CBOR/COSE, theme and manifest files. Crashes found become regression tests.
5. **Dependency changes are reviewed as code**: what changed in `go.sum`, what the new version
   changed upstream, and a `DEPENDENCIES.md` entry.
6. **End of each phase**: a whole-repository review (architecture, security, consistency with
   this plan), and the threat model brought up to date.
7. **Before Shoebox faces the internet**: an external review of authentication and access
   control by someone who did not write it, and a test of the running site from outside
   (headers, TLS, session handling, upload limits, brute-force throttling).

Each review at levels 2, 6 and 7 is recorded in `docs/reviews/` — date, scope, findings, and
what was done about each — so that "was this reviewed?" has an answer that doesn't rely on
memory.

---

## Phases

Each phase ends with something that works end to end, and with the docs and the threat model
brought up to date.

**0 — Foundations**
- Repositories, licences, `DEPENDENCIES.md`, CI gates (format, vet, test, govulncheck; type
  check, build, browser tests, size budget).
- Verify the Node-free toolchain: esbuild through Go, tsgo, the WebDriver harness on all three
  engines.
- Verify the HEIC path on real iPhone photos: does `sips` keep the colour profile and bit
  depth, and does it apply `irot`/`imir` or leave them to us? Does a Photos TIFF export carry
  title, caption, keywords and location where we expect them? The answers decide whether the
  Swift helper is needed. Nothing from these files is committed.
- Design: the visual language and the five themes as static mock-ups with real photographs,
  reviewed on phone, laptop and large screen. Draft theme and manifest schemas.

**1 — Clack, first release**
- Core: slides, manual navigation (controls, keys, swipe), responsive images, decode-before-show,
  preloading, ThumbHash placeholders, crossfade.
- Sizing modes and the full-size view with zoom and pan.
- Auto-play with pause control, visibility and reduced-motion handling.
- Metadata display (hidden by default), theme loading and validation, the five themes.
- Accessibility pass, strict-CSP test page, documentation, a demo page.
- Then: Ken Burns, thumbnail strip, URL fragment sync.

**2 — Shoebox, read-only core**
- Server skeleton, config, SQLite storage and migrations (driver decided by benchmark).
- The media pipeline: decoding under limits, our EXIF/XMP/IPTC parser and HEIF box reader
  (fuzzed), orientation, ICC handling, Lanczos in linear light, the JPEG segment stripper,
  renditions, ThumbHash.
- HEIC through macOS, TIFF from Photos exports, albums proposed from topic folders.
- `shoebox import` from a directory tree, built for the whole collection from the start: parallel,
  resumable, duplicate-aware, with progress; the local whole-collection parser check.
- Public albums only; collections; home, paged index, photo page, slideshow; site themes; all
  of it run locally.

**3 — Accounts and access**
- WebAuthn, passwords with TOTP, invitations, recovery codes, sessions, step-up, throttling,
  audit log.
- Users, groups, grants; unlisted, password and restricted albums; metadata classes and
  permissions, including approximate location; the access-control matrix test.
- The security reviews of [Reviews](#reviews), level 2, on every branch of this phase.

**4 — Administration**
- Web upload (resumable), processing status, reorder, bulk captions, metadata review, site
  settings, theme management, backup and restore.

**5 — Production**
- ACME, header and threat-model review, load test, restore drill, deployment documentation,
  the external review (level 7) before the first deployment outside `localhost`.
  PostgreSQL if a deployment needs it. Measure JPEG output and decide on our own encoder.

**6 — Maps, activities, more formats**
- Pre-rendered photo maps, activity albums, video slides, HEIC uploads to a non-Mac server, the
  PhotoKit helper, notifier interface for email/SMS, OIDC.

---

## Decided

| question | decision | where it shows |
|---|---|---|
| names | Shoebox (the album), Clack (the slideshow, `<clack-show>`) | everywhere |
| languages | Go for Shoebox; TypeScript, no runtime dependencies, for Clack | [Decision 1](#decision-1-languages) |
| toolchain | no Node: esbuild via Go, tsgo, our own WebDriver client | [Decision 2](#decision-2-build-and-test-tooling--no-node) |
| licence | Apache-2.0, both repositories | [Repositories](#repositories) |
| repositories | two: Clack, and Shoebox | [Repositories](#repositories) |
| originals | only the stripped full-size file is kept; Shoebox is not a backup | [The media pipeline](#the-media-pipeline) |
| passwords | supported alongside passkeys, with TOTP; required for admins using a password | [Authentication](#authentication) |
| WebAuthn | our own implementation, under the review conditions | [Authentication](#authentication), [Reviews](#reviews) |
| reviews | thorough, for every repo; strictest for security-critical code | [Reviews](#reviews) |
| first deployment | locally, for testing | [Deployment](#deployment) |
| scale | a handful of users; a few albums of ~100 photos first, a collection of 125,000+ eventually | [A collection that starts in 1999](#a-collection-that-starts-in-1999) |
| sources | topic folders and Apple Photos, mostly JPEG and HEIC | [HEIC, and Apple Photos](#heic-and-apple-photos) |

Still to be settled by measurement rather than preference: the SQLite driver (phase 2), whether
`tsgo` replaces the `typescript` package (phase 0), whether HEIC needs the Swift helper
(phase 0), and whether we write our own JPEG encoder (phase 5).

## Not planned

To keep the scope honest: comments, likes, social features, public sign-up, face recognition,
search beyond titles and captions, editing photos in the browser, and RAW processing. Any of
these can be argued in later; none is assumed.
