---
title: Set up Bicep development and deployment environments
description: How to configure Bicep development and deployment environments
ms.topic: conceptual
ms.date: 09/10/2021
ms.custom: devx-track-azurepowershell, devx-track-azurecli
---

# Install Bicep tools

Let's make sure your environment is set up for developing and deploying Bicep files.

## VS Code and Bicep extension

To create Bicep files, you need a good Bicep editor. We recommend:

- **Visual Studio Code** - If you don't already have Visual Studio Code, [install it](https://code.visualstudio.com/).
- **Bicep extension for Visual Studio Code**.  Visual Studio Code with the Bicep extension provides language support and resource autocompletion. The extension helps you create and validate Bicep files.

  To install the extension, search for *bicep* in the **Extensions** tab or in the [Visual Studio marketplace](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-bicep).

  Select **Install**.

  :::image type="content" source="./media/install/install-extension.png" alt-text="Install Bicep extension":::

To verify you've installed the extension, open any file with the `.bicep` file extension. You should see the language mode in the lower right corner change to **Bicep**.

:::image type="content" source="./media/install/language-mode.png" alt-text="Bicep language mode":::

## Deployment environment

After setting up your development environment, you need to install the tools for your deployment environment. To set up a local deployment environment, install the Bicep CLI with [Azure CLI](#azure-cli) or [Azure PowerShell](#azure-powershell). Those steps are shown in the next sections.

To deploy Bicep files from an Azure Pipeline, see [Integrate Bicep with Azure Pipelines](add-template-to-azure-pipelines.md). To deploy Bicep files through GitHub Actions, see [Deploy Bicep files by using GitHub Actions](deploy-github-actions.md).

## Azure CLI

You must have Azure CLI version 2.20.0 or later installed. To install or update Azure CLI, see:

- [Install Azure CLI on Windows](/cli/azure/install-azure-cli-windows)
- [Install Azure CLI on Linux](/cli/azure/install-azure-cli-linux)
- [Install Azure CLI on macOS](/cli/azure/install-azure-cli-macos)

To verify your current version, run:

```azurecli
az --version
```

You now have everything you need to [deploy](deploy-cli.md) and [decompile](decompile.md) Bicep files. You have everything because Azure CLI 2.20.0 or later automatically installs the Bicep CLI when a command is executed that needs it.

To manually start the Bicep CLI installation, use:

```azurecli
az bicep install
```

To upgrade to the latest version, use:

```azurecli
az bicep upgrade
```

To validate the install, use:

```azurecli
az bicep version
```

For more commands, see [Bicep CLI](bicep-cli.md).

> [!IMPORTANT]
> Azure CLI installs a self-contained instance of the Bicep CLI. This instance doesn't conflict with any versions you may have manually installed. Azure CLI doesn't add Bicep CLI to your PATH.

## Azure PowerShell

You must have Azure PowerShell version 5.6.0 or later installed. To update or install, see [Install Azure PowerShell](/powershell/azure/install-az-ps).

Azure PowerShell doesn't automatically install the Bicep CLI. Instead, you must [manually install the Bicep CLI](#install-manually).

> [!IMPORTANT]
> The self-contained instance of the Bicep CLI installed by Azure CLI isn't available to PowerShell commands. Azure PowerShell deployments fail if you haven't manually installed the Bicep CLI.

When you manually install the Bicep CLI, run the Bicep commands with the `bicep` syntax, instead of the `az bicep` syntax for Azure CLI.

To deploy Bicep files, use Bicep CLI version 0.3.1 or later. To check your Bicep CLI version, run:

```cmd
bicep --version
```

## Install manually

The following methods install the Bicep CLI and add it to your PATH. You must manually install for any use other than Azure CLI.

- [Linux](#linux)
- [macOS](#macos)
- [Windows](#windows)

### Linux

```sh
# Fetch the latest Bicep CLI binary
curl -Lo bicep https://github.com/Azure/bicep/releases/latest/download/bicep-linux-x64
# Mark it as executable
chmod +x ./bicep
# Add bicep to your PATH (requires admin)
sudo mv ./bicep /usr/local/bin/bicep
# Verify you can now access the 'bicep' command
bicep --help
# Done!
```

> [!NOTE]
> For lightweight Linux distributions like [Alpine](https://alpinelinux.org/), use **bicep-linux-musl-x64** instead of **bicep-linux-x64** in the preceding script.

### macOS

#### via homebrew

```sh
# Add the tap for bicep
brew tap azure/bicep

# Install the tool
brew install bicep
```

#### via BASH

```sh
# Fetch the latest Bicep CLI binary
curl -Lo bicep https://github.com/Azure/bicep/releases/latest/download/bicep-osx-x64
# Mark it as executable
chmod +x ./bicep
# Add Gatekeeper exception (requires admin)
sudo spctl --add ./bicep
# Add bicep to your PATH (requires admin)
sudo mv ./bicep /usr/local/bin/bicep
# Verify you can now access the 'bicep' command
bicep --help
# Done!

```

### Windows

#### Windows Installer

Download and run the [latest Windows installer](https://github.com/Azure/bicep/releases/latest/download/bicep-setup-win-x64.exe). The installer doesn't require administrative privileges. After the installation, Bicep CLI is added to your user PATH. Close and reopen any open command shell windows for the PATH change to take effect.

#### Chocolatey

```powershell
choco install bicep
```

#### Winget

```powershell
winget install -e --id Microsoft.Bicep
```

#### Manual with PowerShell

```powershell
# Create the install folder
$installPath = "$env:USERPROFILE\.bicep"
$installDir = New-Item -ItemType Directory -Path $installPath -Force
$installDir.Attributes += 'Hidden'
# Fetch the latest Bicep CLI binary
(New-Object Net.WebClient).DownloadFile("https://github.com/Azure/bicep/releases/latest/download/bicep-win-x64.exe", "$installPath\bicep.exe")
# Add bicep to your PATH
$currentPath = (Get-Item -path "HKCU:\Environment" ).GetValue('Path', '', 'DoNotExpandEnvironmentNames')
if (-not $currentPath.Contains("%USERPROFILE%\.bicep")) { setx PATH ($currentPath + ";%USERPROFILE%\.bicep") }
if (-not $env:path.Contains($installPath)) { $env:path += ";$installPath" }
# Verify you can now access the 'bicep' command.
bicep --help
# Done!
```

## Install on air-gapped cloud

To install Bicep CLI in an air-gapped environment, you need to download the Bicep CLI executable manually and save it to a certain location.

- **Linux**

    1. Download **bicep-linux-x64** from the [Bicep release page](https://github.com/Azure/bicep/releases/latest/) in a non-air-gapped environment.
    1. Copy the executable to the **$HOME/.azure/bin** directory on an air-gapped machine.

- **macOS**

    1. Download **bicep-osx-x64** from the [Bicep release page](https://github.com/Azure/bicep/releases/latest/) in a non-air-gapped environment.
    1. Copy the executable to the **$HOME/.azure/bin** directory on an air-gapped machine.

- **Windows**

    1. Download **bicep-win-x64.exe** from the [Bicep release page](https://github.com/Azure/bicep/releases/latest/) in a non-air-gapped environment.
    1. Copy the executable to the **%UserProfile%/.azure/bin** directory on an air-gapped machine.

Note `bicep install` and `bicep upgrade` commands don't work in an air-gapped environment.

## Install the nightly builds

If you'd like to try the latest pre-release bits of Bicep before they're released, see [Install nightly builds](https://github.com/Azure/bicep/blob/main/docs/installing-nightly.md).

> [!WARNING]
> These pre-release builds are much more likely to have known or unknown bugs.

## Next steps

For more information about using Visual Studio Code and the Bicep extension, see [Quickstart: Create Bicep files with Visual Studio Code](./quickstart-create-bicep-use-visual-studio-code.md).
