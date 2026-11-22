# Phone tables (`magnetite-tables`)

A phone cannot prepare Horizon's resources: `horizon-prepare` runs Mojang's own exporters on a
JDK, and iOS forbids child processes while Android has no JDK
([`docs/unification/mobile-boot.md`](../../docs/unification/mobile-boot.md) §2). So a PC prepares
them once and publishes everything that is **not Mojang's** as a release here. The phone downloads
that release plus the client jar's `assets/` and `data/` and the asset index's objects from
Mojang's own servers, after a sign-in — see `crates/horizon-client/src/game_files.rs`.

`tables.yml` is this repository's workflow, kept in Magnetite for review; it is copied here as
`.github/workflows/tables.yml`. The two scripts it runs live in Magnetite's `tools/`:
`fetch_minecraft.py` and `make_tables.py`.

## Running it

Actions → **tables** → *Run workflow*, with `magnetite_ref` (a branch, tag or commit; `main` by
default). One run does:

1. builds `horizon-prepare` from Magnetite at that ref (Java 25, Rust stable);
2. `fetch_minecraft.py mc` — the pinned 26.3 `version.json`, `client.jar`, `server.jar`,
   `libraries/`, the asset index and its objects, each checked against `research-lock.json`'s
   SHA-1s (the version JSON may be any of the republished ones; only its asset index differs);
3. `horizon-prepare ... --output prepared --complete` — the whole resource folder, including the
   sounds, and `--check` must then pass;
4. `make_tables.py prepared mc tables.zip` — classifies every file of the folder, writes
   `setup.json` (with `copies`: Mojang's records the phone copies from its own jar, such as the
   structure templates) and the rest as `tables/…`, and proves the archive holds no Mojang bytes;
5. publishes a release `tables-<short sha>` in this repository with
   `magnetite-tables-26.3-<short sha>.zip`, and prints the asset's URL and SHA-256 in the run's
   summary.

Set those two values in Magnetite as repository variables:

```
MAGNETITE_TABLES_URL=https://github.com/izrikmizrik/magnetite-tables/releases/download/tables-<sha>/magnetite-tables-26.3-<sha>.zip
MAGNETITE_TABLES_SHA256=<the zip's sha256>
```

A build without them offers no download (`game_files.rs`, `spec_from_build`).

## What is published, and why no Mojang files

The zip holds `setup.json` and `tables/<path>` for every file of the prepared folder that is not
Mojang's: the exporters' tables (`facts.json`, `damage-types.json`, `items.json`, `registries.json`,
`creative-inventory.json`, the data-fix tables…), the world-generation bundles (`worldgen/`), the
block behaviour files, and the `horizon-resources.json` marker. `setup.json` names the client jar
and the asset index the phone downloads itself, with their URLs, SHA-1s and the exact jar members
to extract — no version manifest is fetched on the phone.

Mojang's files never enter the archive, and `make_tables.py` proves it by content, not by name: no
file of the archive has the SHA-1 of any `client.jar` member or any asset-index object (files of
64 bytes or less are exempt). It also refuses anything under `assets/` or `data/` the install would
not write at that path, so the tables cannot shadow Mojang's assets.

One consequence worth knowing: the prepared `worldgen/structure/**/*.nbt` are byte copies of jar
members under other names. They are Mojang's bytes, so they stay out of the zip (and cannot be
named in `jar_files`, which the installer applies at the member's own path). Worlds on a phone
therefore have no structure templates until that path is settled; the tables' generation data is
unaffected.

## Nobody uploads anything but the zip

There is no account, no cookie and no upload step beyond the release asset. `fetch_minecraft.py`
reads https from Mojang's four hosts and writes only into its own folder; the workflow's other
steps build, prepare, classify and publish that one zip. Retries, resume by hash and the eight
download lanes are described at the top of both scripts.
