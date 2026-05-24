# upload_build_to_testflight

A composite GitHub Action that uploads a **pre-built iOS `.ipa`** to App Store Connect / TestFlight and finishes the post-upload setup:

1. Selects the latest installed Xcode.
2. Uploads the IPA with `xcrun altool` using an App Store Connect API key.
3. Waits for the build to finish processing and appear in App Store Connect.
4. Optionally posts TestFlight "What's New" release notes.
5. Optionally sets the build's encryption compliance flag.

> This action does **not** build or sign your app. Build and sign the IPA in your
> workflow first, then pass its path via `ipa-path`. See [the example workflow](.github/workflows/launch.yaml).

## Usage

```yaml
- name: Upload to TestFlight
  uses: crianpiro/upload_build_to_testflight@v1
  with:
    working-directory: example
    ipa-path: ${{ steps.ipa.outputs.path }}
    app-store-connect-api-key-id: ${{ secrets.ASC_API_KEY_ID }}
    app-store-connect-api-issuer-id: ${{ secrets.ASC_API_ISSUER_ID }}
    app-store-connect-api-key-base64: ${{ secrets.ASC_API_KEY_BASE64 }}
    release-notes: ${{ secrets.RELEASE_NOTES }}
    uses-non-exempt-encryption: "false"
```

The action runs only on **macOS runners** (`runs-on: macos-latest`), since it relies on Xcode / `xcrun altool`.

## Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `ipa-path` | **yes** | — | Path to the pre-built `.ipa` to upload (absolute, or relative to `working-directory`). |
| `app-store-connect-api-key-id` | **yes** | — | App Store Connect API key id (the `kid`). |
| `app-store-connect-api-issuer-id` | **yes** | — | App Store Connect API issuer id. |
| `app-store-connect-api-key-base64` | **yes** | — | Base64 of the App Store Connect API private key (`.p8` contents). |
| `working-directory` | no | `./` | Root of the Flutter app. Every step runs here, and it's the base for relative `ipa-path` values. Used to read `pubspec.yaml` (build number) and `ios/Runner.xcodeproj/project.pbxproj` (bundle id). |
| `release-notes` | no | `""` | TestFlight "What's New" text. Empty skips this step. |
| `locale` | no | `en-US` | Locale used when creating the beta build localization. |
| `uses-non-exempt-encryption` | no | `"false"` | Encryption compliance flag. `"false"` for apps using only exempt encryption, `"true"` otherwise. **Empty skips the step** (e.g. when `ITSAppUsesNonExemptEncryption` is declared in `Info.plist`). |

This action has no outputs.

### How the build is matched

After upload, the action resolves the app by the **bundle id** read from `ios/Runner.xcodeproj/project.pbxproj`, then polls for the build whose version equals the **build number** read from `pubspec.yaml` (the value after `+`, e.g. `1` in `version: 1.0.0+1`). It retries for up to ~15 minutes (30 × 30s) while App Store Connect processes the upload.

## Creating the App Store Connect API key

1. In [App Store Connect → Users and Access → Integrations → App Store Connect API](https://appstoreconnect.apple.com/access/integrations/api), create a key with the **App Manager** role.
2. Note the **Key ID** and **Issuer ID**, and download the `.p8` (downloadable once).
3. Base64-encode the key for use as a secret:
   ```bash
   base64 -i AuthKey_XXXXXXXXXX.p8 | pbcopy
   ```
   Store the result as `ASC_API_KEY_BASE64`, the Key ID as `ASC_API_KEY_ID`, and the Issuer ID as `ASC_API_ISSUER_ID`.

## Example workflow

[`.github/workflows/launch.yaml`](.github/workflows/launch.yaml) demonstrates the full flow against the bundled `example/` app. It builds and signs the IPA with the companion [`crianpiro/build_flutter_app`](https://github.com/marketplace/actions/build-flutter-app) action — which writes the artifact to `<working-directory>/build/ios/ipa/app-release.ipa` — then passes that path to this action.

It expects these repository **secrets**:

| Secret | Purpose |
|--------|---------|
| `P12_BASE64` | Base64 of the signing certificate (`.p12`). |
| `P12_PASSWORD` | Password for the `.p12`. |
| `PROVISIONING_PROFILE_BASE64` | Base64 of the `.mobileprovision`. |
| `RUNNER_KEYCHAIN_PASSWORD` | Any password used for the temporary keychain. |
| `EXPORT_OPTIONS` | Raw Xcode `ExportOptions.plist` contents (XML). |
| `ASC_API_KEY_ID` | App Store Connect API key id. |
| `ASC_API_ISSUER_ID` | App Store Connect API issuer id. |
| `ASC_API_KEY_BASE64` | Base64 of the App Store Connect API key (`.p8`). |
| `RELEASE_NOTES` | *(optional)* "What's New" text for the build. |

## License

BSD 3-Clause — see [LICENSE](LICENSE) © Cristian Andres Picon Rodriguez.
