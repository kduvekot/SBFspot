# Reproducibility notes — Linux release pipeline

Short companion to [`.github/workflows/linux-release.md`](../.github/workflows/linux-release.md)
focused on the evidence that the pipeline's output is reproducible,
plus the canonical hashes from the reference rebuild of V3.9.12.

## Summary

Given a fixed git tag, the pipeline produces byte-identical tarballs
run after run. This has been demonstrated by rebuilding the same
tag twice, minutes apart, and verifying every one of the 30
artefacts (15 main tarballs + 15 debug sidecar tarballs) has an
identical SHA256 across both runs.

This property is guaranteed by construction — not luck — via:

- A single `SOURCE_DATE_EPOCH` derived from the tag commit's
  timestamp, propagated to every build tool in the environment.
- Deterministic tar flags (`--sort=name`, zero uid/gid,
  `--format=ustar`, `--mtime=@$SOURCE_DATE_EPOCH`).
- Deterministic gzip (`gzip -n`, no mtime or filename in header).
- Pinned Debian snapshot URL for arm64 cells.

See [`linux-release.md §9`](../.github/workflows/linux-release.md#9-reproducibility-guarantees)
for the full mechanism description and the known limitation on
arm cells (live Raspbian archive — long-term drift modulo
security updates).

## Canonical SHA256s — reference V3.9.12 rebuild

The pipeline was exercised against the V3.9.12 tag during
development on a contributor's fork. All 15 cells succeeded, and
both runs produced the same tarball SHA256s — demonstrating
reproducibility end-to-end.

**Pipeline input:** `SBFspot/SBFspot` @ `V3.9.12`
(commit `2d9d3b191edc23b08b1ac4545961132d79b53c69`,
commit timestamp `1739908502` = 2025-02-18 19:55:02 UTC).

**Main tarballs (15):**

```
fd82b1179bc38a41bf799ac4cc9e487ce05217185a1cb9847c4e5c8ff8227676  sbfspot-mariadb-arm-linux-bookworm.tar.gz
12504c342b4e640b5588bed5d215b2c82e2028f985b3806e417a0fcebec05b5f  sbfspot-mariadb-arm-linux-bullseye.tar.gz
9f09b1f92e9905489ed2dd8dbf3523cc565c606b07fa28bc7588e1a45c14ef09  sbfspot-mariadb-arm-linux-buster.tar.gz
42362ffff188d93ad120b029c9f2ea3a3dfea055dfe32e2188a429187e2390ad  sbfspot-mariadb-arm64-linux-bookworm.tar.gz
3a923f5a70e8d683aa38880660987a6a8bfb405206455b9276fe26777bfca1fd  sbfspot-mariadb-arm64-linux-bullseye.tar.gz
6e0cdda5aaec81fb3bfa25e3fa9ea5f4951c550d252faba661c1ae05d9c6000c  sbfspot-nosql-arm-linux-bookworm.tar.gz
789e45f5d3a1fd88fdb6c7dc9c164d268c1225e66c7886d3bd08daefc1bbad11  sbfspot-nosql-arm-linux-bullseye.tar.gz
5525908c5b3b8f47847597291b02fad33cfff7a76629fbcffd0e3551f585058f  sbfspot-nosql-arm-linux-buster.tar.gz
338486335d166e1a4224bf2c4af4ed404c7692bef94c5c906580bdea6358b384  sbfspot-nosql-arm64-linux-bookworm.tar.gz
6e2ebc26e6525eaebf7f549b4b38320f4999dfc6efcd92501e6a793c26b51991  sbfspot-nosql-arm64-linux-bullseye.tar.gz
07abe22168489a0b512cd4a82861ab035146fdd4267586a4546d74ad2feda67a  sbfspot-sqlite-arm-linux-bookworm.tar.gz
7b90be7ab591dbff099b28183fe001550fe4ad31b6512eb6df24e3daaa315013  sbfspot-sqlite-arm-linux-bullseye.tar.gz
b545b61bf5b7d0c0946cbdc2102202b28c0e164a08424936550d5815c60c61b9  sbfspot-sqlite-arm-linux-buster.tar.gz
d6b2d4a628185d26fd2d2221084f54442534a9bd9411f387fdee8b589e2dc46d  sbfspot-sqlite-arm64-linux-bookworm.tar.gz
9500cbe87a145e68c3eacdb18d397e90f6b88f49e929781442a4586b65eabaf4  sbfspot-sqlite-arm64-linux-bullseye.tar.gz
```

**Debug sidecar tarballs (15):**

```
d21eb22c4682b16a7b6bbd24330230bfd4eb8cdf86e12f0f0a1668d7cee9ffc2  sbfspot-mariadb-arm-linux-bookworm.debug.tar.gz
06e0cbb2eec335b1bfa8e87225dc595a2d38ff20c15436a0a1cad4c2d6f10cf7  sbfspot-mariadb-arm-linux-bullseye.debug.tar.gz
e5cab3c01e174610498b40aba600684b4beebf122d276523e993514033963212  sbfspot-mariadb-arm-linux-buster.debug.tar.gz
75edf6ccee5882d0d3ac016023d1dda92e4615f5027d02c305377260fadcc567  sbfspot-mariadb-arm64-linux-bookworm.debug.tar.gz
04cb6d952969bbb5417a0172f089a986f63ff54f1aa18d8ad23abaf6c5d79d56  sbfspot-mariadb-arm64-linux-bullseye.debug.tar.gz
b953b36b62de47fe3eaa2a79c304fa7d3017587f54255f13adaf38bb180b7355  sbfspot-nosql-arm-linux-bookworm.debug.tar.gz
1cd6e440836a92ce6959108693d387139a28e8d1883ce893fda5b27bcf0d591f  sbfspot-nosql-arm-linux-bullseye.debug.tar.gz
7b5d0e06325f0e914f375c63d3b55c2b15af203f8c20c747f20ee0a2b3521174  sbfspot-nosql-arm-linux-buster.debug.tar.gz
21693fea81d607d2b21e9fc535b18c18a4be488420406debe0328a7fde5656bd  sbfspot-nosql-arm64-linux-bookworm.debug.tar.gz
a33c0d9e116860d935ba8fcc8fa175ce7f9d7a25f0378ffcf63ecbdbfaa90e14  sbfspot-nosql-arm64-linux-bullseye.debug.tar.gz
85c41be9f40150d86465754d4ad86d81b602e14f243e24373151b4c3e128754f  sbfspot-sqlite-arm-linux-bookworm.debug.tar.gz
23f0ed1bd72f8e1d30a91fa5b24fecd26cc95c1a902c3406746b366f1205e61c  sbfspot-sqlite-arm-linux-bullseye.debug.tar.gz
8442b32dde76dc1c4d95d709c902cbf54dffa31817bce4e588bea5a9723e7cdc  sbfspot-sqlite-arm-linux-buster.debug.tar.gz
21eb5b278810c970fb0ec15554ddf846feff190e91d92c060db8807dc9cacf52  sbfspot-sqlite-arm64-linux-bookworm.debug.tar.gz
3645360b006062f416d89176a3cc81eff206c733a93e6e08bb1e45a5837f4ab7  sbfspot-sqlite-arm64-linux-bullseye.debug.tar.gz
```

### Caveat on these exact hashes

These hashes came from the reference rebuild during development.
If someone re-runs the pipeline on V3.9.12 today, they will
likely see **small drift** on the 9 arm cells — because Raspbian's
live archive has probably pushed library updates since the
reference rebuild date (see
[`linux-release.md §9 — Known limitation`](../.github/workflows/linux-release.md#known-limitation-live-raspbian-archive)).

The 6 arm64 cells will still match exactly, because those are
built against the pinned Debian snapshot.

The reproducibility guarantee is:

- **Within a single workflow run's lifetime** (two dispatches,
  minutes apart): all 30 hashes byte-identical.
- **Across time**: the 6 arm64 hashes are stable; the 9 arm
  hashes drift with Raspbian security updates.

For the release-attached tarballs on GitHub Releases, the hashes
above can be archived in the release notes to document what was
shipped for that tag.

## How to measure reproducibility yourself

```sh
# Trigger the workflow twice, grab the run IDs, then:
gh run download --repo SBFspot/SBFspot <run-id-A> -p 'sbfspot-*'
mv sbfspot-* run-A
gh run download --repo SBFspot/SBFspot <run-id-B> -p 'sbfspot-*'
mv sbfspot-* run-B

( cd run-A && find . -name '*.tar.gz' -exec sha256sum {} + ) | sort > /tmp/A.sha
( cd run-B && find . -name '*.tar.gz' -exec sha256sum {} + ) | sort > /tmp/B.sha
diff /tmp/A.sha /tmp/B.sha
```

Empty diff → clean reproducibility across those two runs.
