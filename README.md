# Awesome npm Security Best Practices ![Awesome](https://cdn.rawgit.com/sindresorhus/awesome/d7305f38d29fed78fa85652e3a63e154dd8e8829/media/badge.svg)

A curated and practical list of security best practice for using npm packages.

Scope:
- safe-by-default npm package manager command-line options
- hardening against supply chain attacks
- deterministic and secure dependency resolution
- security vulnerabilities scanning and package health signals

---

## Table of Contents

- 1 [Disable Post-Install Scripts](#1-disable-post-install-scripts)
- 2 [Install with Cooldown](#2-install-with-cooldown)
  - 2.1. [npm before flag cooldown](#21-npm-before-flag-cooldown)
  - 2.2. [pnpm minimumReleaseAge cooldown](#22-pnpm-minimumreleaseage-cooldown)
  - 2.3. [Snyk automated dependency upgrades with cooldown](#23-snyk-automated-dependency-upgrades-with-cooldown)
- 3 [Use npq for hardening package installs](#3-use-npq-for-hardening-package-installs)
- 4 [Use npm ci](#4-use-npm-ci)

---

## 1. Disable Post-Install Scripts

> [!WARNING]
> Post-install scripts are a common and recurring attack vector for supply chain attacks.
>

> [!TIP]
> **Security Best Practice**: Configure npm to disable lifecycle scripts when installing packages so any npm package, direct or indirect, cannot execute arbitrary code or commands on your system during installation.

> [!NOTE]
> **How to implement?**
> 
> Disable npm's post-install scripts when performing ad-hoc package install using the command line:
> ```bash
> npm install --ignore-scripts <package-name>
> ```
> Or, *highly recommended* at a global configuration, set npm's `ignore-scripts` configuration to `true` to disable all post-install scripts.
> ```bash
> npm config set ignore-scripts true
> ```


## Author

**npm Security Best Practices** © [Liran Tal](https://github.com/lirantal), Released under [Apache 2.0](./LICENSE) License.