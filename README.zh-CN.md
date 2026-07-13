# Aseprite Builder

[English](README.md)

本项目通过 GitHub Actions 编译并打包便携版 Aseprite。手动运行时可以任意选择需要的平台和格式；定时构建以及 `main` 分支推送构建默认启用全部目标。

## 支持的产物

| 工作流选项 | 输出文件 | 验证状态 |
| --- | --- | --- |
| macOS Apple Silicon | `Aseprite-<版本>-macOS-Apple-Silicon.zip` | ✅ 已验证 |
| macOS Intel | `Aseprite-<版本>-macOS-Intel.zip` | ✅ 已验证 |
| Windows x64 | `Aseprite-<版本>-Windows-x64.zip` | ✅ 已验证 |
| Windows ARM64 | `Aseprite-<版本>-Windows-arm64.zip` | 🟡 待验证 |
| Linux x64 Debian | `Aseprite-<版本>-Linux-deb-amd64.tar.gz` | 🟡 待验证 |
| Linux x64 AppImage | `Aseprite-<版本>-Linux-AppImage-x86_64.tar.gz` | 🟡 待验证 |
| Linux ARM64 Debian | `Aseprite-<版本>-Linux-deb-arm64.tar.gz` | 🟡 待验证 |
| Linux ARM64 AppImage | `Aseprite-<版本>-Linux-AppImage-aarch64.tar.gz` | 🟡 待验证 |

同一 Linux 架构同时选择 Debian 和 AppImage 时只会编译一次，然后分别生成两种格式。

## 在自己的 Fork 中构建

1. 登录 GitHub，在本项目页面点击 **Fork**。
2. 进入自己的 Fork，打开 **Actions** 标签页。
3. 如果 GitHub 显示 Fork 工作流警告，点击 **I understand my workflows, go ahead and enable them**。
4. 打开 **Settings → Actions → General**。
5. 在 **Workflow permissions** 中选择 **Read and write permissions** 并保存。创建 Release 和上传文件需要该权限。
6. 返回 **Actions**，选择 **Build and deploy Aseprite**。
7. 点击 **Run workflow**。
8. 选择包含工作流的分支，通常为 `main`。
9. 只勾选自己需要的产物，至少需要保留一个选项。
10. 点击绿色的 **Run workflow** 按钮开始构建。

不需要创建自定义 Secret 或个人访问令牌。工作流使用仓库自动提供的 `GITHUB_TOKEN`。

## 选择需要打包的平台

打开 **Actions → Build and deploy Aseprite → Run workflow**，运行表单中会显示 8 个相互独立的复选框：

| 复选框 | 对应产物 |
| --- | --- |
| Build macOS Apple Silicon ZIP | Apple Silicon 芯片版 macOS |
| Build macOS Intel ZIP | Intel 芯片版 macOS |
| Build Windows x64 ZIP | Intel/AMD 处理器使用的 Windows 64 位版 |
| Build Windows ARM64 ZIP | ARM64 处理器使用的 Windows 版 |
| Build Linux x64 Debian package | Linux x64 Debian 安装包 |
| Build Linux x64 AppImage | Linux x64 AppImage |
| Build Linux ARM64 Debian package | Linux ARM64 Debian 安装包 |
| Build Linux ARM64 AppImage | Linux ARM64 AppImage |

取消勾选不需要的产物，至少保留一个选项，然后点击 **Run workflow**。例如：

- 只构建常规 Windows 64 位版：只保留 **Build Windows x64 ZIP**。
- 同时构建 Linux x64 的两种格式：保留 **Build Linux x64 Debian package** 和 **Build Linux x64 AppImage**。Aseprite 只编译一次，再从同一结果生成两种安装包。
- 同时构建两种 macOS 架构：勾选两个 macOS 选项。

所有复选框默认启用。如果需要修改自己 Fork 中显示的默认选择，请编辑 [`.github/workflows/aseprite_build_deploy.yml`](.github/workflows/aseprite_build_deploy.yml) 中 `on.workflow_dispatch.inputs` 对应项目的 `default`：

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

这些复选框只控制手动运行。定时任务和向 `main` 分支推送时仍会构建全部支持的目标。

## 获取构建结果

每个文件打包成功后会立即上传到对应 Aseprite 版本的 Release。某个平台失败不会阻止其他成功平台发布。

相同文件也会保存为工作流 Artifact：

1. 打开已经完成的工作流运行记录。
2. 滚动到 **Artifacts** 区域。
3. 下载名为 `packages-<平台>` 的 Artifact。

对同一个 Aseprite 版本重复运行时，Release 中的同名文件会被新产物替换。

## 自动构建

工作流还会在以下情况运行：

- 按现有定时配置每天运行一次；
- 向 `main` 分支推送后运行。

自动运行会启用全部支持的产物。版本缓存会避免定时任务重复构建同一个稳定版 Aseprite；包含 `beta` 或 `rc` 的预发布版本不会被自动构建。手动运行始终会按照所选目标开始构建。

如果需要让自动构建也只编译指定平台，请编辑 [`.github/workflows/aseprite_build_deploy.yml`](.github/workflows/aseprite_build_deploy.yml) 中的触发器或矩阵配置。页面复选框仅用于手动 `workflow_dispatch` 运行。

## 构建实现

- macOS Apple Silicon 使用原生 Apple Silicon runner。
- macOS Intel 使用原生 Intel runner。
- Windows x64 使用 Aseprite 官方预编译 Skia。
- Windows ARM64 因官方 Skia Release 没有对应压缩包，会从源码编译 `aseprite-m124` 分支。
- Linux x64 使用官方预编译 Skia。
- Linux ARM64 因官方 Release 没有对应压缩包，会从源码编译 Skia。
- Skia 按操作系统和架构缓存，ARM64 第一次构建会明显慢于后续构建。
- Windows 使用静态 curl 和系统 Schannel，并检查是否意外依赖 OpenSSL DLL。
- 打包前会验证 PE、Mach-O 或 ELF 的实际架构是否与目标一致。

## 自定义配置

工作流顶部包含最常修改的参数：

```yaml
env:
  BUILD_TYPE: Release
  SKIA_VERSION: m124-08a5439a6b
  ASEPRITE_RELEASE_API_URL: https://api.github.com/repos/aseprite/aseprite/releases/latest
```

Aseprite 版本来源由 `ASEPRITE_RELEASE_API_URL` 配置。如果修改 Aseprite 源码版本，需要确认该版本要求的 Skia 与 `SKIA_VERSION` 一致。

Linux 图标和桌面文件位于 [`Linux/`](Linux/)，macOS Bundle 资源位于 [`macOS/`](macOS/)。

## 常见问题

- **Actions 中没有工作流：**确认工作流文件存在于 Fork 的默认分支，并且已经启用 Actions。
- **上传 Release 返回 403：**在仓库 Actions 设置中启用 **Read and write permissions**。
- **没有创建 Release：**先检查 `check-version` job，确认 `GITHUB_TOKEN` 拥有 `contents: write` 权限。
- **ARM64 构建很慢：**第一次需要从源码编译 Skia，后续成功运行会使用 Actions 缓存。
- **某个平台失败：**其他成功产物仍会正常上传。只需打开失败的矩阵 job 查看日志，修复后可以重新运行工作流。

## 许可与再分发

本项目只负责自动编译和打包，不会额外授予 Aseprite、Skia、翻译文件或其他依赖的使用与分发权利。发布或再分发产物前，请检查 Aseprite 源码包以及所有依赖中包含的许可证和条款。
