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
