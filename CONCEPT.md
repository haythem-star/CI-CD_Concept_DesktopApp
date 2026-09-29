# CI/CD Concept – Contoso.DesktopApp (.NET Windows desktop application)

**one build pipeline on every push, one publish pipeline on version tags**.

## 1. Overall flow

```
build.yml    (every push / PR) : restore ─► build ─► NUnit unit + integration ─► Sonar ─► vulnerability audit
publish.yml  (tag vX.Y.Z)      : build ─► UI tests (interactive agent) ─► package & sign ─► [approval] ─► publish
```

## 2. Key design decisions

- **Two pipelines** `build.yml` gives fast feedback on every change. `publish.yml` produces a release. Separate triggers, permissions and secrets, and no release logic in the everyday pipeline.
- **Version comes from the git tag** (GitVersion): a release is an explicit, auditable action, and the version is baked into binaries and installer.
- **UI tests only in the release pipeline.** They are slow and need the scarce interactive agent. 
- **Release is built once.** The `app-binaries` artifact is what UI tests validate, what gets signed and what goes into the installer.
- **Plain CLI steps** (`dotnet build/test/publish`) easy to reproduce locally. Tasks are used only for tooling (test publishing, artifacts, Azure CLI).

## 3. Branching, PRs, releases

- **Trunk-based:** short-lived branches → PR → `main`. Branch policy: build validation (`build.yml`), ≥1 reviewer, resolved comments.
- **Release = tag `vX.Y.Z` on `main`.** Tag creation is restricted to maintainers. If a shipped version needs a hotfix.

## 4. Build, test, packaging, artifacts

- Restore via internal feed (`NuGetAuthenticate`); .NET SDK and tools pinned (`global.json`, tool manifest with GitVersion, SonarScanner, dotnet-coverage); self-hosted agents with `clean: all`.
- **NUnit tests are split by category:** unit + integration in `build.yml` (results in the Tests tab, coverage sent to Sonar); `UI` tests on the dedicated agent.
- **Interactive agent:** auto-logon session, no lock screen, fixed resolution, run as a process, not a service. The job checks `UserInteractive`, kills leftover app processes and uploads screenshots on failure.
- **Artifacts:** `app-binaries`, `ui-tests`, `installer` (signed MSI + SHA-256). Published to storage in an immutable versioned folder plus a `latest` copy.
- **Quality/security:** Sonar analysis, NuGet vulnerability audit (warning now; could block later).

## 5. Secrets, permissions, service connections

- Secrets sit in **Key Vault-backed variable groups**, mapped to steps via `env:`; never in YAML or logs.
- The **signing certificate never leaves Key Vault**: the pipeline can sign, not export.
- Azure access uses a **service connection with workload identity federation (OIDC)**, no stored credentials, scoped to one storage account.
- The `desktop-release` **environment requires approval** before publishing. Signing and storage secrets exist only in the release group, so PR builds cannot reach them.
