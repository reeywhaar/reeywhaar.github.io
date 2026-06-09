---
title: "Notarizing macOS Binaries in GitHub Actions"
date: 2026-06-09T00:00:00+00:00
draft: false
tags:
  - Development
  - Unix
summary: How to configure a GitHub Actions release pipeline that signs and notarizes macOS binaries using Apple's notarytool.
---

This post walks through the GitHub Actions release workflow I use for [diff-tgz](https://github.com/Reeywhaar/diff-tgz) — a Rust CLI tool distributed via Homebrew. The pipeline builds for four targets (macOS arm64/x86_64, Linux arm64/x86_64), signs and notarizes the macOS binaries, publishes a GitHub release, and bumps the Homebrew tap formula automatically.

The full workflow file lives at [`.github/workflows/release.yml`](https://github.com/Reeywhaar/diff-tgz/blob/main/.github/workflows/release.yml) and is triggered on any tag push matching `*.*.*`.

## Secrets you need to configure

Before the workflow can run you have to add seven secrets in your repository settings (`Settings → Secrets and variables → Actions`).

### `APPLE_ID`

Your Apple ID email address — the one associated with your Apple Developer account.

### `DISTRIBUTION_CERT_BASE_64`

A base64-encoded Developer ID Application certificate exported from Xcode.

To get it:

1. Open **Xcode → Settings → Accounts**, select your Apple ID and click **Manage Certificates**.
2. Right-click your **Developer ID Application** certificate and choose **Export Certificate…**.
3. Save it as a `.p12` file and choose a strong export password (this becomes `DISTRIBUTION_CERT_PASS`).
4. Base64-encode the file:

```sh
base64 -i certificate.p12 | pbcopy
```

Paste the result as the secret value.

### `DISTRIBUTION_CERT_PASS`

The password you chose when exporting the `.p12` file above.

### `NOTARY_TOOL_PASS`

An **app-specific password** for your Apple ID. Create one at [appleid.apple.com](https://appleid.apple.com) under **Sign-In and Security → App-Specific Passwords**. This is what `xcrun notarytool` uses to authenticate with Apple's notarization service.

### `SIGNING_IDENTITY`

The full name of your signing identity as it appears in the keychain. Run this locally to find it:

```sh
security find-identity -v -p codesigning
```

It comes in the form `Developer ID Application: Your Name (CERTID)`. Copy the whole string including the certificate type prefix.

### `TEAM_ID`

Your 10-character Apple Developer Team ID. You can find it in [App Store Connect](https://appstoreconnect.apple.com) under **Users and Access → Keys**, or at [developer.apple.com/account](https://developer.apple.com/account) in the **Membership** section.

### `TAP_REPO_TOKEN`

A GitHub **Personal Access Token** with `contents: write` access scoped to your Homebrew tap repository. The workflow uses it to check out the tap repo and push the updated formula after each release. Create one at **GitHub → Settings → Developer settings → Personal access tokens → [Fine-grained tokens](https://github.com/settings/personal-access-tokens)** and grant it write access to the tap repository only.

## How the workflow uses them

### Setting up the certificate keychain

```yaml
- name: Configure certificates
  if: runner.os == 'macOS'
  run: >
    echo $DISTRIBUTION_CERT_BASE_64 | base64 --decode > cert.p12 &&
    security create-keychain -p $KEYCHAIN_PASS $KEYCHAIN &&
    security default-keychain -s ~/Library/Keychains/$KEYCHAIN-db &&
    security set-keychain-settings $KEYCHAIN &&
    security list-keychains -s $KEYCHAIN &&
    security unlock-keychain -p $KEYCHAIN_PASS $KEYCHAIN &&
    security import ./cert.p12 -k $KEYCHAIN -P $DISTRIBUTION_CERT_PASS -A -T /usr/bin/codesign -T /usr/bin/security &&
    security set-key-partition-list -S apple-tool:,apple: -s -k $KEYCHAIN_PASS $KEYCHAIN &&
    security find-identity -p codesigning -v
  env:
    KEYCHAIN: "def.keychain"
    KEYCHAIN_PASS: "hmmmm" # whatever, throwaway keychain
    DISTRIBUTION_CERT_BASE_64: ${{ secrets.DISTRIBUTION_CERT_BASE_64 }}
    DISTRIBUTION_CERT_PASS: ${{ secrets.DISTRIBUTION_CERT_PASS }}
```

This step decodes the base64 certificate back to a `.p12` file, creates a temporary keychain, imports the certificate into it, and grants `codesign` access to the key without a GUI prompt. The keychain password (`KEYCHAIN_PASS`) is just a local ephemeral value used within this job — it doesn't need to be a secret.

### Storing notarytool credentials

```yaml
- name: Configure notarytool
  if: runner.os == 'macOS'
  run: >
    xcrun notarytool store-credentials notarytool
    --apple-id $APPLE_ID
    --team-id $TEAM_ID
    --password $NOTARY_TOOL_PASS
  env:
    APPLE_ID: ${{ secrets.APPLE_ID }}
    NOTARY_TOOL_PASS: ${{ secrets.NOTARY_TOOL_PASS }}
    TEAM_ID: ${{ secrets.TEAM_ID }}
```

`notarytool store-credentials` saves the credentials under the profile name `notarytool` in the keychain. Later steps reference this profile by name so the credentials never appear in plain text in the command line.

### Signing and notarizing

```yaml
- name: Sign binary
  if: runner.os == 'macOS'
  run: >
    codesign -s "$SIGNING_IDENTITY" --deep -v -f -o runtime
    target/${{ matrix.target }}/release/diff-tgz
  env:
    SIGNING_IDENTITY: ${{ secrets.SIGNING_IDENTITY }}

- name: Notarize binary
  if: runner.os == 'macOS'
  run: |
    zip notary.zip target/${{ matrix.target }}/release/diff-tgz
    xcrun notarytool submit notary.zip --keychain-profile notarytool --wait
```

`codesign` uses the `--options runtime` flag (`-o runtime`), which is required by Apple for notarization. The binary is then zipped — notarytool requires an archive — and submitted. The `--wait` flag blocks until Apple returns a verdict, making it easy to catch failures in the CI log.

### Bumping the Homebrew tap

The `bump_homebrew` job runs after the GitHub release is created. It checks out the tap repository using `TAP_REPO_TOKEN`, computes SHA-256 checksums for all four release archives, runs a Ruby script to update the formula, and pushes the commit back:

```yaml
- uses: actions/checkout@v6
  with:
    ref: main
    path: tap
    repository: Reeywhaar/homebrew-tap
    token: ${{ secrets.TAP_REPO_TOKEN }}
```

Without `TAP_REPO_TOKEN` the checkout would use the default `GITHUB_TOKEN`, which only has access to the current repository.

## Summary

| Secret                      | Where to get it                               |
| --------------------------- | --------------------------------------------- |
| `APPLE_ID`                  | Your Apple ID email                           |
| `DISTRIBUTION_CERT_BASE_64` | Export from Xcode, `base64 -i cert.p12`       |
| `DISTRIBUTION_CERT_PASS`    | Password chosen during export                 |
| `NOTARY_TOOL_PASS`          | App-specific password from appleid.apple.com  |
| `SIGNING_IDENTITY`          | `security find-identity -v -p codesigning`    |
| `TEAM_ID`                   | App Store Connect → Users and Access → Keys   |
| `TAP_REPO_TOKEN`            | GitHub PAT with `contents: write` on tap repo |
