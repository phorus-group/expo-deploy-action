# Expo Deploy Action

Composite GitHub Action that runs **one** [EAS](https://expo.dev/eas) `build` **or** `submit` for **one** platform, and **waits** for it to finish — so the job's pass/fail reflects the real EAS result, not just that the request was queued.

It's designed to be called from separate jobs (build-android, submit-android, build-ios, submit-ios) so each stage shows up as its own status check. See the [`mobile.expo.yml`](https://github.com/phorus-group/workflows) reusable workflow for the full 4-job pipeline.

Builds run on Expo's servers; all signing and store credentials live on EAS (`eas credentials`). The only secret this action needs is an Expo access token.

## Usage

Build one platform, capture the build id, then submit it:

```yaml
jobs:
  build-android:
    runs-on: ubuntu-latest
    outputs:
      build-id: ${{ steps.build.outputs.build-id }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 24 }
      - id: build
        uses: phorus-group/expo-deploy-action@main
        with:
          expo-token: ${{ secrets.EXPO_TOKEN }}
          command: build
          platform: android
          profile: production

  submit-android:
    needs: build-android
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 24 }
      - uses: phorus-group/expo-deploy-action@main
        with:
          expo-token: ${{ secrets.EXPO_TOKEN }}
          command: submit
          platform: android
          profile: production
          build-id: ${{ needs.build-android.outputs.build-id }}
```

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `expo-token` | Yes | — | Expo access token used to authenticate the EAS CLI |
| `command` | No | `build` | `build` or `submit` |
| `platform` | Yes | — | `android` or `ios` (single platform) |
| `profile` | No | `production` | EAS build/submit profile from `eas.json` |
| `build-id` | No | — | EAS build id to submit (required when `command: submit`) |
| `eas-version` | No | `latest` | EAS CLI version to install |
| `working-directory` | No | `.` | Directory containing the Expo app |

## Outputs

| Output | Description |
|---|---|
| `build-id` | The EAS build id produced by a `build` command (empty for `submit`) — pass it to a downstream `submit` job |

## Behavior notes

- **Both commands wait for EAS.** `build` polls until the EAS build finishes and fails the step if the build status isn't `FINISHED`. `submit` waits for the store upload and fails if it's rejected. Expect build jobs to run 15–40 min (billed as cheap Linux minutes; the actual build runs on EAS).
- **iOS submissions land in TestFlight.** Submitting for App Store review and releasing remain manual steps in App Store Connect.
- **Android submissions** go to the track configured in the `eas.json` submit profile (e.g. `track: production`, `releaseStatus: completed`).
- **First release per store cannot be automated** — use this from the second release onward.

## License

[Apache 2.0](LICENSE)
