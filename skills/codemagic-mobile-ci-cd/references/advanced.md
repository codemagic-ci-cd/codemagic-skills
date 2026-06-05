# Advanced Topics (Link Index)

Consult when phases 0–5 are done or the user explicitly asks. Do not block onboarding on these.

## Wiring CI into the dev cycle

Add after first manual build succeeds. Not part of Phase 0 onboarding.

| Topic | When | Link |
|-------|------|------|
| Webhooks + `triggering:` | Auto-build on push/PR — yaml `triggering:` and **Create webhook** in app settings | https://docs.codemagic.io/yaml-running-builds/webhooks/ |
| Branch/PR triggers | Filter by branch, PR, or changeset | https://docs.codemagic.io/yaml-running-builds/starting-builds-automatically/ |
| Scheduling | Cron-style builds | https://docs.codemagic.io/yaml-running-builds/scheduling/ |

## Post-release CI/CD

| Topic | When | Link |
|-------|------|------|
| Testing in CI | User wants unit/integration tests in workflow | https://docs.codemagic.io/yaml-testing/testing/ |
| Firebase Test Lab | Android device testing | https://docs.codemagic.io/yaml-testing/firebase-test-lab/ |
| Caching | Speed up repeat builds | https://docs.codemagic.io/knowledge-codemagic/caching/ |

## Alternative distribution

| Topic | When | Link |
|-------|------|------|
| Firebase App Distribution | User names Firebase instead of TestFlight | https://docs.codemagic.io/yaml-distributing/firebase-app-distribution/ |
| Tester groups | Codemagic-native tester distribution | https://docs.codemagic.io/yaml-distributing/tester-groups/ |
| Build dashboards | Share artifacts (no logs) | https://docs.codemagic.io/yaml-distributing/build-dashboards/ |

## Repo patterns

| Topic | When | Link |
|-------|------|------|
| Monorepo (deep) | Multi-app repo, complex path filters | https://docs.codemagic.io/knowledge-others/monorepo-apps/ |
| White-label | Customer-specific app generation | https://docs.codemagic.io/yaml-quick-start/white-label-getting-started/ |

Monorepo basics (`working_directory`, `changeset`) are in [connect-and-yaml-basics.md](connect-and-yaml-basics.md).

## Integrations

Link to specific integration when user names a tool: https://docs.codemagic.io/integrations/

Common: Fastlane, Maestro, SonarQube, Slack notifications.

## Other stacks (out of core skill scope)

| Stack | Link |
|-------|------|
| Unity | https://docs.codemagic.io/yaml-quick-start/building-a-unity-app/ |
| KMM | https://docs.codemagic.io/yaml-quick-start/building-a-kmm-app/ |
| .NET MAUI | https://docs.codemagic.io/yaml-quick-start/building-a-dotnet-maui-app/ |
| Ionic / Cordova | https://docs.codemagic.io/yaml-quick-start/building-an-ionic-app/ |

## Related skills

| Topic | Skill |
|-------|-------|
| CodePush OTA | `codemagic-codepush` (not docs) |

## Account and API

| Topic | Link |
|-------|------|
| Billing | https://docs.codemagic.io/billing/ |
| macOS/Xcode specs | https://docs.codemagic.io/specs-macos/ |
| REST API v3 schema | https://codemagic.io/api/v3/schema |
| Migrate from Bitrise | https://docs.codemagic.io/yaml-quick-start/migrating-from-bitrise/ |
| Migrate from App Center | https://docs.codemagic.io/yaml-quick-start/migrating-from-app-center/ |

## YAML reference

Full configuration: https://docs.codemagic.io/yaml-basic-configuration/yaml-getting-started/
Cheatsheet: https://docs.codemagic.io/codemagic-yaml-cheatsheet.html
