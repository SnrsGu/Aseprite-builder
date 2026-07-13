# Aseprite Builder

[中文说明](README.zh-CN.md)

This repository builds portable Aseprite packages with GitHub Actions. A manual run can build any combination of the supported targets, while scheduled and `main` branch builds keep all targets enabled.

## Supported packages

| Workflow option | Output | Validation status |
| --- | --- | --- |
| macOS Apple Silicon | `Aseprite-<version>-macOS-Apple-Silicon.zip` | ✅ Verified working |
| macOS Intel | `Aseprite-<version>-macOS-Intel.zip` | ✅ Verified working |
| Windows x64 | `Aseprite-<version>-Windows-x64.zip` | ✅ Verified working |
| Windows ARM64 | `Aseprite-<version>-Windows-arm64.zip` | 🟡 Pending validation |
| Linux x64 Debian | `Aseprite-<version>-Linux-deb-amd64.tar.gz` | 🟡 Pending validation |
| Linux x64 AppImage | `Aseprite-<version>-Linux-AppImage-x86_64.tar.gz` | 🟡 Pending validation |
| Linux ARM64 Debian | `Aseprite-<version>-Linux-deb-arm64.tar.gz` | 🟡 Pending validation |
| Linux ARM64 AppImage | `Aseprite-<version>-Linux-AppImage-aarch64.tar.gz` | 🟡 Pending validation |

Each Linux architecture is compiled only once when both Debian and AppImage are selected.

## Build packages in your own fork

1. Sign in to GitHub and click **Fork** on this repository.
2. Open the fork and select the **Actions** tab.
3. If GitHub displays a warning for forked workflows, click **I understand my workflows, go ahead and enable them**.
4. Open **Settings → Actions → General**.
5. Under **Workflow permissions**, select **Read and write permissions**, then save. This permission is required to create a Release and upload its files.
6. Return to **Actions** and select **Build and deploy Aseprite**.
7. Click **Run workflow**.
8. Select the branch containing the workflow, normally `main`.
9. Enable only the packages you need. At least one option must remain selected.
10. Click the green **Run workflow** button.

No custom secret or personal access token is required. The workflow uses the repository-provided `GITHUB_TOKEN`.

## Choose which platforms to build

Open **Actions → Build and deploy Aseprite → Run workflow**. The form contains eight independent checkboxes:

| Checkbox | Package that will be built |
| --- | --- |
| Build macOS Apple Silicon ZIP | macOS for Apple Silicon |
| Build macOS Intel ZIP | macOS for Intel processors |
| Build Windows x64 ZIP | Windows 64-bit for Intel/AMD processors |
| Build Windows ARM64 ZIP | Windows for ARM64 processors |
| Build Linux x64 Debian package | Debian package for Linux x64 |
| Build Linux x64 AppImage | AppImage for Linux x64 |
| Build Linux ARM64 Debian package | Debian package for Linux ARM64 |
| Build Linux ARM64 AppImage | AppImage for Linux ARM64 |

Uncheck every package you do not need, leave at least one checked, and then click **Run workflow**. For example:

- To build only normal 64-bit Windows, leave only **Build Windows x64 ZIP** checked.
- To build both Linux x64 formats, leave **Build Linux x64 Debian package** and **Build Linux x64 AppImage** checked. Aseprite is compiled once and both packages are produced from that build.
- To build macOS for both processor families, enable both macOS checkboxes.

All checkboxes default to enabled. To change the defaults shown in your fork, edit the corresponding `default` values under `on.workflow_dispatch.inputs` in [`.github/workflows/aseprite_build_deploy.yml`](.github/workflows/aseprite_build_deploy.yml):

```yaml
workflow_dispatch:
  inputs:
    windows_x64:
      description: Build Windows x64 ZIP
      type: boolean
      default: true
    windows_arm64:
      description: Build Windows ARM64 ZIP
      type: boolean
      default: false
```

The checkboxes control manual runs only. Scheduled runs and pushes to `main` intentionally build all supported targets.

## Where to find the results

Every successfully packaged file is uploaded immediately to the Release matching the Aseprite version. A failed platform does not prevent successful platforms from being published.

The same files are also stored as workflow artifacts:

1. Open the completed workflow run.
2. Scroll to **Artifacts**.
3. Download the artifact named `packages-<platform>`.

Running the workflow again for the same Aseprite version replaces files with the same name in the existing Release.

## Automatic builds

The workflow also runs:

- once per day on its configured schedule;
- after a push to `main`.

Automatic runs enable every supported package. Version caching prevents scheduled runs from rebuilding the same stable Aseprite release. Prerelease tags containing `beta` or `rc` are ignored by automatic runs. A manual run always starts a build with the selected targets.

To build only selected platforms automatically, edit the trigger or the matrix configuration in [`.github/workflows/aseprite_build_deploy.yml`](.github/workflows/aseprite_build_deploy.yml). The checkboxes apply to manual `workflow_dispatch` runs.

## Build implementation

- macOS Apple Silicon uses a native Apple Silicon runner.
- macOS Intel uses a native Intel runner.
- Windows x64 uses the official prebuilt Aseprite Skia package.
- Windows ARM64 builds the `aseprite-m124` Skia branch from source because the official Skia release does not include a Windows ARM64 archive.
- Linux x64 uses the official prebuilt Skia package.
- Linux ARM64 builds Skia from source because the official release does not include a Linux ARM64 archive.
- Skia is cached per operating system and architecture. The first ARM64 build can take considerably longer than later runs.
- Windows packages use static curl with Windows Schannel and are checked for accidental OpenSSL DLL dependencies.
- Output binaries are checked to ensure that their PE, Mach-O, or ELF architecture matches the requested target.

## Customization

The most commonly changed values are at the top of the workflow:

```yaml
env:
  BUILD_TYPE: Release
  SKIA_VERSION: m124-08a5439a6b
  ASEPRITE_RELEASE_API_URL: https://api.github.com/repos/aseprite/aseprite/releases/latest
```

The Aseprite release queried by the workflow is configured with `ASEPRITE_RELEASE_API_URL`. If you change the Aseprite source version, verify that its required Skia version matches `SKIA_VERSION`.

Linux icons and desktop metadata are stored in [`Linux/`](Linux/). macOS bundle resources are stored in [`macOS/`](macOS/).

## Troubleshooting

- **No workflow appears in Actions:** ensure the workflow file exists on the fork's default branch and Actions are enabled.
- **Release upload returns 403:** enable **Read and write permissions** under the repository's Actions settings.
- **No Release is created:** check the `check-version` job first. It must be able to use `GITHUB_TOKEN` with `contents: write` permission.
- **An ARM64 build is slow:** the first run compiles Skia from source. Later successful runs restore it from the Actions cache.
- **One platform failed:** successful packages are still uploaded. Open only the failed matrix job to inspect its log, then rerun the workflow if needed.

## License and redistribution

This project automates compilation and packaging; it does not grant additional rights to Aseprite, Skia, translations, or other dependencies. Before publishing or redistributing artifacts, review the licenses and terms in the upstream Aseprite source release and every included dependency.
