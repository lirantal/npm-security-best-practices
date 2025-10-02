# Awesome npm Security Best Practices ![Awesome](https://cdn.rawgit.com/sindresorhus/awesome/d7305f38d29fed78fa85652e3a63e154dd8e8829/media/badge.svg)

A curated and practical list of security best practice for using npm packages.

Scope:
- safe-by-default npm package manager command-line options
- hardening against supply chain attacks
- deterministic and secure dependency resolution
- security vulnerabilities scanning and package health signals
- instructions for the `pnpm` and `bun` package managers where applicable

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

Recent attacks like Shai-Hulud[^1], Nx[^2] and long-standing attacks like event-stream[^3] have all leveraged npm `postinstall` scripts to execute arbitrary code on a developer's machine during package installation in order to exfiltrate sensitive data, trigger a worm-like propagation, or perform other malicious activities.

> [!TIP]
> **Security Best Practice**: Configure npm to disable lifecycle scripts when installing packages so any npm package, direct or indirect, cannot execute arbitrary code or commands on your system during installation.

> [!NOTE]
> **How to implement?**
> 
> It is *highly recommended* at a global configuration to set npm's `ignore-scripts` configuration to `true` to disable all post-install scripts for all projects on your machine:
> ```bash
> npm config set ignore-scripts true
> ```
>
> Or disable npm's post-install scripts when performing ad-hoc package install using the command line:
> ```bash
> npm install --ignore-scripts <package-name>
> ```

### 📦 pnpm disable post-install scripts

Beginning with version 10.0 [pnpm disables postinstall scripts by default](https://pnpm.io/supply-chain-security). pnpm allows an "escape hatch" to re-enable postinstall scripts or set an explicit allow-list of packages that are allowed to run postinstall scripts.

### 📦 Bun disable post-install scripts

[Bun disables postinstall scripts by default](https://bun.com/docs/install/lifecycle) and maintains its own internal allow-list of packages that are allowed to run postinstall scripts. Bun allows an "escape hatch" to allow postinstall scripts for specific [trusted packages](https://bun.com/docs/install/lifecycle#trusteddependencies) via a `trustedDependencies` field in `package.json`.

---

## Author

**npm Security Best Practices** © [Liran Tal](https://github.com/lirantal), Released under [Apache 2.0](./LICENSE) License.

[^1]: [Shai-Hulud: A Large-Scale Backdoor in the npm Ecosystem](https://snyk.io/blog/embedded-malicious-code-in-tinycolor-and-ngx-bootstrap-releases-on-npm/)
[^2]: [Malicious Code Found in Popular Nx Dev Tool](https://snyk.io/blog/weaponizing-ai-coding-agents-for-malware-in-the-nx-malicious-package/)
[^3]: [Event-Stream Incident Post-Mortem](https://snyk.io/blog/a-post-mortem-of-the-malicious-event-stream-backdoor/)
