# cogstore-arcade

A **games cog store for Cognitum**, served at **https://arcade.shaal.dev**.

Built and signed with [gearbox](https://github.com/shaal/gearbox) (the cog-store protocol +
tooling). This repo is **just curation + the publish pipeline** — it pins a release of the cog
and signs/hosts it. The cog itself is an independent unit in
[`shaal/cog-doom`](https://github.com/shaal/cog-doom), whose CI cross-builds `cog-doom-arm` and
publishes it as a GitHub Release; this store **downloads** that release (it does not build it).
None of this depends on the upstream PR (`cognitum-one/cogs#28`) being accepted.

| | |
|---|---|
| **store_id** | `shaal-arcade` (cogs install as `shaal-arcade/<cog>`) |
| **catalog** | https://arcade.shaal.dev/app-registry.json |
| **signing key id** | `shaal-arcade-2026` |
| **key fingerprint** | `3a5479e78741cd4e41f0988d90ba780bd9b38bbf7412fe00612a264b63c60c11` |

## What's here

- `.github/workflows/publish.yml` — on every push it downloads the pinned `shaal/cog-doom`
  release (`cog-doom-arm` + `cog.toml`), fetches the FreeDoom WAD, signs the store with gearbox,
  and deploys to GitHub Pages. **The pin is the curation knob:** the `COG_DOOM_VERSION` at the top
  of the workflow chooses which cog release the store offers.
- That's it — no cog source lives here (it's in `shaal/cog-doom`).

## Offer a newer cog version

1. Cut a new release in `shaal/cog-doom` (`git tag v0.2.0 && git push --tags`).
2. Bump `COG_DOOM_VERSION` in `.github/workflows/publish.yml` to `v0.2.0` and push. The store
   re-signs + redeploys with the new binary.

## One-time setup (these are yours to do)

1. **Generate the signing key** locally — it must never be committed or leave your machine:
   ```sh
   openssl rand -hex 32 > arcade-signing.key      # 32-byte ed25519 seed, hex
   ```
   Back it up offline. If it ever leaks, rotate to `shaal-arcade-2027` and re-publish.
2. **Add it as a repo secret:** Settings → Secrets and variables → Actions → New repository secret
   → name `STORE_SIGNING_KEY`, value = the 64-hex contents of `arcade-signing.key`.
3. **Enable Pages:** Settings → Pages → Source = **GitHub Actions**; set custom domain
   `arcade.shaal.dev`.
4. **DNS:** add `CNAME  arcade  →  shaal.github.io` at your `shaal.dev` DNS provider.
5. **Push** (or run the workflow manually) → it signs and deploys. After the first run, copy the
   printed fingerprint into the table above.

## Publish from your laptop instead

```sh
# 1. pull the pinned cog release (binary + manifest)
mkdir -p manifests/doom staged/cogs/arm/wads
gh release download v0.1.0 --repo shaal/cog-doom --pattern cog-doom-arm --pattern cog.toml --dir /tmp/cd
cp /tmp/cd/cog.toml manifests/doom/cog.toml
cp /tmp/cd/cog-doom-arm staged/cogs/arm/cog-doom-arm
# 2. fetch the FreeDoom WAD (sha256-checked by publish-store.sh)
curl -L https://github.com/freedoom/freedoom/releases/download/v0.13.0/freedoom-0.13.0.zip -o fd.zip
unzip -j fd.zip '*/freedoom1.wad' -d staged/cogs/arm/wads
# 3. sign + stage (from a gearbox checkout)
examples/publish-store.sh \
  --store-id shaal-arcade --name "Shaal Arcade — games for Cognitum" \
  --base-url https://arcade.shaal.dev --key-id shaal-arcade-2026 \
  --seed-file ./arcade-signing.key \
  --cogs-dir ./manifests --artifacts-dir ./staged \
  --generated-at "$(date -u +%Y-%m-%dT%H:%M:%SZ)" --out ./public --attest
# then upload ./public/ to your host
```

## Notes

- **Licensing:** the DOOM cog binary is **GPL-2.0** (`doomgeneric`); the FreeDoom IWAD is
  BSD-3-Clause. The corresponding source is [`shaal/cog-doom`](https://github.com/shaal/cog-doom)
  (with its `LICENSE`/`NOTICE`) — link it from your store page to satisfy GPLv2's offer-of-source.
- **Install support:** this produces a valid, signed, verifiable store. A Cognitum **Seed** can
  *install* from it once the device runtime supports custom stores (configurable
  `StoreDescriptor` + `https://` fetch + catalog verify — gearbox protocol §2/§4).
- **Verify it yourself** any time:
  ```sh
  gearbox store-info verify public/store.json
  PUB=$(python3 -c "import json;print(json.load(open('public/store.json'))['keys'][0]['pubkey_b64'])")
  gearbox verify public/app-registry.json --key-id shaal-arcade-2026 --pubkey-b64 "$PUB"
  ```
