# Licensing and attribution

This repository is MIT licensed (see [LICENSE](LICENSE)) and contains only our
own work: the skill documentation, a helper script for proposing changes
(`bin/propose`), and CI configuration.

## Not affiliated with the TAK Product Center

ATAK, TAK, and the Android Team Awareness Kit are products of the TAK Product
Center and the U.S. Government, and those names are their marks. This is an
independent, unofficial reference. It is **not affiliated with, endorsed,
sponsored or approved by** the TAK Product Center, the U.S. Army Combat
Capabilities Development Command, or any part of the U.S. Government. Those
names appear only to describe what the documented facts are about.

## No ATAK SDK material is included

The TAK Software License Agreement permits deriving new works and applications
from the SDK and forbids copying, publishing or distributing the SDK itself.
Nothing here is copied from it: no code, no resources, no gradle scripts, no
binaries. Get the SDK yourself from <https://tak.gov>.

CI helps, but does not prove it. The workflow rejects committed binaries and
archives and a list of known SDK filenames; it cannot recognise an SDK text
file that has been renamed. It is a backstop against the common mistake, not a
guarantee.

## How the claims here were established

Everything is measured against a running ATAK-CIV 5.8.0.1 on an Android 14
emulator: observed behaviour, logcat output, file layout, and the contents of
archives inspected with standard tooling. Where a claim restates something the
TAK Product Center publishes — the Third Party Pipeline requirements, for
instance — tak.gov is authoritative and this is a description of their process.

Nothing here is the product of decompiling, disassembling or reverse
engineering ATAK, which the licence forbids. Contributions derived that way are
not accepted, however accurate. See [CONTRIBUTING.md](CONTRIBUTING.md).

## Contributions

By opening a pull request you agree that your contribution is licensed under
the same MIT licence as this project, and that you have the right to submit it.
