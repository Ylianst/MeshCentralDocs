# Agent downloads

MeshCentral obtains its default agent binaries from public GitHub releases.
The agent executables are no longer included in the npm package or the
container images: a new server downloads them on its first start, and a server
without internet access needs them copied in by hand (see
[Offline servers](#offline-servers)). The
release manifest selects an exact repository, tag, asset, size and full-file
SHA384 for each file. It can reference several repositories and release
versions. These downloads do not require a GitHub token and do not use Actions
artifacts or a moving `latest` URL.

The server needs outbound HTTPS to `github.com` and
`release-assets.githubusercontent.com` for the downloads, and to
`api.github.com` for the release checks described below. To go through a
forward proxy, set `settings.npmProxy` in `config.json` to the proxy URL, or
`HTTPS_PROXY` (or `HTTP_PROXY`) in the server's environment. `npmProxy` is used
first when both are set.

## Startup

At startup, MeshCentral checks its download cache before signing Windows agents
or loading installer and update files. Missing files are downloaded, verified
and only then moved into the cache. A complete cache works without internet
access. A file that fails to download or does not match its expected size and
SHA384 is reported as a server warning and left unavailable, while the server
continues starting so administrators can recover it.

Startup waits a few seconds for downloads at most. Files that arrive later are
signed and loaded as soon as they land, without a restart. A first start on a
slow link logs:

```text
Downloading 31 default agent files (119 MiB) from GitHub releases...
Default agent downloads are still running. Starting without them; they load when they finish.
Default agent files downloaded, loading them.
Default agent files loaded.
```

A start with every file already cached logs none of these.

## When GitHub cannot be reached

A connection to GitHub gets 10 seconds to open. When one is refused or never
opens, for example behind a firewall that blocks outbound HTTPS, the remaining
downloads stop at once instead of each waiting out its own timeout, and the
server logs one warning naming the release pages and the folder the files
belong in:

```text
WARNING: Unable to download the 31 default agent files from GitHub: Connection timed out. Retrying in the background. On a server without internet access, download them from https://github.com/Ylianst/MeshAgent/releases/tag/legacy-1.2.6 and https://github.com/Ylianst/MeshCentralAndroidAgent/releases/tag/legacy-1.2.6 into "/opt/meshcentral/meshcentral-data/agentbuilds/legacy-1.2.6".
```

The server runs without the missing files. Agents still connect, but the
installers and agent updates for those platforms are unavailable until the
files arrive.

Missing files are looked for again in the background, first after a minute and
then with the wait doubling up to an hour. Each attempt checks the folder for
files copied in by hand before trying GitHub, so neither a network that comes
back nor copied files need a restart. Attempts stay quiet until files load,
which logs the last two lines shown under [Startup](#startup).

## Offline servers

Copy the release files into `meshcentral-data/agentbuilds/<tag>/`, using the
tag and filenames in the manifest. With the default manifest that means the
agent files from the `legacy-1.2.6` releases of
[MeshAgent](https://github.com/Ylianst/MeshAgent/releases/tag/legacy-1.2.6) and
[MeshCentralAndroidAgent](https://github.com/Ylianst/MeshCentralAndroidAgent/releases/tag/legacy-1.2.6),
saved under their release names in
`meshcentral-data/agentbuilds/legacy-1.2.6/`, for example
`meshcentral-data/agentbuilds/legacy-1.2.6/meshagent_x86-64`. Each file must
match the size and SHA384 in the manifest; anything else in the folder is
ignored. Files are used at the next background check or restart. In a
container, copy them into `agentbuilds/<tag>/` on the `meshcentral-data` volume,
or [build them into your image](#building-them-into-an-image).

Files in `meshcentral-data/agents/` and per-domain `agents-<domain>/` are served
ahead of release files, as before, so existing overrides keep working.

Windows agents are signed at startup with a timestamp from an online timestamp
server. Without internet access, or behind a proxy, this can log
`Failed to sign "<file>"` warnings and the release files are served unsigned;
set `settings.agentTimeStampServer` to false to sign them without a timestamp.

To stop the download attempts on a server that never has internet access, merge
this into `config.json`:

```json
{
  "settings": {
    "agentDownloads": {
      "enabled": false
    }
  }
}
```

A disabled server still loads verified cached files and files copied in by
hand, and logs one warning listing the files that are missing and where they
go.

### Building them into an image

Before downloading, MeshCentral also looks in its own `agents` folder. A file
there that matches the manifest is copied into
`meshcentral-data/agentbuilds/<tag>/` at startup without contacting GitHub. In
a container that folder is part of the image, not the data volume, so an image
built where GitHub can be reached carries the files to a server that cannot
reach it:

```dockerfile
FROM ghcr.io/ylianst/meshcentral:latest
WORKDIR /opt/meshcentral/meshcentral
RUN node agents/check-releases.js --output agents
WORKDIR /opt/meshcentral
```

`check-releases.js --output` downloads the files in the shipped manifest,
checks each one against its size and SHA384, and saves it under its file name.
Files already in the folder that match are kept, and the build fails if any
file cannot be downloaded or verified. On an npm install the folder is
`node_modules/meshcentral/agents`.

`--output` is newer than MeshCentral 1.3.0. On 1.3.0, download the files listed
in `agents/agent-defaults.json` into the same folder yourself and check them
with `node agents/check-releases.js --directory agents`, which fails on a
missing or mismatched file.

The server then needs neither the timestamp server nor GitHub, so set both
options described above:

```json
{
  "settings": {
    "agentTimeStampServer": false,
    "agentDownloads": {
      "enabled": false
    }
  }
}
```

## New releases

Once a day, with a small random delay, the server checks each repository in
its manifest for new releases. Any published release that carries at least one
agent file under its served name, and is not already a pinned default, is
copied to `meshcentral-data/agentbuilds/<tag>/`, newest three per repository
(`settings.agentDownloads.keepReleases`). Copies that are no longer among the
newest are removed; the pinned defaults never are, and neither is a folder the
server did not write.
Each file is verified against the digest GitHub records for the asset and
listed in the folder's `SHA256SUMS` once it is complete, so an interrupted copy
resumes with the files that are missing. The server logs
`Agent release <repository> <tag> downloaded to <directory>` for each copy.

Copies are never served on their own. To try one, copy its file into
`meshcentral-data/agents/` and restart, or wait for the Agents tab, which will
list these copies as builds that can be selected as the server default.

Nothing is downloaded, default or copy, if it would leave less than 256 MiB free
on the disk (`settings.agentDownloads.minFreeSpaceMiB`); the file is reported
as unavailable instead.

Prereleases, such as testing builds, are skipped unless
`settings.agentDownloads.includePrereleases` is true. Set
`settings.agentDownloads.fetchReleases` to false to keep the checks but stop the
copies, and `settings.agentDownloads.checkIntervalHours` to an integer from 1
to 168 to change the schedule, or 0 for no scheduled checks. Setting
`settings.agentDownloads.enabled` to false disables checks, copies and default
downloads together.

## Choosing the defaults

The default manifest is `agents/agent-defaults.json` in the MeshCentral package.
An administrator can lay another manifest over it using
`settings.agentDownloads.manifest`, with a path relative to the data directory
or an absolute path. Each file it lists replaces the shipped pin for that file,
and files it does not list keep the shipped pin, so a release that only carries
some agents can be tried without losing the others. Files MeshCentral has no
agent type for are ignored and named in the log.

For example, to try a MeshAgent testing release, download its
`agent-release.json` into the data directory and merge this into
`config.json`:

```json
{
  "settings": {
    "agentDownloads": {
      "manifest": "agent-release.json"
    }
  }
}
```

This is a server-wide setting: it changes the installers and the files offered
to agents that update. Changing the manifest requires a restart.

The initial defaults preserve the binaries previously shipped in MeshCentral
1.2.6, using the `legacy-1.2.6` migration releases in MeshAgent and
MeshCentralAndroidAgent, so upgrading the server does not change the files
agents receive. The release workflow verifies every pinned download before the
npm release. See [Agent release format](agent-releases.md) for the publication
order.

## Signing

The server signs Windows agents at startup as before. Signing skips a type whose
release file is missing. If an external signing job fails, its incomplete output
is removed, the unsigned release file is served, and a server warning names the
file. A signed copy is only served when it was produced from the release file
currently selected, so a copy left over from an earlier release is never served
for a newer one.

## Storage

Every release lives in `meshcentral-data/agentbuilds/<tag>/`, defaults and
copies alike, with a `SHA256SUMS` listing each file that arrived and verified.
A default is only used after its size and SHA384 match the manifest, so a
partial or replaced file is never served. Signed copies stay in
`meshcentral-data/signedagents/`. Back up
`meshcentral-data` as before; the cache can always be rebuilt from the releases.

## Troubleshooting

- `Unable to download ... from GitHub: Connection timed out.` or
  `Unable to download from this host (<code>).` The server cannot reach GitHub.
  Allow outbound HTTPS to the hosts listed at the top of this page, set a proxy,
  or copy the files in as described under [Offline servers](#offline-servers).
- `Downloads from private or reserved addresses are not allowed.` The server's
  DNS answered with a private or reserved address for a GitHub host, as a DNS
  filter or sinkhole does. Fix the DNS answer or set a proxy.
- `Agent downloads are disabled and ... missing.` Downloads are off and the
  named files are not cached. Copy them into the folder the warning names.
- `<file>: Release asset not found.` The pinned release or asset does not exist
  in the repository. Check the manifest tag and asset names.
- `<file>: The release file does not match its expected size and SHA384.` The
  published asset differs from the manifest. It is not downloaded again until
  the next start. Do not replace published assets; publish a new release and
  update the manifest.
- `Not enough free disk space: ...` Free up space, or lower
  `settings.agentDownloads.minFreeSpaceMiB`. The next background check picks
  the files up.
- `External signing failed (...)` for a file: the unsigned release file is being
  served. Fix the job and restart, or sign the file and place it in
  `meshcentral-data/agents/`.
