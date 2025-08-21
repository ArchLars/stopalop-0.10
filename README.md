# StopAlop, README plan (essentials plus deps, updates, removal)

## Project goal

Make StopAlop a daily driver package manager for a modern LFS install, while keeping the spirit of the original script: simple, transparent, easy to audit, minimal runtime deps, and friendly to hand-rolled workflows. The LFS book emphasizes tracking installs, careful handling of config files, and easy removal or upgrade, we align with that. ([linuxfromscratch.org][1])

## Philosophy

* Keep the core tiny, scriptable, and readable (single small CLI that delegates to standard tools).
* Prefer plain text for human inspection, and SQLite for speed and atomicity.
* Always do staged installs into `DESTDIR`, then commit to the live root in one step.
* Make every transaction explain itself before it runs, with a dry-run plan.
* Add smart features in thin layers, not hidden magic.

---

## On-disk layout

```
/var/lib/stopalop/db.sqlite              (authoritative DB, ACID transactions)
/var/lib/stopalop/pkgs/<pkg>.manifest    (human-readable manifest with checksums and metadata)
/var/cache/stopalop/packages/            (signed package archives)
/etc/stopalop/trustedkeys.gpg            (keyring for verification)
/var/log/stopalop/transactions.log       (append-only log)
```

Rationale, follow FHS for state in `/var/lib`, logs in `/var/log`, and caches in `/var/cache`. RPM uses `/var/lib/rpm` for its DB, which is consistent with this layout. ([refspecs.linuxfoundation.org][2], [Red Hat Docs][3])

---

## Package database (minimal spec)

Each installed package entry records:

* name, version, release, arch, build date
* dependency hints (depends, conflicts, provides, replaces, optional)
* full file manifest with mode, uid, gid, mtime, SHA-256, xattrs, ACLs, SELinux label
* install reason (explicit or dependency), install transaction id

Storage: keep a human-readable manifest per package, plus a compact SQLite index for fast queries and crash-safe commits. ([refspecs.linuxfoundation.org][2])

---

## Package archives and metadata

* Use `tar` with POSIX pax format.
* Create archives with all relevant metadata: `--format=pax --xattrs --acls --selinux --numeric-owner`.
* Extract as root with `--xattrs --acls --selinux --same-owner`.
* File capabilities live in the `security.capability` xattr, capture and restore these. ([GNU][4], [Man7][5])

---

## Build workflow

* Always build into a clean staging root `DESTDIR`, then commit to `/`.
* You can use `fakeroot` while packing to record ownership and modes without real root, then let install apply ownership. This mirrors common LFS guidance and community practice. ([linuxfromscratch.org][1], [Reddit][6])

---

## Dependency model

StopAlop implements a pragmatic dependency layer that stays true to the script’s spirit, but learns from established ecosystems.

### Relationships

Supported fields in package metadata:

* `Depends` (hard runtime deps)
* `Recommends` and `Suggests` (soft deps, not enforced by default)
* `Conflicts` and `Breaks` (mutual exclusivity)
* `Provides` (virtual capability names)
* `Replaces` or `Obsoletes` (supersede or rename)
  These mirror Debian and RPM relationship models, which gives us predictable semantics. ([Debian][7], [jfearn.fedorapeople.org][8], [Red Hat Docs][9])

### Resolver

* Default: a straightforward resolver that topologically sorts dependency graphs and rejects conflicts.
* Advanced option: integrate a SAT solver backend via libsolv for complex scenarios, as used by DNF and others. This stays optional so the core remains light. ([Fedora Project][10], [GitHub][11], [LWN.net][12])

### Virtuals

* Packages can `Provide` capabilities, for example `sh` or `mpi-impl`, so dependencies can be expressed against features rather than package names. This follows Debian and RPM patterns. ([Debian][7], [Red Hat Docs][9])

### Pinning and holds

* Support hold and ignore lists similar to pacman’s `HoldPkg` and `IgnorePkg`, so users can freeze or skip updates for selected packages. ([Arch Linux Manual Pages][13])

---

## Updates

* Plan phase: solver computes a consistent target set, prints a summary with reasons for each action (upgrade, install, replace), and highlights conflicts to resolve.
* Execution phase: verify signatures, download archives, stage to a temporary root, run pre-commit checks, then atomically apply files, update the DB, and run triggers.
* Obsoletes and replaces are honored, for example a renamed package takes over the old one. ([Red Hat Docs][9])

### Config files on upgrade

* Config files are flagged in the manifest.
* On upgrade use a three-way merge, or install `.new` and keep the admin-edited file. The LFS book stresses preserving local configuration. ([linuxfromscratch.org][1])

---

## Removal

* Safe remove: uninstall the target package, refuse if it would break another package that depends on it, unless the user confirms a recursive removal.
* Recursive remove: remove the target and all reverse dependencies (like pacman’s cascade or recursive modes) with a very loud confirmation step. ([Arch Linux Manual Pages][14], [Unix & Linux Stack Exchange][15])

### Orphans and autoremove

* Track install reason, explicit or dependency.
* `autoremove` proposes to remove packages that were installed as dependencies and are no longer required. Show a plan first. Popular managers implement this pattern, and users are warned to review the plan carefully. ([Ask Ubuntu][16], [Manjaro Linux Forum][17])

### Purge

* `purge` removes the package plus its config files and any package-owned state, similar to `dpkg --purge`. Only remove files tracked in the manifest, and never touch user data outside the manifest. ([Man7][18])

---

## Post-install and post-remove triggers

Run once per transaction, not per file:

* `ldconfig` when new shared libraries are installed.
* Icon cache and desktop database updates: `gtk4-update-icon-cache`, `update-desktop-database`.
* GLib and GIO: `gio-querymodules`, `glib-compile-schemas`.
* MIME database updates when relevant.
* SELinux relabel: `restorecon` or `setfiles` if policy is present.
  Keep triggers declarative and idempotent, inspired by Debian maintainer scripts and RPM scriptlets. ([Man7][5], [Debian][7])

---

## Verification and trust

* Sign packages and repository metadata with GPG.
* Verify signature and checksums before installation or upgrade, similar to RPM and APT based systems. ([Red Hat Docs][9], [Debian][19])

---

## Reproducible builds

* Record toolchain versions, source hashes, flags, and environment.
* Honor `SOURCE_DATE_EPOCH` in builds and archives to normalize timestamps, which is a cornerstone of reproducible builds work. ([linuxfromscratch.org][1])

---

## CLI, first cut

```
stopalop build    <name> <version> --recipe ./pkg.toml --dest ./out/
stopalop sign     ./out/<pkg>.tar.zst --key ./keys/privkey.gpg
stopalop verify   ./cache/<pkg>.tar.zst --keyring /etc/stopalop/trustedkeys.gpg
stopalop install  <pkg> [--yes] [--dry-run]
stopalop remove   <pkg> [--recursive] [--purge] [--yes] [--dry-run]
stopalop upgrade  [<pkg>...] [--hold <name>] [--ignore <name>] [--dry-run]
stopalop autoremove [--yes] [--dry-run]
stopalop query    --owns /path/file  |  --list <pkg>  |  --deps <pkg>  |  --rdeps <pkg>
stopalop check    --verify-files <pkg>  |  --verify-all
```

---

## Defaults that keep the original spirit

* Single small binary or script, no mandatory heavyweight daemons.
* Uses system `tar` and core utilities, optional libsolv backend for advanced dependency solving.
* Text manifests you can read in a pager, one SQLite file for speed and atomicity.
* No hidden state, everything lives under `/var/lib/stopalop` and `/var/log/stopalop`.
* Always support dry-run with a clear, colorized plan.

---

## Milestones

1. **DB and manifests**, write and read full manifests, hashes, and xattrs, plus a minimal SQLite index. ([GNU][20])
2. **Install and remove**, staged extract with metadata, safe remove with reverse dependency checks, recursive remove behind an extra prompt. ([Arch Linux Manual Pages][14])
3. **Config handling and triggers**, mark configs, preserve admin edits, add `ldconfig`, icon, desktop, GIO, GSettings, MIME, SELinux triggers. ([Man7][5], [Debian][7])
4. **Verification**, package and repo signing, enforce verify by default. ([Red Hat Docs][9])
5. **Dependency solver**, start with simple topological resolver, add optional libsolv backend for complex cases. ([GitHub][11])
6. **Quality of life**, holds and ignores, autoremove with review, clear transaction logs. ([Arch Linux Manual Pages][13], [Ask Ubuntu][16])

---

## References

* LFS package management guidance on tracking files and preserving config. ([linuxfromscratch.org][1])
* FHS for directory layout, examples of RPM DB in `/var/lib/rpm`. ([refspecs.linuxfoundation.org][2], [Red Hat Docs][3])
* GNU tar docs for xattrs and pax format, file capabilities as xattrs. ([GNU][20], [Man7][5])
* Debian relationships and policy, plus dpkg purge semantics. ([Debian][19], [Man7][18])
* RPM provides, conflicts, obsoletes, and update behavior. ([Red Hat Docs][9], [jfearn.fedorapeople.org][8])
* libsolv and SAT solving in modern resolvers, DNF uses libsolv. ([GitHub][11], [Fedora Project][10], [LWN.net][12])
* Pacman holds and removal modes for comparison. ([Arch Linux Manual Pages][13])

---


[1]: https://www.linuxfromscratch.org/lfs/view/development/chapter08/pkgmgt.html?utm_source=chatgpt.com "8.2. Package Management"
[2]: https://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html?utm_source=chatgpt.com "Filesystem Hierarchy Standard"
[3]: https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/4/html/reference_guide/s1-filesystem-fhs?utm_source=chatgpt.com "3.2. Overview of File System Hierarchy Standard (FHS)"
[4]: https://www.gnu.org/s/tar/manual/html_section/create-options.html?utm_source=chatgpt.com "GNU tar 1.35: 4.3 Options Used by --create"
[5]: https://man7.org/linux/man-pages/man7/capabilities.7.html?utm_source=chatgpt.com "capabilities(7) - Linux manual page"
[6]: https://www.reddit.com/r/linuxfromscratch/comments/10rt5uj/how_to_package_manager/?utm_source=chatgpt.com "How to package manager? : r/linuxfromscratch"
[7]: https://www.debian.org/doc/debian-policy/ch-relationships.html?utm_source=chatgpt.com "7. Declaring relationships between packages"
[8]: https://jfearn.fedorapeople.org/en-US/RPM/4/html/RPM_Guide/ch-dependencies.html?utm_source=chatgpt.com "Chapter 5. Package Dependencies"
[9]: https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html-single/rpm_packaging_guide/index?utm_source=chatgpt.com "RPM Packaging Guide | Red Hat Enterprise Linux | 7"
[10]: https://fedoraproject.org/wiki/Features/DNF?utm_source=chatgpt.com "Features/DNF - Fedora Project Wiki"
[11]: https://github.com/openSUSE/libsolv?utm_source=chatgpt.com "openSUSE/libsolv: Library for solving packages and ..."
[12]: https://lwn.net/Articles/503581/?utm_source=chatgpt.com "DNF, which may or may not replace Yum"
[13]: https://man.archlinux.org/man/pacman.conf.5.en?utm_source=chatgpt.com "pacman.conf(5)"
[14]: https://man.archlinux.org/man/pactrans.1.en.txt?utm_source=chatgpt.com "pactrans.1.en.txt"
[15]: https://unix.stackexchange.com/questions/16769/arch-linux-with-xfce-desktop-no-longer-starts?utm_source=chatgpt.com "Arch Linux with Xfce-Desktop no longer starts"
[16]: https://askubuntu.com/questions/943287/how-to-find-reason-for-orphaned-packages-in-apt-get-autoremove?utm_source=chatgpt.com "How to find reason for orphaned packages in apt-get ..."
[17]: https://forum.manjaro.org/t/pamac-incorrectly-removes-unrequired-dependencies-on-fresh-installation/136154?utm_source=chatgpt.com "Pamac incorrectly removes \"unrequired\" dependencies on ..."
[18]: https://man7.org/linux/man-pages/man1/dpkg.1.html?utm_source=chatgpt.com "dpkg(1) - Linux manual page"
[19]: https://www.debian.org/doc/debian-policy/?utm_source=chatgpt.com "Debian Policy Manual"
[20]: https://www.gnu.org/software/tar/manual/html_node/Extended-File-Attributes.html?utm_source=chatgpt.com "GNU tar 1.35: 4.3.2 Extended File Attributes"
