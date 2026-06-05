# Connect and YAML Basics (Phase 0)

## Steps

1. **Account** — User needs a Codemagic account. Guide to sign up or log in if missing.
2. **Connect repo** — Link GitHub, GitLab, or Bitbucket. Confirm Codemagic has repository access.
3. **Add application** — Codemagic UI → **Add application** → select repo → confirm project type.
4. **Create `codemagic.yaml`** at repo root (exact name, not `.yml`).

Validate:

- At least one workflow under `workflows:`
- Scripts do not rely on `cd` across separate blocks

5. **Detect config** — App settings → select branch → **Check for configuration file**.
6. **Start first build** — Codemagic UI → **Start new build** → select workflow (manual). Wire CI into the dev cycle later — `triggering:` in yaml + webhook in app settings; see [advanced.md](advanced.md).

Full guide: https://docs.codemagic.io/yaml-basic-configuration/yaml-getting-started/

Adding apps: https://docs.codemagic.io/getting-started/adding-apps/

## Monorepo snippet

If the app lives in a subfolder:

```yaml
workflows:
  my-workflow:
    working_directory: path/to/app
```

Deep dive: https://docs.codemagic.io/knowledge-others/monorepo-apps/

## Credential gates (before Phase 2)

Do not add signing or publishing blocks until prerequisites exist:

**iOS** — user completes UI steps via the guide; agent adds yaml and signing scripts:

[Signing iOS apps](https://docs.codemagic.io/yaml-code-signing/signing-ios/) — user: API key, certificate, provisioning profile in Codemagic UI; agent: `ios_signing`, `integrations`, `xcode-project use-profiles`

- [ ] Credentials uploaded in Code signing identities / Team integrations (user, via guide)
- [ ] App record in App Store Connect (first version often uploaded manually)

**Android** — user completes UI steps via the guide; agent wires Gradle and yaml:

[Signing Android apps](https://docs.codemagic.io/yaml-code-signing/signing-android/) — user: keystore generate/upload in Codemagic UI; agent: `build.gradle` `CI=true` signing + `android_signing` in yaml

- [ ] Keystore uploaded in Code signing identities (user, via guide)
- [ ] Google Play service account JSON as secret `GOOGLE_PLAY_SERVICE_ACCOUNT_CREDENTIALS` (for Play publish — phase 4+)

If credentials are missing, link the relevant signing guide and walk the user through it step by step. Do not substitute a shortened checklist:

- iOS — [Signing iOS apps](https://docs.codemagic.io/yaml-code-signing/signing-ios/)
- Android — [Signing Android apps](https://docs.codemagic.io/yaml-code-signing/signing-android/)

## YAML schema

IDE validation: https://docs.codemagic.io/yaml-basic-configuration/yaml-getting-started/#validating-codemagic-yaml

On failure, see [common-failures.md](common-failures.md).
