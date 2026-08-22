# Electris Energy Meter — firmware releases

Published firmware images for the Electris energy meter, plus the manifest the
devices read to find them. **Binaries only** — the source lives in the private
`TOT102/Energy-meter` repository.

This repository is public because it has to be. Devices carry no credentials,
and a GitHub token in device flash would be extractable from a flash dump and
awkward to rotate across a fleet.

## How a device finds an update

Devices fetch raw files over HTTPS:

```
https://raw.githubusercontent.com/TOT102/energy-meter-firmware/main/manifest.json
```

Deliberately **not** GitHub Releases. Release assets redirect from
`github.com` to `release-assets.githubusercontent.com` with a ~950-character
signed URL that expires within the hour, which would mean redirect handling, a
large URL buffer, and a second certificate authority on the device
(`github.com` is Sectigo; the `githubusercontent.com` hosts are Let's
Encrypt). Raw files are one host, one CA, no redirects, and stable URLs.

## Manifest format

```json
{
  "schema": 1,
  "channels": {
    "stable": {
      "esp32c6": {
        "version": "1.1.0",
        "name": "ota-support",
        "notes": "One line, shown in the web UI",
        "file": "firmware/em-esp32c6-1.1.0.bin",
        "size": 1262998,
        "sha256": "..."
      }
    }
  }
}
```

Keyed by channel and then by target, so a device is never offered an image
built for a different chip. `file` is relative to the manifest, so everything
can be rehosted by changing one setting on the device.

A device installs an entry only when its numeric version is strictly higher
than what the device is running. A development build reporting
`1.1.0-4-gabc1234` has the triplet `1.1.0`, so it is not offered `1.1.0` as an
update — only a genuinely higher release.

## Verification

`sha256` is checked on the device as the image streams in, and **before** the
new slot is made bootable, so a truncated or substituted image is rejected
rather than installed. Published files are never overwritten: a device that
already fetched a version keeps getting the same bytes.

## Publishing a release

From the firmware source repository:

```bash
git tag -a v1.1.0 -m "..." && git push origin v1.1.0
pio run -e esp32-c6-devkitc-1                     # bakes the tag into the binary
python scripts/release.py --repo ../energy-meter-firmware \
    --name ota-support --notes "Self-updating over HTTPS"
```

Then review the diff here and push. The script refuses to publish a version
that is not an exact clean tag, because a `-dirty` build would be offered to
every device as an update to somebody's working tree.
