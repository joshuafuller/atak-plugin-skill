# Security

## Reporting

Report a vulnerability privately through GitHub's
[security advisories](https://github.com/joshuafuller/atak-plugin-skill/security/advisories/new)
rather than a public issue. Expect an acknowledgement within a few days.

## What this repository is, and what that implies

This is documentation that gets loaded into an AI agent's context and acted
on. Its content influences what an agent does on a developer's machine — a
machine with `adb` access to a device, a build toolchain, and often
credentials in the environment.

So the content itself is the security surface. Guidance here is expected to be:

- **Measured, not invented.** Every claim states how it was established, so it
  can be re-checked rather than trusted.
- **Free of commands that fetch and execute.** Nothing here should tell a
  reader to pipe a download into a shell, add a package source, or write
  outside the project.
- **Honest about tak.gov.** The ATAK SDK and ATAK builds come from tak.gov and
  nowhere else. Any text suggesting otherwise is a defect, whatever its
  justification.
- **Free of ATAK SDK material.** The TAK licence permits deriving applications
  from the SDK and forbids redistributing it.

If you find content here that breaches any of the above, that is a security
report, not a documentation nit. Please treat it as one.

## Changes are reviewed

`main` is protected and every change arrives by pull request with an explicit
review. There are no repository secrets, and workflows run without write
permissions.
