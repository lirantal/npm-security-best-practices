<p align="center">

  <h1 align="center">
    Awesome npm Security Best Practices
  </h1>
  <p align="center">
    A curated and practical list of security best practice for using npm packages.
  </p>

<!-- Shields -->
<p align="center">
 <img src="https://cdn.rawgit.com/sindresorhus/awesome/d7305f38d29fed78fa85652e3a63e154dd8e8829/media/badge.svg" alt="Awesome" />
 <img src="https://badgen.net/badge/total%20best%20practices/14/blue" alt="npm security best practices" />
 <img src="https://badgen.net/badge/Last%20Update/Nov%2025/green" />
 <a href="https://www.github.com/lirantal/nodejs-cli-apps-best-practices" target="_blank">
  <img src="https://badgen.net/badge/npm/Security Best Practices/purple" alt="npm Security Best Practices"/>
 </a>
</p>
<!-->

</p>

**Scope**:
- safe-by-default npm package manager command-line options
- hardening against supply chain attacks
- deterministic and secure dependency resolution
- security vulnerabilities scanning and package health signals
- instructions for the `pnpm` and `bun` package managers where applicable

**Context**: Shai-Hulud, Nx and other incidents are a growing concern of supply chain security attacks and compromised npm packages. Follow these developer security best practices around npm, package maintenance and secure local development to mitigate security risks.

---

## Table of Contents

**npm Security Best Practices:**

- 1 [Disable Post-Install Scripts](#1-disable-post-install-scripts)
  - 1.1. [pnpm disable post-install scripts](#11-pnpm-disable-post-install-scripts)
  - 1.2. [Bun disable post-install scripts](#12-bun-disable-post-install-scripts)
  - 1.3. [Run the scripts you need](#13-run-the-scripts-you-need) 
  - 1.4. [pnpm trust policy](#14-pnpm-trust-policy)
- 2 [Install with Cooldown](#2-install-with-cooldown)
  - 2.1. [npm / pnpm / Bun / Yarn minimumReleaseAge cooldown](#21-npm--pnpm--bun--yarn-minimumreleaseage-cooldown)
  - 2.2. [Snyk automated dependency upgrades with cooldown](#22-snyk-automated-dependency-upgrades-with-cooldown)
  - 2.3. [Dependabot automated dependency upgrades with cooldown](#23-dependabot-automated-dependency-upgrades-with-cooldown)
  - 2.4. [Renovate bot automated dependency upgrades with cooldown](#24-renovate-bot-automated-dependency-upgrades-with-cooldown)
- 3 [Harden package installs with security tools](#3-harden-package-installs-with-security-tools)
  - 3.1. [Use npq for hardening package installs](#31-use-npq-for-hardening-package-installs)
  - 3.2. [Use Socket Firewall (sfw) for blocking malicious packages](#32-use-socket-firewall-sfw-for-blocking-malicious-packages)
- 4 [Prevent npm lockfile injection](#4-prevent-npm-lockfile-injection)
- 5 [Use npm ci](#5-use-npm-ci)
- 6 [Avoid blind npm package upgrades](#6-avoid-blind-npm-package-upgrades)

**Secure Local Development Best Practices:**

- 7 [No plaintext secrets in .env files](#7-no-plaintext-secrets-in-env-files)
- 8 [Work in Dev Containers](#8-work-in-dev-containers)

**npm Maintainer Security Best Practices:**

- 9 [Enable 2FA for npm accounts](#9-enable-2fa-for-npm-accounts)
- 10 [Publish with Provenance Attestations](#10-publish-with-provenance-attestations)
- 11 [Publish with OIDC](#11-publish-with-oidc)
- 12 [Reduce your package dependency tree](#12-reduce-your-package-dependency-tree)

**npm Package Health Best Practices:**

- 13 [Consult the Snyk Security Database for package health](#13-consult-the-snyk-security-database-for-package-health)
- 14 [Do not trust the official npmjs.org registry](#14-do-not-trust-the-official-npmjsorg-registry)

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
> $ npm config set ignore-scripts true
> ```
>
> Or disable npm's post-install scripts when performing ad-hoc package install using the command line:
> ```bash
> $ npm install --ignore-scripts <package-name>
> ```

> [!WARNING]
> **Closing the git-dependency loop (npm CLI 11.10.0+)**
>
> Even with `--ignore-scripts`, a git-based dependency URL (e.g. `"pkg": "git+https://github.com/org/pkg"`) can ship its own `.npmrc` file that re-enables lifecycle scripts, effectively bypassing the flag. npm CLI 11.10.0 introduced the `--allow-git` option to control whether git-hosted packages may run lifecycle scripts at all.
>
> Use `--allow-git=none` together with `--ignore-scripts` to fully close this attack vector:
> ```bash
> $ npm install --ignore-scripts --allow-git=none
> ```
>
> To make this the permanent default, add both settings to your global npm configuration:
> ```bash
> $ npm config set ignore-scripts true
> $ npm config set allow-git none
> ```
>
> See the [official npm documentation for `--allow-git`](https://docs.npmjs.com/cli/v11/commands/npm-install#allow-git) for more details.

### 1.1. pnpm disable post-install scripts

Beginning with version 10.0 [pnpm disables postinstall scripts by default](https://pnpm.io/supply-chain-security). pnpm allows an "escape hatch" to re-enable postinstall scripts or set an explicit allow-list of packages that are allowed to run postinstall scripts.

Use `pnpm-workspace.yaml` to control which packages are permitted to run build scripts:

```yaml
# pnpm-workspace.yaml

# Allow only specific packages to run lifecycle scripts (pnpm 10+)
onlyBuiltDependencies:
  - esbuild
  - fsevents
  - nx@21.6.4 || 21.6.5   # pin allowed versions

# Silently block specific packages and suppress the warning
ignoredBuiltDependencies:
  - sharp
```

> [!NOTE]
> As of pnpm 10.26+, `allowBuilds` is the preferred replacement for both `onlyBuiltDependencies` and `ignoredBuiltDependencies` (the earlier settings are deprecated). It provides a single map of package-name → `true`/`false` to allow or deny build scripts:
> ```yaml
> allowBuilds:
>   esbuild: true
>   core-js: false
>   nx@21.6.4 || 21.6.5: true
> ```

To make unreviewed build scripts a hard error (rather than a warning), enable `strictDepBuilds` (pnpm 10.3+):

```yaml
strictDepBuilds: true
```

With `strictDepBuilds: true`, `pnpm install` will exit with a non-zero code if any dependency tries to run a lifecycle script that has not been explicitly allowed, turning silent warnings into CI-blocking failures.

### 1.2. Bun disable post-install scripts

[Bun disables postinstall scripts by default](https://bun.com/docs/install/lifecycle) and maintains its own internal allow-list of packages that are allowed to run postinstall scripts. Bun allows an "escape hatch" to allow postinstall scripts for specific [trusted packages](https://bun.com/docs/install/lifecycle#trusteddependencies) via a `trustedDependencies` field in `package.json`.

### 1.3 Run the scripts you need
Some of the install scripts are there for a reason. If you need to run them, do it in an auditable way and avoid npm trusting the package name in package.json too much.

Use https://www.npmjs.com/package/@lavamoat/allow-scripts
to create an allowlist of specific positions in your dependency graph where scripts are allowed.

### 1.4. pnpm trust policy

pnpm 10.21+ ships a `trustPolicy` setting that detects when a package's **publish-time trust level has decreased** compared to earlier releases — for example, when a package previously published via a Trusted Publisher (OIDC/GitHub Actions) is now published without provenance or signatures. This can be an early signal of an account compromise or supply chain attack.

> [!TIP]
> **Security Best Practice**: Set `trustPolicy: no-downgrade` so that pnpm refuses to install any package version whose trust evidence is weaker than a previously published version of that package.

> [!NOTE]
> **How to implement?**
>
> In `pnpm-workspace.yaml`:
> ```yaml
> # Fail if a package's trust level has decreased (pnpm 10.21+)
> trustPolicy: no-downgrade
>
> # Allow specific packages or versions to bypass the check when needed
> trustPolicyExclude:
>   - 'chokidar@4.0.3'
>   - 'webpack@4.47.0 || 5.102.1'
>
> # Ignore the check for packages published more than 30 days ago (pnpm 10.27+)
> # Useful for older packages that pre-date provenance support
> trustPolicyIgnoreAfter: 43200  # minutes (30 days)
> ```

Trust levels pnpm recognises (strongest → weakest):
1. **Trusted Publisher** – published via a configured Trusted Publisher (e.g. GitHub Actions OIDC)
2. **Provenance** – published with an npm provenance attestation
3. **Signatures** – package registry signature present
4. **No evidence** – no trust signals at all

When `trustPolicy: no-downgrade` is enabled, if any previously published version of a package had a higher trust level than the version being installed, the install is aborted.

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
> Set a persistent minimum release age in npm's configuration so that every `npm install` skips any package version published less than the specified number of days ago:
> ```bash
> $ npm config set min-release-age 3
> ```
>
> Or use the `--before` flag for a one-off install to only consider packages published before a specific date:
> ```bash
> $ npm install express --before=2025-01-01
> ```
>
> Or use shell command evaluation with `--before` to make it dynamic with a 7-day cooldown:
> ```bash
> $ npm install express --before="$(date -v -7d)"
> ```
>
> Note: The `--before` approach requires manual date management and isn't ideal for automated workflows due to hardcoded dates. Prefer `min-release-age` for a persistent configuration.

### 2.1. npm / pnpm / Bun / Yarn minimumReleaseAge cooldown

Configure npm, pnpm, Bun, or Yarn to delay package installations by setting a minimum release age in your package manager's configuration file.

For npm, set `min-release-age` in your `.npmrc` (or via `npm config set`):

```ini
# .npmrc
min-release-age=3
```

Or set it globally so that all projects on your machine benefit:

```bash
$ npm config set min-release-age 3
```

For pnpm 10.16+, use [`pnpm-workspace.yaml`](https://pnpm.io/settings#minimumreleaseageexclude):

```yaml
minimumReleaseAge: 20160  # 2 weeks (in minutes)
# Allow instant upgrades for @types/react and typescript
minimumReleaseAgeExclude:
  - '@types/react'
  - typescript
```

For Bun 1.3+, use [`bunfig.toml`](https://github.com/oven-sh/bun/issues/22679#issuecomment-3371327793):

```yaml
# bunfig.toml
[install]
# Only install package versions published at least 3 days ago
minimumReleaseAge = 259200 # seconds - in #23162 it'll allow "3d" too

# These packages will bypass the 3-day minimum age requirement
minimumReleaseAgeExcludes = ["@types/bun", "typescript"]
```

For Yarn 4.10+, use [`.yarnrc.yml`](https://yarnpkg.com/configuration/yarnrc#npmMinimalAgeGate):

```yaml
# .yarnrc.yml
# Only consider npm package versions published at least 3 days ago
npmMinimalAgeGate: "3d"

# These packages bypass the age gate (package descriptors or glob patterns)
npmPreapprovedPackages:
  - "@types/react"
  - "typescript"
```

These configurations prevent package managers from installing any package version that was published less than the specified time period ago.

### 2.2. Snyk automated dependency upgrades with cooldown

[Snyk automatically includes a built-in cooldown period](https://docs.snyk.io/scan-with-snyk/pull-requests/snyk-pull-or-merge-requests/upgrade-dependencies-with-automatic-prs-upgrade-prs/upgrade-open-source-dependencies-with-automatic-prs#automatic-dependency-upgrade-prs) for dependency upgrade Pull Requests. Snyk does not recommend upgrades to versions that are less than 21 days old to avoid:

- Versions that introduce functional bugs and are subsequently unpublished
- Versions released from compromised accounts where the owner has lost control to malicious actors

### 2.3. Dependabot automated dependency upgrades with cooldown

Dependabot has a [`cooldown`](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/dependabot-options-reference#cooldown-) configuration option, for setting the number of days before a specific version of a dependency will be updated:

> Defines a **cooldown** period for dependency updates, allowing updates to be delayed for a configurable number of days.

### 2.4. Renovate bot automated dependency upgrades with cooldown

Renovate bot has a [`minimumReleaseAge`](https://docs.renovatebot.com/configuration-options/#minimumreleaseage) config option, for setting the minimum age of each package version before a pull request will be created for it:

> Time required before a new release is considered stable.

---

## 3. Harden package installs with security tools

> [!WARNING]
> You should never install npm packages without properly auditing their package health and security signals.

How do you know if an npm package is safe to install? maybe it was just published yesterday? maybe you have an accidental typo in the package name and land on a similarly named malicious package? maybe the package has known vulnerabilities or malicious post-install scripts? Malicious packages can execute arbitrary code during installation, exfiltrate sensitive data, or introduce vulnerabilities into your system without your knowledge.

Installing a new ad-hoc npm package can expose your system to supply chain attacks. Many attacks compromised trusted and popular npm packages, exploit typosquatting, or introduce malicious code in pre/post-install scripts that execute during the installation process.

### 3.1. Use npq for hardening package installs

> [!TIP]
> **Security Best Practice**: Use [npq](https://github.com/lirantal/npq) as a proactive security control that audits npm packages before installation, providing comprehensive security checks, package health signals, and interactive warnings for potentially dangerous or high-risk packages.

> [!NOTE]
> **How to implement?**
> 
> Install `npq` globally to audit packages before installation:
> ```bash
> $ npm install -g npq
> ```
>
> Use npq instead of npm for package installations:
> ```bash
> $ npq install express
> ```
>
> For seamless integration, alias npm to use npq automatically:
> ```bash
> $ alias npm='npq-hero'
> ```
> Note: installing npq provides both `npq` and `npq-hero` commands.
> or add it to your shell profile for persistence:
> ```bash
> $ echo "alias npm='npq-hero'" >> ~/.zshrc  # or ~/.bashrc
> $ source ~/.zshrc
> ```

#### What npq validates

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

#### pnpm and Bun compatibility

npq works with different package managers through environment variables:

```bash
# Use with pnpm
NPQ_PKG_MGR=pnpm npq install fastify

# Use with Bun  
NPQ_PKG_MGR=bun npq install fastify

# Set permanent aliases
alias pnpm="NPQ_PKG_MGR=pnpm npq-hero"
```

#### Advanced usage options

Run security checks without installing packages:
```bash
$ npq install express --dry-run
```

Disable specific security marshalls when needed:
```bash
$ MARSHALL_DISABLE_SNYK=1 npq install express
```

### 3.2. Use Socket Firewall (sfw) for blocking malicious packages

> [!TIP]
> **Security Best Practice**: Use [Socket Firewall (`sfw`)](https://socket.dev/blog/introducing-socket-firewall) as a real-time firewall that intercepts supported package manager commands and blocks packages flagged for malicious behavior, using Socket's deep package analysis and threat intelligence.

> [!NOTE]
> **How to implement?**
>
> Install `sfw` globally:
> ```bash
> $ npm install -g sfw
> ```
>
> Socket Firewall Free runs in wrapper mode. Prefix your package manager command with `sfw`:
> ```bash
> $ sfw npm install express
> ```
>
> Examples with other supported package managers:
> ```bash
> $ sfw pnpm add express
> $ sfw yarn add express
> $ sfw pip install requests
> $ sfw uv pip install flask
> $ sfw cargo fetch
> ```
>
> Socket Firewall will block the package fetch/install if a package is flagged, and prompt you with details so you can make an informed decision.

#### What Socket Firewall checks

Socket Firewall performs deep analysis on packages using Socket's threat intelligence, checking for:

- **Malicious code detection**: Identifies packages that contain known malware or obfuscated code
- **Install script risks**: Flags packages with suspicious pre/post-install scripts
- **Typosquatting detection**: Catches packages with names similar to popular packages
- **Dependency confusion**: Detects potential dependency confusion attacks
- **Known vulnerabilities**: Cross-references CVE databases for disclosed vulnerabilities
- **Protestware and env variable access**: Warns about packages that access environment variables or exhibit protestware behavior
- **Network and filesystem access**: Highlights packages that perform unexpected network or disk operations

#### Comparison with npq

Both `npq` and `sfw` intercept package installations and provide security warnings, but they differ in approach:

| | npq | sfw (Socket Firewall) |
|---|---|---|
| **Analysis method** | Pre-install checks via configurable "marshalls" | Real-time deep package analysis via Socket's platform |
| **Data sources** | Snyk CVE database, npm registry metadata | Socket's proprietary threat intelligence and static analysis |
| **Interactivity** | Interactive prompts before install | Blocks installs and prompts for flagged packages |
| **Package manager support** | npm, pnpm, Bun (via env vars) | npm, yarn, pnpm, pip, uv, cargo |
| **Open source** | Yes | Client is open source; analysis platform is proprietary |

---

## 4. Prevent npm lockfile injection

> [!WARNING]
> Malicious actors can inject compromised packages into your lockfiles through pull requests, potentially compromising your entire application during the next installation.

In September 2019, Liran Tal disclosed security research about inherent security risks with package lockfiles in developer workflows. Both JavaScript package managers, Yarn and npm, were found to be susceptible to lockfile injection attacks.

The security threat occurs when malicious actors gain the ability to contribute source code changes via mechanisms such as pull requests. If they update a lockfile such as `package-lock.json` or `yarn.lock` to include a new npm package dependency, or modify the source URL of an existing package, then any invocation of package install commands would fetch the malicious code.

Furthermore, JavaScript package managers allow users to install packages from unconventional sources, such as GitHub gists or directly from source code repositories. Attackers can update the lockfile to specify a new source location (in the `resolved` key) that they control, and set the SHA512 integrity value accordingly to avoid detection.

> [!TIP]
> **Security Best Practice**: Use [lockfile-lint](https://www.npmjs.com/package/lockfile-lint) to validate that your lockfiles adhere to security policies, ensuring that package sources come from trusted registries and that no malicious modifications have been introduced.

> [!NOTE]
> **How to implement?**
> 
> Install lockfile-lint to validate your lockfiles:
> ```bash
> npm install --save-dev lockfile-lint
> ```
>
> Validate `package-lock.json` with multiple allowed sources:
> ```bash
> npx lockfile-lint --path package-lock.json --type npm --allowed-hosts npm yarn --validate-https
> ```

### Lockfile-lint validation options

The `lockfile-lint` CLI provides comprehensive validation to ensure lockfile integrity:

- **Host validation**: Restrict packages to trusted registry hosts (npm, yarn, verdaccio)
- **HTTPS enforcement**: Ensure all package sources use secure HTTPS protocol
- **Scheme validation**: Control allowed URI schemes (https:, git+https:, git+ssh:)
- **Package name validation**: Verify resolved URLs match declared package names
- **Integrity validation**: Ensure integrity hashes use secure SHA-512 algorithm

### CI/CD integration

Integrate lockfile-lint into your development workflow, such as the following `lint:lockfile` script in `package.json` that runs before every install:

```bash
{
  "scripts": {
    "lint:lockfile": "lockfile-lint --path package-lock.json --type npm --allowed-hosts npm --validate-https",
    "preinstall": "npm run lint:lockfile"
  }
}
```

### pnpm lockfile injection security

pnpm is not susceptible to the same lockfile injection vulnerabilities as npm and yarn because:
- It doesn't maintain tarball sources that can be maliciously modified
- It won't install packages listed in the lockfile that aren't declared in `package.json`
- The `pnpm-lock.yaml` format is more resistant to injection attacks

### pnpm blockExoticSubdeps

Even with a clean lockfile, a transitive dependency can pull in code from an arbitrary git repository or a raw tarball URL — sources that are opaque to typical registry security scanning. pnpm 10.26+ introduces `blockExoticSubdeps` to prevent this.

> [!TIP]
> **Security Best Practice**: Enable `blockExoticSubdeps: true` so that only your direct dependencies (those declared in `package.json`) are allowed to use exotic sources such as git repositories or direct tarball URLs. All transitive dependencies must be resolved from the configured registry, local file paths, workspace links, or trusted GitHub repositories.

> [!NOTE]
> **How to implement?**
>
> In `pnpm-workspace.yaml`:
> ```yaml
> blockExoticSubdeps: true
> ```
>
> Exotic sources that are blocked for transitive dependencies include:
> - Git repositories (`git+ssh://...`, `git+https://...`)
> - Direct URL links to tarballs (`https://.../package.tgz`)
>
> Direct dependencies listed in your root `package.json` are still permitted to use exotic sources.

### Bun lockfile linting

Bun uses its own lockfile format - either `bun.lock` (text-based, default since v1.2) or `bun.lockb` (binary format). Currently, `lockfile-lint` does not support Bun's lockfile formats.

> [!NOTE]
> If you are using Bun as your package manager, `lockfile-lint` is not currently available for validating `bun.lock` or `bun.lockb` files. Users should:
> - Monitor the [lockfile-lint GitHub repository](https://github.com/lirantal/lockfile-lint) for future Bun support
> - Check for alternative security tools that may support Bun lockfiles
> - Follow Bun's security best practices, including using `minimumReleaseAge` cooldown (see [section 2.3](#23-bun-minimumreleaseage-cooldown))

---

## 5. Use npm ci

> [!WARNING]
> Using `npm install` in production can lead to inconsistent installations when lockfiles and package.json files are out of sync, potentially introducing unintended package versions and security vulnerabilities that are resolved during install-time.

Package managers like npm and yarn compensate for inconsistencies between `package.json` and lockfiles by installing different versions than those recorded in the lockfile. This behavior can be hazardous for build and production environments as they could pull in unintended package versions, rendering the entire benefit of lockfile determinism futile. Developers should also favor deterministic package resolution in their local development workflows.

> [!TIP]
> **Security Best Practice**: Use deterministic installation command `npm ci` that enforce strict lockfile adherence, ensuring that only the exact versions specified in the lockfile are installed, and abort installation if inconsistencies are detected.

> [!NOTE]
> **How to implement?**
> 
> Use `npm ci` instead of `npm install` for deterministic installations:
> ```bash
> $ npm ci
> ```
>
> For automated environments like CI/CD, always use the deterministic installation command:
> ```bash
> # In your CI/CD pipeline
> $ npm ci --only=production
> ```
>
> Ensure lockfiles are committed and up-to-date in your repository.

### Yarn, Bun, Deno and pnpm Package manager deterministic installations

Different package managers provide specific commands for enforcing lockfile adherence:

**yarn**: Validate the lockfile (and local cache) did not mutate:
```bash
$ yarn install --immutable --immutable-cache
```

**pnpm**: Use frozen lockfile installation:
```bash
$ pnpm install --frozen-lockfile
```

**Bun**: Use frozen lockfile mode:
```bash
$ bun install --frozen-lockfile
```

**Deno**: Use frozen installation:
```bash
$ deno install --frozen
```

### Lockfile management best practices

Ensure proper lockfile management across your development workflow:

**Commit all lockfiles to version control:**
- `package-lock.json` (npm)
- `pnpm-lock.yaml` (pnpm)  
- `yarn.lock` (yarn)
- `bun.lock` (Bun)
- `deno.lock` (Deno)

---

## 6. Avoid blind npm package upgrades

> [!WARNING]
> Blindly upgrading all dependencies to their latest versions can expose your application to security vulnerabilities, dependency confusion attacks, and malicious packages released from compromised accounts.

Some developers automatically upgrade all dependencies to the latest versions as part of continuous integration processes or local development practices, aiming to ensure forward compatibility or stay at "bleeding edge". Blind dependency upgrades can pull in malicious packages from compromised accounts, introduce functional bugs, or expose applications to supply chain attacks like the colors[^4] and node-ipc[^5] security incidents.

> [!TIP]
> **Security Best Practice**: Use automated dependency management tools with security policies and manual review processes instead of blindly upgrading all packages to their latest versions.

> [!CAUTION]
> **Anti-pattern**:
> Avoid dependency upgrades commands without review:
> ```bash
> $ npm update
> $ npx npm-check-updates -u
> ```

> [!NOTE]
> **How to implement?**
> 
> 1. Use controlled dependency management: `npx npm-check-updates --interactive`
> 2. Use [Snyk Automated Dependency Update PRs](https://docs.snyk.io/scan-with-snyk/pull-requests/snyk-pull-or-merge-requests/upgrade-dependencies-with-automatic-prs-upgrade-prs/upgrade-open-source-dependencies-with-automatic-prs)
> 3. Use [Dependabot Dependency Update PRs](https://docs.github.com/en/code-security/getting-started/dependabot-quickstart-guide)

---

## 7. No plaintext secrets in .env files

> [!WARNING]
> Storing secrets in plaintext environment variables or `.env` files creates a significant security risk, making sensitive data easily accessible to attackers who successfully launch supply chain attacks or gain access to your system.

Environment variables and `.env` files are commonly used to store configuration and sensitive data like API keys, database passwords, and tokens. However, these secrets are stored in plaintext and can be easily exfiltrated by malicious npm packages, compromised dependencies, or attackers who gain access to your development environment.

Even if `.env` files are not committed to version control, they remain vulnerable targets during supply chain attacks where malicious code can read process environment variables or scan the filesystem to locate known configuration files containing secrets.

> [!TIP]
> **Security Best Practice**: Use secrets management solutions that only store references in environment variable data and require additional authentication (like Touch ID on macOS) to access the actual secret values just-in-time.

> [!CAUTION]
> **Anti-pattern**:
> Avoid storing plaintext secrets in `.env` files:
> ```bash
> DATABASE_PASSWORD=my-secret-password
> API_KEY=sk-1234567890abcdef
> ```

> [!NOTE]
> **How to implement?**
>
> Step 1: Use secret references in `.env` files:
> ```bash
> DATABASE_PASSWORD=op://vault/database/password
> API_KEY=infisical://project/env/api-key
> ```
> Step 2: Use the secret manager CLI to inject secrets at runtime:
> ```bash
> $ op run -- npm start
>
> # or more verbosely: 
>
> $ op run --env-file="./.env" -- node --env-file="./.env" server.js
> ```

### Follow-up secure secrets resources

- Liran Tal's [Do Not Use Secrets in Environment Variables](https://www.nodejs-security.com/blog/do-not-use-secrets-in-environment-variables-and-here-is-how-to-do-it-better)
- 1Password's [Secrets Automation with 1Password CLI](https://developer.1password.com/docs/cli/get-started/)
- Infisical's [Getting Started with Infisical CLI](https://infisical.com/blog/stop-using-env-files)

---

## 8. Work in Dev Containers

> [!WARNING]
> Running npm packages directly on your host development machine exposes your entire system to potential malware, allowing malicious packages to access sensitive files, spawning agentic coding CLIs, agent environment variables, and system resources.

[Development containers](https://code.visualstudio.com/docs/devcontainers/containers) (dev containers) provide an isolated, sandboxed environment that limits the blast radius of supply chain attacks. When malicious npm packages execute during installation or runtime, they are confined to the container environment rather than having access to your entire host system where you may have running other projects, sensitive files, or personal data.

> [!TIP]
> **Security Best Practice**: Use dev containers to isolate your project's local development workflows from your host system so that npm package execution and other project development practices are limiting the potential impact of supply chain attacks and malicious package behavior.

> [!NOTE]
> **How to implement?**
> 
> Step 1. Create a `.devcontainer/devcontainer.json` file in your project:
> ```json
> {
>   "name": "Node.js Dev Container",
>   "image": "mcr.microsoft.com/devcontainers/javascript-node:18",
>   "features": {
>     "ghcr.io/devcontainers/features/1password:1": {}
>   },
>   "postCreateCommand": "npm ci"
> }
> ```
>
> Step 2. Use VS Code to open your project in the dev container

### Follow-up resources

- Step-by-step guide on [Setting up Dev Containers and 1Password Secrets for Node.js Local Development](https://www.nodejs-security.com/blog/mitigate-supply-chain-security-with-devcontainers-and-1password-for-nodejs-local-development)
- Consider further hardening of the Dev Container:
```jsonc
  "runArgs": [
    "--security-opt=no-new-privileges:true",
    "--cap-drop=ALL",
    "--cap-add=CHOWN",
    "--cap-add=SETUID",
    "--cap-add=SETGID"
  ],
  "containerEnv": {
    "NODE_OPTIONS": "--disable-proto=delete"
  },
```
- Consider a Custom Dockerfile for enhanced security

---

## 9. Enable 2FA for npm accounts

> [!WARNING]
> npm accounts without two-factor authentication are vulnerable to credential theft and account takeover attacks, potentially allowing malicious actors to publish compromised versions of your packages.

The eslint-scope[^6] incident in 2018 demonstrated the risks of compromised npm accounts when attackers published malicious code after stealing developer credentials. Two-factor authentication provides essential protection against such attacks by requiring additional verification beyond just username and password.

> [!TIP]
> **Security Best Practice**: Enable two-factor authentication on all npm accounts, especially for package maintainers, to prevent unauthorized access and malicious package publications.

> [!NOTE]
> **How to implement?**
> 
> Enable 2FA for authentication and publishing:
> ```bash
> $ npm profile enable-2fa auth-and-writes
> ```
>
> For login and profile changes only:
> ```bash
> $ npm profile enable-2fa auth-only
> ```

---

## 10. Publish with Provenance Attestations

> [!WARNING]
> Packages without provenance attestations cannot be verified for their build origin or authenticity, making it difficult for users to trust the integrity of your published packages in order to determine if they were built from the intended source code on GitHub or by malicious actors who may have compromised your npm account.

Provenance statements provide cryptographic proof of where and how your packages were built, establishing a verifiable link between your source code and published packages. This transparency helps users verify package authenticity and detect tampering.

> [!TIP]
> **Security Best Practice**: Generate provenance attestations for your packages using supported CI/CD platforms to provide users with verifiable build information and enhance supply chain security.

> [!NOTE]
> **How to implement?**
> 
> Publish with provenance in GitHub Actions:
> ```yaml
> permissions:
>   id-token: write
> steps:
>   - run: npm publish --provenance
> ```
>
>
> Note: publishing to npm with provenance requires npm CLI 9.5.0+ and GitHub Actions or GitLab CI/CD with cloud-hosted runners.

---

## 11. Publish with OIDC

> [!WARNING]
> Long-lived npm tokens can be compromised, accidentally exposed in logs, or provide persistent unauthorized access if stolen, posing significant security risks to your packages.

Trusted publishing eliminates the need for long-lived npm tokens by using OpenID Connect (OIDC) authentication from your CI/CD environment. This approach uses short-lived, cryptographically-signed tokens that are specific to your workflow and cannot be extracted or reused. This npm package release method is tightly scoped to only allow publishing from your trusted CI environment (GitHub Actions or GitLab) and your specifically authorized workflow files.

> [!TIP]
> **Security Best Practice**: Configure trusted publishing for your packages to eliminate token-based authentication risks and automatically generate provenance attestations.

> [!NOTE]
> **How to implement?**
> 
> Configure trusted publisher on npmjs.com for your package, then update your CI/CD:
> 
> GitHub Actions:
> ```yaml
> permissions:
>   id-token: write
> steps:
>   - run: npm publish
> ```

Trusted publishing supports GitHub Actions and GitLab CI/CD, and automatically generates provenance attestations which complies with OpenSSF standards.

---

## 12. Reduce your package dependency tree

> [!WARNING]
> Each dependency in your package increases the attack surface and potential for supply chain vulnerabilities, as users inherit all transitive dependencies when installing your package.

Minimizing dependencies reduces security risks, improves performance, and decreases the likelihood of supply chain attacks. Fewer dependencies mean fewer potential points of failure and reduced exposure to malicious packages in the dependency tree.

> [!TIP]
> **Security Best Practice**: Design packages with minimal or zero dependencies by leveraging modern JavaScript features and standard library capabilities instead of external packages.

> [!NOTE]
> **How to implement?**
> 
> Replace common dependencies with native JavaScript:
> ```javascript
> // Instead of lodash
> const unique = [...new Set(array)];
> 
> // Instead of axios for simple requests
> const response = await fetch(url);
> 
> // Instead of utility libraries
> const isEmpty = obj => Object.keys(obj).length === 0;
> ```

Modern JavaScript provides many built-in capabilities that previously required external libraries. Consider the maintenance burden, security implications, and bundle size impact before adding any dependency.

---

## 13. Consult the Snyk Security Database for package health

> [!WARNING]
> Installing npm packages without reviewing their health signals can expose your project to unmaintained, insecure, or low-quality dependencies.

Package health encompasses more than just known vulnerabilities — it includes maintenance activity, community adoption, popularity trends, and security posture. A package that is rarely maintained or has a shrinking community may be at greater risk of future compromise or abandonment.

> [!TIP]
> **Security Best Practice**: Before adopting a new npm package, consult the [Snyk Security Database](https://security.snyk.io) to review its health score, including maintenance, popularity, security, and community signals.

> [!NOTE]
> **How to implement?**
>
> Visit the Snyk Security Database and search for the npm package you are evaluating. For example, here is the health score for `lodash`:
> [https://security.snyk.io/package/npm/lodash](https://security.snyk.io/package/npm/lodash)
>
> The package health page provides:
> - **Security**: known vulnerabilities and CVEs
> - **Popularity**: download trends and adoption metrics
> - **Maintenance**: release frequency and activity
> - **Community**: contributor activity and issue responsiveness

---

## 14. Do not trust the official npmjs.org registry

> [!WARNING]
> The npmjs.org website presents an incomplete and potentially misleading view of npm package metadata, which can create a false sense of security when evaluating packages.

The official [npmjs.org](https://www.npmjs.com) registry website does not represent npm package information in full completeness. For example, the npmjs.org website omits Git and HTTPS-based dependencies even when they are declared in a package's `package.json` file. This means a package may have non-registry dependencies that are invisible to users browsing the registry website.

Furthermore, it has been demonstrated that the source code displayed on the npmjs.org website can drift from the actual tarball that gets installed when running `npm install`. This means the code you review on the website may not be what ends up on your machine.

> [!TIP]
> **Security Best Practice**: Do not rely solely on the npmjs.org website to evaluate a package's dependencies or source code. Always inspect the actual installed package contents and use dedicated security tools to audit packages before use.

> [!NOTE]
> **How to implement?**
>
> Inspect the actual contents of a published tarball before installation:
> ```bash
> $ npm pack <package-name> --dry-run
> ```
>
> Or review the unpacked tarball contents:
> ```bash
> $ npm pack <package-name>
> $ tar -tzf <package-name>-<version>.tgz
> ```
>
> Use [npq](https://github.com/lirantal/npq) (see [section 3.1](#31-use-npq-for-hardening-package-installs)) to audit packages before installation, as it consults multiple security data sources beyond what npmjs.org displays.

---

## Author

**npm Security Best Practices** © [Liran Tal](https://github.com/lirantal), Released under [Apache 2.0](./LICENSE) License.

[^1]: [Shai-Hulud: A Large-Scale Backdoor in the npm Ecosystem](https://snyk.io/blog/embedded-malicious-code-in-tinycolor-and-ngx-bootstrap-releases-on-npm/)
[^2]: [Malicious Code Found in Popular Nx Dev Tool](https://snyk.io/blog/weaponizing-ai-coding-agents-for-malware-in-the-nx-malicious-package/)
[^3]: [Event-Stream Incident Post-Mortem](https://snyk.io/blog/a-post-mortem-of-the-malicious-event-stream-backdoor/)
[^4]: [Colors Package Incident](https://snyk.io/blog/open-source-npm-packages-colors-faker/)
[^5]: [Node-ipc Incident](https://snyk.io/blog/peacenotwar-malicious-npm-node-ipc-package-vulnerability/)
[^6]: [Eslint-scope Incident](https://eslint.org/blog/2018/07/postmortem-for-malicious-package-publishes/)
