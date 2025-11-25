# timelock_npm_test

Test project demonstrating [timelock-npm-registry](https://github.com/pyoner/timelock-npm-registry) with Yarn 1.x.

## What is timelock-npm-registry?

A proxy npm registry that adds supply chain attack protection by requiring packages to be at least N minutes old before installation. This delays newly published (potentially malicious) versions.

## Configuration

### `.yarnrc` (Yarn 1.x format)
```
registry "https://timelock-npm-registry.dev/lock/1440/"
```

### `.npmrc` (for compatibility)
```
registry=https://timelock-npm-registry.dev/lock/1440/
```

The `1440` means packages must be at least 24 hours (1440 minutes) old.

## Usage

```bash
yarn install
```

Yarn will fetch packages through the timelock proxy, which enforces the minimum age requirement.

## Verification

```bash
# Check registry is configured
yarn config get registry
# Output: https://timelock-npm-registry.dev/lock/1440/
```

## Debug Mode

Use `--verbose` to see exactly where packages are fetched from:

```bash
# Clear cache first to see full flow
yarn cache clean lodash
yarn install --verbose
```

Example output showing the two-step fetch process:

```
# 1. Package METADATA is fetched through timelock proxy (checks publish age)
verbose 0.120 Performing "GET" request to "https://timelock-npm-registry.dev/lock/1440/lodash".
verbose 0.250 Request "https://timelock-npm-registry.dev/lock/1440/lodash" finished with status code 200.

# 2. Package TARBALL is downloaded directly from npm registry
verbose 0.256 Performing "GET" request to "https://registry.npmjs.org/lodash/-/lodash-4.17.21.tgz".
```

### How it works

1. **Metadata request** → timelock proxy checks if the package version is old enough (24h+)
2. **Tarball download** → actual package contents come directly from `registry.npmjs.org`

The timelock proxy acts as a gatekeeper for package metadata, but doesn't proxy the actual package downloads.

## Security Considerations

### Trust Model

When using any third-party registry proxy (including timelock), you're trusting it to return legitimate metadata. If the proxy were compromised, it could:

1. Modify the `tarball` URL to point to a malicious server
2. Modify the `integrity` hash to match the malicious tarball

**The integrity hash alone doesn't protect you** - it's provided by the same source as the tarball URL.

### How yarn.lock Protects You

Once you commit `yarn.lock`, it contains hardcoded:
- `resolved` - the exact tarball URL (e.g., `registry.npmjs.org`)
- `integrity` - the SHA-512 hash of the tarball

```
lodash@^4.17.21:
  version "4.17.21"
  resolved "https://registry.npmjs.org/lodash/-/lodash-4.17.21.tgz#679591c564c3bffaae8454cf0b3df370c3d6911c"
  integrity sha512-v2kDEe57lecTulaDIuNTPy3Ry4gLGJ6Z1O3vE1krgXZNrsQ+LFTGHVxVjcXPs17LhbZVGedAJv8XZ1tvj5FvSg==
```

Yarn uses these values directly for locked packages - **it doesn't ask timelock again**.

### Verification Steps

Before committing changes to `yarn.lock`, verify all tarballs come from `registry.npmjs.org`:

```bash
# Check all resolved URLs in lockfile
grep "resolved" yarn.lock

# Fail if any non-npmjs URLs are found
grep "resolved" yarn.lock | grep -v "registry.npmjs.org" && echo "WARNING: Non-npmjs tarball found!" || echo "✓ All tarballs from npmjs.org"
```

### CI Protection

Use `--frozen-lockfile` in CI to prevent lockfile changes:

```bash
yarn install --frozen-lockfile
```

This fails if any package would need to be resolved fresh, ensuring only pre-verified lockfile entries are used.

### Recommended Workflow

1. **When adding/updating packages locally:**
   ```bash
   yarn add <package>
   # Review the new yarn.lock entries
   grep "resolved" yarn.lock | grep -v "registry.npmjs.org"
   # If empty, safe to commit
   ```

2. **In CI/CD:**
   ```bash
   yarn install --frozen-lockfile
   ```

3. **Periodic audit:**
   ```bash
   # Verify no lockfile entries point to unexpected registries
   grep "resolved" yarn.lock | grep -v "registry.npmjs.org"
   ```
