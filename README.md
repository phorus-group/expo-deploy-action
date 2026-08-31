# Expo Deploy Action

Composite GitHub Action that triggers an [EAS](https://expo.dev/eas) build + store submission (or a JS-only OTA update) for an Expo app.

Builds run on **Expo's servers** — this action only queues them, so it runs fine on `ubuntu-latest` and finishes in about a minute with the default `no-wait: true`. All signing and store credentials (Android keystore, iOS certificates, App Store Connect API key, Google Play service account) are managed by EAS via `eas credentials`; the only secret this action needs is an Expo access token.

## Prerequisites

- An Expo app already linked to an EAS project (`extra.eas.projectId` in the app config) with build/submit profiles in `eas.json`.
- All credentials bootstrapped on EAS **before** CI runs (`eas credentials` for both platforms) — `--non-interactive` fails hard on any missing credential.
- An [Expo access token](https://docs.expo.dev/accounts/programmatic-access/) (robot token recommended) stored as a secret in the calling repository.
- For `ota: true`: `expo-updates` installed and configured (fingerprint-based build-or-update is a Phase 3 enhancement; the current OTA path runs `eas update --auto`).

## Usage

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 24
          cache: npm

      - run: npm ci

      - name: Deploy to stores
        uses: phorus-group/expo-deploy-action@main
        with:
          expo-token: ${{ secrets.EXPO_TOKEN }}
          platform: all
          profile: production
          auto-submit: true
```

Most callers should prefer the reusable workflow [`phorus-group/workflows/.github/workflows/mobile.expo.yml`](https://github.com/phorus-group/workflows), which wraps this action with checkout, Node setup, dependency install, concurrency control and a GitHub Environment.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `expo-token` | Yes | — | Expo access token used to authenticate the EAS CLI |
| `platform` | No | `all` | Platform to build: `ios`, `android` or `all` |
| `profile` | No | `production` | EAS build profile; with auto-submit, the same-named submit profile is used |
| `auto-submit` | No | `true` | Auto-submit the build to the stores on success |
| `ota` | No | `false` | Publish a JS-only OTA update (`eas update`) instead of a full build |
| `no-wait` | No | `true` | Exit once the build is queued instead of blocking until it finishes |
| `eas-version` | No | `latest` | EAS CLI version to install |
| `working-directory` | No | `.` | Directory containing the Expo app |

## Behavior notes

- **`no-wait: true`** (default) means an EAS build failure will **not** fail the workflow — the runner exits once the build is queued. Set `no-wait: false` to make GitHub the source of truth (the job then blocks for the duration of the build).
- **iOS submissions land in TestFlight.** Submitting for App Store review and releasing remain manual steps in App Store Connect.
- **Android submissions** go to the track configured in the `eas.json` submit profile (e.g. `track: production` with `releaseStatus: completed` rolls out as soon as Google review clears).
- **First release per store cannot be automated** — store listing and the initial production release must be done by hand; use this action from the second release onward.
- Two rapid triggers can race two submissions — callers should set a `concurrency` group (the reusable workflow does).

## License

[Apache 2.0](LICENSE)
