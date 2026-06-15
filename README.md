# cogstore-arcade

A **games cog store for Cognitum**, served at **https://arcade.shaal.dev**.

Built and signed with [gearbox](https://github.com/shaal/gearbox) (the cog-store protocol +
tooling). This store is **fully self-contained**: the DOOM cog's source is vendored here under
`cog/doom/`, so the store builds, signs, and serves it with **no dependency on any other repo**
(the only external fetch is the FreeDoom IWAD from FreeDoom's own release, sha256-pinned). The
upstream cog lives in [`cognitum-one/cogs`](https://github.com/cognitum-one/cogs) (PR #28), but
this store does not depend on it being accepted.

| | |
|---|---|
| **store_id** | `shaal-arcade` (cogs install as `shaal-arcade/<cog>`) |
| **catalog** | https://arcade.shaal.dev/app-registry.json |
| **signing key id** | `shaal-arcade-2026` |
| **key fingerprint** | _(printed on first publish — paste it here so users can confirm it on add-store / TOFU)_ |

## What's here

- `cog/doom/` — the **vendored DOOM cog source** (`cog.toml` + Rust/C engine + assets + GPLv2
  `LICENSE`/`NOTICE`). This is both the build source and the catalog input — the single source of
  truth for what the store offers.
- `.github/workflows/publish.yml` — on every push: build the ARM binary from `cog/doom/`, fetch
  FreeDoom, sign the store with gearbox, and deploy to GitHub Pages.

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
# in a gearbox checkout, with the ARM binary + freedoom1.wad staged under ./staged/cogs/arm/…
examples/publish-store.sh \
  --store-id shaal-arcade --name "Shaal Arcade — games for Cognitum" \
  --base-url https://arcade.shaal.dev --key-id shaal-arcade-2026 \
  --seed-file ./arcade-signing.key \
  --cogs-dir ./cog --artifacts-dir ./staged \
  --generated-at "$(date -u +%Y-%m-%dT%H:%M:%SZ)" --out ./public --attest
# then upload ./public/ to your host
```

## Notes

- **Licensing:** the DOOM cog binary is **GPL-2.0** (vendored `doomgeneric`); the FreeDoom IWAD is
  BSD-3-Clause. GPLv2 compliance is satisfied in-repo — the **corresponding source ships right
  here** under `cog/doom/` (with its `LICENSE`/`NOTICE`).
- **Install support:** this produces a valid, signed, verifiable store. A Cognitum **Seed** can
  *install* from it once the device runtime supports custom stores (configurable
  `StoreDescriptor` + `https://` fetch + catalog verify — gearbox protocol §2/§4).
- **Verify it yourself** any time:
  ```sh
  gearbox store-info verify public/store.json
  PUB=$(python3 -c "import json;print(json.load(open('public/store.json'))['keys'][0]['pubkey_b64'])")
  gearbox verify public/app-registry.json --key-id shaal-arcade-2026 --pubkey-b64 "$PUB"
  ```
