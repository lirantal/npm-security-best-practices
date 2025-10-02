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
  - 1.1. [📦 pnpm disable post-install scripts](#11--pnpm-disable-post-install-scripts)
  - 1.2. [📦 Bun disable post-install scripts](#12--bun-disable-post-install-scripts)
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

By disabling post-install scripts, you can mitigate the risk of such attacks by preventing the execution of potentially harmful code during the installation process.

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

### 1.1. 📦 pnpm disable post-install scripts

Beginning with version 10.0 [pnpm disables postinstall scripts by default](https://pnpm.io/supply-chain-security). pnpm allows an "escape hatch" to re-enable postinstall scripts or set an explicit allow-list of packages that are allowed to run postinstall scripts.

### 1.2. 📦 Bun disable post-install scripts

[Bun disables postinstall scripts by default](https://bun.com/docs/install/lifecycle) and maintains its own internal allow-list of packages that are allowed to run postinstall scripts. Bun allows an "escape hatch" to allow postinstall scripts for specific [trusted packages](https://bun.com/docs/install/lifecycle#trusteddependencies) via a `trustedDependencies` field in `package.json`.

---

## 2. Install with Cooldown

> [!WARNING]
> Newly released packages and versions may contain malicious code that are often-times quickly picked up by the community in matter of hours or days and subsequently unpublished.

Attackers build on the npm versioning and publishing model which prefers and resolves to latest semvar ranges to employ attacks by publishing new versions of packages. By implementing a "cooldown" period before installing or upgrading to new package versions, you reduce the risk of installing compromised packages that may be quickly discovered and removed from the registry.

> [!TIP]
> **Security Best Practice**: Configure your package manager to delay installations of recently published packages, allowing time for the community to discover and report potential security issues or functional problems.

> [!NOTE]
> **How to implement?**
> 
> Use npm's `--before` flag to install packages only if they were published before a specific date:
> ```bash
> npm install express --before=2025-01-01
> ```
>
> Or use shell command evaluation to make it dynamic with a 7-day cooldown:
> ```bash
> npm install express --before="$(date -v -7d)"
> ```
>
> Note: This approach requires manual date management and isn't ideal for automated workflows due to hardcoded dates.

### 2.2. 📦 pnpm `minimumReleaseAge` cooldown

Configure pnpm to delay package installations by setting a minimum release age in your repository's pnpm configuration file `pnpm-workspace.yaml`:

```yaml
minimumReleaseAge: 20160  # 2 weeks (in minutes)
```

This configuration prevents pnpm from installing any package version that was published less than the specified time period ago.

### 2.3. Snyk automated dependency upgrades with cooldown

[Snyk automatically includes a built-in cooldown period](https://docs.snyk.io/scan-with-snyk/pull-requests/snyk-pull-or-merge-requests/upgrade-dependencies-with-automatic-prs-upgrade-prs/upgrade-open-source-dependencies-with-automatic-prs#automatic-dependency-upgrade-prs) for dependency upgrade Pull Requests. Snyk does not recommend upgrades to versions that are less than 21 days old to avoid:

- Versions that introduce functional bugs and are subsequently unpublished
- Versions released from compromised accounts where the owner has lost control to malicious actors

---

## 3. Use npq for hardening package installs

> [!WARNING]
> You should never install npm packages without properly auditing their package health and security signals.

How do you know if an npm package is safe to install? maybe it was just published yesterday? maybe you have an accidental typo in the package name and land on a similarly named malicious package? maybe the package has known vulnerabilities or malicious post-install scripts? Malicious packages can execute arbitrary code during installation, exfiltrate sensitive data, or introduce vulnerabilities into your system without your knowledge.

Installing a new ad-hoc npm package can expose your system to supply chain attacks. Many attacks compromised trusted and popular npm packages, exploit typosquatting, or introduce malicious code in pre/post-install scripts that execute during the installation process.

> [!TIP]
> **Security Best Practice**: Use [npq](https://github.com/lirantal/npq) as a proactive security control that audits npm packages before installation, providing comprehensive security checks, package health signals, and interactive warnings for potentially dangerous or high-risk packages.

> [!NOTE]
> **How to implement?**
> 
> Install `npq` globally to audit packages before installation:
> ```bash
> npm install -g npq
> ```
>
> Use npq instead of npm for package installations:
> ```bash
> npq install express
> ```
>
> For seamless integration, alias npm to use npq automatically:
> ```bash
> alias npm='npq-hero'
> ```
> Note: installing npq provides both `npq` and `npq-hero` commands.
> or add it to your shell profile for persistence:
> ```bash
> echo "alias npm='npq-hero'" >> ~/.zshrc  # or ~/.bashrc
> source ~/.zshrc
> ```

### What npq validates

npq performs comprehensive security audits using "marshalls" - specialized security validators that check for:

- **Vulnerability scanning**: Consults Snyk's database for known CVE vulnerabilities
- **Package age analysis**: Flags packages less than 22 days old (new package detection) 
- **Typosquatting detection**: Identifies packages with names similar to popular packages
- **Registry signature verification**: Validates npm registry signatures using published keys
- **Provenance attestation**: Verifies package build provenance metadata
- **Pre/post-install scripts**: Warns about potentially malicious installation scripts
- **Package health indicators**: Checks for README, LICENSE, repository URL, and download metrics
- **Version maturity**: Flags package versions published less than 7 days ago
- **Binary introduction**: Warns when new command-line binaries are added
- **Deprecation status**: Alerts for deprecated packages
- **Maintainer domain validation**: Checks for expired domains in maintainer emails

### 📦 pnpm and Bun compatibility

npq works with different package managers through environment variables:

```bash
# Use with pnpm
NPQ_PKG_MGR=pnpm npq install fastify

# Use with Bun  
NPQ_PKG_MGR=bun npq install fastify

# Set permanent aliases
alias pnpm="NPQ_PKG_MGR=pnpm npq-hero"
```

### Advanced usage options

Run security checks without installing packages:
```bash
npq install express --dry-run
```

Disable specific security marshalls when needed:
```bash
MARSHALL_DISABLE_SNYK=1 npq install express
```

---

## Author

**npm Security Best Practices** © [Liran Tal](https://github.com/lirantal), Released under [Apache 2.0](./LICENSE) License.

[^1]: [Shai-Hulud: A Large-Scale Backdoor in the npm Ecosystem](https://snyk.io/blog/embedded-malicious-code-in-tinycolor-and-ngx-bootstrap-releases-on-npm/)
[^2]: [Malicious Code Found in Popular Nx Dev Tool](https://snyk.io/blog/weaponizing-ai-coding-agents-for-malware-in-the-nx-malicious-package/)
[^3]: [Event-Stream Incident Post-Mortem](https://snyk.io/blog/a-post-mortem-of-the-malicious-event-stream-backdoor/)
