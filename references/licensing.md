# Licensing: what you may read, run, and publish

Two different licences cover two different artifacts, and conflating them is
what produces both unnecessary caution and real breaches. Read the actual text
rather than trusting this summary — every claim below is quoted from it.

## Where the licences are

| Artifact | Licence | Where to read it |
| --- | --- | --- |
| ATAK-CIV **object code** (`atak.apk`) and the **SDK** | TAK Software License Agreement | `license/LICENSE-atak-civ.txt` at the root of the SDK you downloaded; also presented on <https://tak.gov> before download |
| ATAK-CIV **source** | **GPL-3.0** | `LICENSE.md` in the source tree; <https://github.com/TAK-Product-Center/atak-civ> (public, delayed) and tak.gov (current, permissioned account) |

The SDK ships a whole `license/` directory — the TAK agreement plus third-party
notices for GDAL, ODbL, LGPL and others. Worth a look before publishing
anything that embeds them.

## The binary and the SDK: use it, do not take it apart or hand it on

From `LICENSE-atak-civ.txt`, clause 3 (TAK-CIV object code):

> You are not permitted to sell, **reverse engineer, disassemble, or decompile**
> the TAK-CIV software, or to permit others to do so.

Clause 6 (TAK-SDK):

> ...you are granted a perpetual, non-exclusive, no-charge, royalty-free right
> to use the TAK-SDK **and to derive new works or applications based on the
> TAK-SDK**. You are not permitted to **copy, modify, publish, distribute,
> sublicense, sell, reverse engineer, disassemble, or decompile** the TAK-SDK
> or to permit others to do so.

Two consequences that actually bite:

- **Building a plugin is expressly permitted** — "derive new works or
  applications" is the grant you are working under.
- **Committing SDK files is not.** Scaffolding from `samples/plugintemplate`
  copies SDK material into your tree, and history is what gets published. See
  `references/project-setup.md` for the check.

## The source: read it freely, copy it deliberately

The published source is GPL-3.0. It carries no anti-reverse-engineering term —
the licence exists to guarantee the right to study and modify. So reading it to
understand ATAK's behaviour is exactly what it is for, and is the fastest way
to answer almost any question about the platform.

GPL-3.0 is **copyleft**, so the constraint is on the other end: pasting its code
into a plugin makes that plugin subject to GPL-3.0, which affects how it may be
distributed — including through the Third Party Pipeline. Understand it, then
write your own.

## What is allowed, plainly

Because the difference gets muddled, and being wrong in either direction is
costly:

| Activity | Status |
| --- | --- |
| Reading the published GPL-3.0 source | **Fine** — that is what publication is for |
| Running ATAK and observing behaviour, logcat, file layout | **Fine** |
| Listing an archive's entries (`unzip -l atak.apk`) | **Fine** — no code is read |
| Reflection at runtime over classes loaded in ATAK's process | **Fine** — inspecting a live object graph from inside the process; nothing is decompiled |
| Building and distributing a plugin derived from the SDK | **Fine** — expressly granted |
| Decompiling or disassembling `atak.apk`, the SDK jars or AARs | **Not permitted** |
| Republishing SDK files — AARs, keystore, template, espresso archives | **Not permitted** |
| Copying GPL-3.0 source into a plugin | Permitted, but makes the plugin GPL-3.0 |

Note the fourth row. Runtime reflection is often loosely described as "reverse
engineering" and is not the thing the licence prohibits: no object code is
transformed back into source. It is, however, a poor way to *learn* an API —
see `reading-the-source.md`.

## If you record a finding about ATAK's internals

State how it was established. "Observed in logcat", "read in
`AtakMapController.java`", "returned by reflection at runtime" are all
defensible and all re-checkable. A finding that can only be phrased in terms of
obfuscated symbol names from a disassembly is not, and does not belong in this
skill.
