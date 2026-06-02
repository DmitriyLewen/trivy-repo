---
name: deploy-packages-to-trivy-repo
description: Move the deploy-packages job from trivy/release.yaml into a standalone workflow in trivy-repo and trigger it via `gh workflow run`. Use when refactoring the Trivy release pipeline to isolate rpm/deb deployment from the main release workflow.
model: inherit
---

# Move Deploy-Packages Job to trivy-repo

Refactor the Trivy release pipeline by extracting the `deploy-packages` job from
`trivy/.github/workflows/release.yaml` into a new `workflow_dispatch` workflow at
`trivy-repo/.github/workflows/deploy-packages.yml`. The trivy repo then triggers it
via `gh workflow run`, passing the release version as an input. Script logic from
`ci/deploy-rpm.sh` and `ci/deploy-deb.sh` is inlined directly into the new workflow
(no external script files needed — the workflow is self-contained).

## Reference Files

Read these files before writing any code:

| File | URL | Purpose |
|------|-----|---------|
| Current `deploy-packages` job | https://raw.githubusercontent.com/aquasecurity/trivy/main/.github/workflows/release.yaml | Source job to extract; also shows how other repos are triggered via `gh workflow run` |
| RPM deploy script | https://raw.githubusercontent.com/aquasecurity/trivy/main/ci/deploy-rpm.sh | Logic to inline into the new workflow |
| DEB deploy script | https://raw.githubusercontent.com/aquasecurity/trivy/main/ci/deploy-deb.sh | Logic to inline into the new workflow |
| `trivy-downloads` example | https://raw.githubusercontent.com/aquasecurity/trivy-downloads/main/.github/workflows/update_version.yml | Reference implementation of `workflow_dispatch` triggered from trivy |

## ASCII Data Flow Schema

```
  +----------------------------------+
  | [1] trivy: push tag v*           |
  +---------------+------------------+
                  |
                  v
  +----------------------------------+
  | [2] release job                  |
  |   reusable-release.yaml          |
  |   → build binaries               |
  |   → publish GitHub Release       |
  |     (assets: .rpm, .deb)         |
  +---------------+------------------+
                  |
                  v
  +----------------------------------+
  | [3] deploy-packages job          |
  |   (trivy/release.yaml)           |
  |   gh workflow run →              |
  |   trivy-repo/deploy-packages.yml |
  |   --field version=$GITHUB_REF_NAME
  +---------------+------------------+
                  |
                  v
  +----------------------------------+
  | [4] deploy-packages.yml          |
  |   (trivy-repo)                   |
  |   gh release download (assets)   |
  |   → install deps                 |
  |     (rpm, reprepro,              |
  |      createrepo-c, distro-info)  |
  |   → create rpm repo              |
  |   → import GPG key               |
  |   → create deb repo              |
  |   → git push → trivy-repo/main   |
  +----------------------------------+

  Parallel (after step 2, not waiting for step 4):
  +----------------------------------+
  | update-chart-version             |
  | trigger-version-update           |
  +----------------------------------+
```

## Inputs

| Input | Required | Description |
|-------|----------|-------------|
| `version` | yes | Trivy release version, e.g. `v0.60.0`. Passed from trivy via `--field version=$GITHUB_REF_NAME`. |

## Required Secrets (in trivy-repo)

| Secret | Purpose |
|--------|---------|
| `ORG_REPO_TOKEN` | git push to trivy-repo + `gh release download` from trivy |
| `GPG_KEY` | Signing deb packages |
| `GPG_PASSPHRASE` | GPG key passphrase |

## Execution Steps

1. **Read all reference files** listed in the Reference Files table above before writing
   any code. Understand the full logic of both scripts before inlining them.

2. **Verify `gh release download` asset patterns** by running:
   ```bash
   gh release view --repo aquasecurity/trivy --json assets --jq '.assets[].name' | sort
   ```
   Check that the asset filenames match the patterns used in the scripts (e.g. `*64bit.rpm`,
   `*ARM64.rpm`, `*Linux-64bit.deb`). If the patterns don't match, propose corrected globs.

3. **Create `trivy-repo/.github/workflows/deploy-packages.yml`:**
   - Trigger: `workflow_dispatch` with required string input `version`
   - Runner: `ubuntu-24.04` (add comment that this runner will changed to ubuntu-2404-2core)
   - Steps (in order):
     1. Checkout `trivy-repo` with `persist-credentials: true` (needed for `git push`)
     2. Setup git user (`github-actions[bot]`)
     3. Install deps: `sudo apt-get install -y rpm reprepro createrepo-c distro-info`
     4. Download release assets into `dist/`: (Check - perhaps we can change `dist` to another name)
        ```bash
        gh release download "${{ inputs.version }}" \
          --repo aquasecurity/trivy \
          --pattern "*.rpm" --pattern "*.deb" \
          --dir dist/
        ```
        Use `GH_TOKEN: ${{ secrets.ORG_REPO_TOKEN }}` (add comment for ORG_REPO_TOKEN that this token will be changed (i mean name))
     5. Inline the RPM creation logic from `deploy-rpm.sh` — adapt paths so that `dist/`
        is relative to the workflow workspace root (scripts originally assumed a parent `dist/`)
     6. Inline the git commit + push from `deploy-rpm.sh`
     7. Import GPG key: `echo -e "${{ secrets.GPG_KEY }}" | gpg --import`
     8. Inline the DEB creation logic from `deploy-deb.sh` — same path adaptation
     9. Inline the git commit + push from `deploy-deb.sh`

4. **Modify `trivy/.github/workflows/release.yaml`:**
   - Replace the entire `deploy-packages` job body with a single `gh workflow run` step
     (follow the same pattern used in `trigger-version-update` for `trivy-telemetry`):
     ```bash
     gh workflow run deploy-packages.yml \
       --repo "$GITHUB_REPOSITORY_OWNER/trivy-repo" \
       --ref main \
       --field "version=$GITHUB_REF_NAME"
     ```
   - Set `GH_TOKEN: ${{ secrets.ORG_REPO_TOKEN }}` on this step
   - Keep `needs: release` on the `deploy-packages` job
   - Remove the steps that are no longer needed (checkout, cache restore, apt install, etc.)

5. **Update downstream job dependencies in `trivy/release.yaml`:**
   - Change `update-chart-version` and `trigger-version-update` from
     `needs: deploy-packages` to `needs: release`
   - **Important:** Before making this change, explicitly document the trade-off:
     these jobs will now run in parallel with rpm/deb deployment. Users could
     theoretically see a new helm chart version before `.deb`/`.rpm` packages are live.
     Report this risk and ask for confirmation before proceeding.

## Key Design Decisions

- **Inline scripts instead of script files** — the workflow is self-contained; no need to
  maintain separate `.sh` files in trivy-repo for logic this small.
- **`gh release download` instead of Actions cache** — the cache is scoped to the originating
  repo and cannot be shared cross-repo. GitHub Release assets are the canonical source for
  published binaries.
- **`needs: release` for downstream jobs** — removes the hard dependency on deploy completion.
  Acceptable if helm chart updates and telemetry triggers do not require packages to be live
  first. Must be flagged to the user explicitly.
- **`workflow_dispatch` only** — the new workflow is triggered exclusively from trivy; it
  must not run on push or schedule.

## Security Properties

- `ORG_REPO_TOKEN` is read from secrets only — never passed as a CLI argument or logged.
- `GPG_KEY` and `GPG_PASSPHRASE` are read from secrets only.
- `persist-credentials: true` is set only where `git push` is required.
- `gh release download` fetches assets over HTTPS from GitHub.
- No third-party actions used beyond `actions/checkout`.

## Troubleshooting

- **`gh workflow run` fails with "HTTP 422"** — the workflow file does not exist yet on `main`
  in trivy-repo; push the new workflow before testing the trigger.
- **`gh release download` finds no matching assets** — run
  `gh release view <version> --repo aquasecurity/trivy --json assets` to inspect actual names
  and correct the `--pattern` globs.
- **`createrepo_c` or `reprepro` not found** — verify package names on ubuntu-22.04 with
  `apt-cache search createrepo`.
- **GPG import fails silently** — ensure `GPG_KEY` contains the full armored private key
  including `-----BEGIN PGP PRIVATE KEY BLOCK-----` header/footer.
- **`update-chart-version` sees new version before packages are live** — expected after
  switching to `needs: release`; if problematic, add `gh run watch` in the
  `deploy-packages` job to block until the trivy-repo workflow finishes.
