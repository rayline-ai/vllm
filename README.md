# arc-patches

The ARC vLLM patch stack as files, byte-for-byte the assets of this repository's `arc-v*` releases:

- `patches/arc-v0.33.0.patch`: branch `arc/v0.33.0` (commit dca0ed16e564ab048405888952fd86f56ef9335a) as one patch
  on vLLM v0.33.0; release `arc-v0.33.0-1`; sha256 91d39ed57058f9ef53ad1967af99424db7b75080a1c072ab6f29ada0d50c6103.

Deployments read a file at a pinned commit of this branch (`git show <commit>:patches/<file>`) and check it against
the plugin's `engine.lock`. Files are only added, never rewritten.

What the patches change, and how to rebase them onto a new vLLM: `ARC-PATCHES.md` on branch `arc/v0.33.0`. Reproduce a
file from its release tag: `git format-patch --stdout v0.33.0..arc-v0.33.0-1 > arc-v0.33.0.patch`.
