# Windows 上 Codex Desktop 的 Chrome 和 Computer Use 插件不可用：一次完整排查与修复

> 适用场景：Windows 版 Codex Desktop 更新后，设置页中 Chrome 插件或 Computer Use 插件显示不可用，或者点击插件设置时弹出 Electron / app path 相关错误。

## 背景

这次问题表面上看是 Chrome 插件或 Computer Use 插件坏了，但实际根因在 Codex Desktop 本地的 bundled plugin marketplace 状态损坏。

我遇到的现象主要有这些：

- Codex 设置页里 `Chrome` 插件显示不可用。
- `Computer Use` 插件显示不可用。
- 点击 Chrome 插件的设置入口时，可能弹出类似 `Error launching app` 的 Electron 错误。
- Chrome 浏览器扩展本身可能已经安装，并且扩展弹窗里显示 `Connected`，但 Codex 里仍然识别失败。
- 日志里反复出现：
  - `bundled_plugins_marketplace_resolve_failed`
  - `Windows Computer Use helper paths are unavailable`
  - `computer-use native pipe startup failed`

所以排查重点不要只盯着 Chrome 扩展本身，而要同时检查 Codex 的本地 marketplace、插件缓存、`latest` 指针和 `codex://` 协议注册表。

## 备份

动手前建议先备份这些文件和注册表项：

```powershell
$stamp = Get-Date -Format 'yyyyMMdd-HHmmss'
$backup = "$env:USERPROFILE\.codex\backups\computer-use-fix-$stamp"
New-Item -ItemType Directory -Force -Path $backup | Out-Null

$files = @(
  "$env:USERPROFILE\.codex\config.toml",
  "$env:USERPROFILE\.codex\.codex-global-state.json",
  "$env:USERPROFILE\.codex\chrome-native-hosts.json",
  "$env:LOCALAPPDATA\OpenAI\extension\com.openai.codexextension.json"
)

foreach ($f in $files) {
  if (Test-Path -LiteralPath $f) {
    Copy-Item -LiteralPath $f -Destination $backup -Force
  }
}

reg export HKCU\Software\Google\Chrome\NativeMessagingHosts\com.openai.codexextension `
  (Join-Path $backup 'chrome-native-host.reg') /y

reg export HKCU\Software\Classes\codex `
  (Join-Path $backup 'codex-url-protocol.reg') /y
```

## 先确认问题

查看 Codex 插件列表：

```powershell
codex plugin marketplace list
codex plugin list
```

如果出现类似下面的错误，说明本地 bundled marketplace 没有被正确识别：

```text
failed to load configured marketplace snapshot(s)
marketplace root does not contain a supported manifest
```

再检查 `config.toml` 里 `openai-bundled` 的配置：

```toml
[marketplaces.openai-bundled]
source_type = "local"
source = '\\?\C:\Users\xxx\.codex\.tmp\bundled-marketplaces\openai-bundled'
```

如果它指向 `.codex\.tmp\bundled-marketplaces\openai-bundled`，而这个目录又是不完整的，就很容易导致 Chrome 和 Computer Use 同时不可用。

正常情况下，更稳定的 source 应该指向：

```text
%USERPROFILE%\.codex\plugins\cache\openai-bundled\marketplace-source
```

并且该目录必须包含：

```text
.agents\plugins\marketplace.json
plugins\browser
plugins\chrome
plugins\computer-use
plugins\latex
```

## 找到当前 Codex 安装包

在 Windows 上可以用这个命令找到 Codex Desktop 当前安装位置：

```powershell
Get-AppxPackage -Name OpenAI.Codex | Select-Object Name,Version,InstallLocation,PackageFullName
```

安装包里的完整 bundled plugin 通常位于：

```text
C:\Program Files\WindowsApps\OpenAI.Codex_版本号_x64__2p2nqsd0c76g0\app\resources\plugins\openai-bundled
```

## 修复 marketplace 和插件缓存

下面脚本会做几件事：

- 从 Codex 安装包复制完整的 `openai-bundled` marketplace 到稳定缓存目录。
- 同步补齐旧的 `.tmp\bundled-marketplaces\openai-bundled`，避免 Codex 进程仍读旧路径时报错。
- 给 `browser`、`chrome`、`computer-use` 创建 `latest` junction。
- 将 `config.toml` 中的 `openai-bundled` source 改到稳定目录，并用 UTF-8 without BOM 保存。

执行前把 `$installOpenAiBundled` 改成你机器上实际的 Codex 安装路径。

```powershell
$installOpenAiBundled = 'C:\Program Files\WindowsApps\OpenAI.Codex_26.527.7698.0_x64__2p2nqsd0c76g0\app\resources\plugins\openai-bundled'

$market = "$env:USERPROFILE\.codex\plugins\cache\openai-bundled\marketplace-source"
$tmpMarket = "$env:USERPROFILE\.codex\.tmp\bundled-marketplaces\openai-bundled"

New-Item -ItemType Directory -Force -Path $market | Out-Null
Get-ChildItem -Force -Path $installOpenAiBundled | ForEach-Object {
  Copy-Item -LiteralPath $_.FullName -Destination $market -Recurse -Force
}

New-Item -ItemType Directory -Force -Path $tmpMarket | Out-Null
Get-ChildItem -Force -Path $installOpenAiBundled | ForEach-Object {
  Copy-Item -LiteralPath $_.FullName -Destination $tmpMarket -Recurse -Force
}

$pluginRoot = "$env:USERPROFILE\.codex\plugins\cache\openai-bundled"
$links = @{
  'browser' = '26.527.60818'
  'chrome' = '26.527.60818'
  'computer-use' = '26.527.60818'
}

foreach ($name in $links.Keys) {
  $link = Join-Path (Join-Path $pluginRoot $name) 'latest'
  $target = Join-Path (Join-Path $pluginRoot $name) $links[$name]

  if (Test-Path -LiteralPath $link) {
    $item = Get-Item -LiteralPath $link -Force
    if ($item.LinkType -eq 'Junction' -and $item.Target -contains $target) {
      continue
    }
    Rename-Item -LiteralPath $link -NewName ("latest.backup-" + (Get-Date -Format 'yyyyMMddHHmmss')) -Force
  }

  New-Item -ItemType Junction -Path $link -Target $target | Out-Null
}

$configPath = "$env:USERPROFILE\.codex\config.toml"
$config = [System.IO.File]::ReadAllText($configPath)
$stableSource = "\\?\$env:USERPROFILE\.codex\plugins\cache\openai-bundled\marketplace-source"
$replacement = '${1}' + "'$stableSource'"
$config = [regex]::Replace(
  $config,
  '(?ms)(\[marketplaces\.openai-bundled\].*?\r?\nsource\s*=\s*)''[^'']*''',
  $replacement
)

$utf8NoBom = New-Object System.Text.UTF8Encoding($false)
[System.IO.File]::WriteAllText($configPath, $config, $utf8NoBom)
```

注意：如果复制 `.tmp` 目录时报 `extension-host.exe` 正在被占用，可以先不强行删除旧目录。只要 stable marketplace 已经完整，并且 `config.toml` 指向 stable marketplace，通常就能解决主要问题。

## 修复 codex:// 协议

如果点击通知、插件设置或深链时仍然弹 Electron app path 错误，继续检查：

```powershell
reg query HKCU\Software\Classes\codex /s
reg query HKCU\Software\Classes\AppXybfp6cjpb1wf0pftw0fd4bz59gzn1401 /s
```

残缺状态通常只有：

```text
HKEY_CURRENT_USER\Software\Classes\codex
    URL Protocol    REG_SZ
    (Default)       REG_SZ    URL:codex
```

正常情况应该带有 `Shell\open\command\DelegateExecute`。可以按 AppX handler 补齐：

```powershell
$appxKey = 'HKCU:\Software\Classes\AppXybfp6cjpb1wf0pftw0fd4bz59gzn1401'
$codexKey = 'HKCU:\Software\Classes\codex'

New-Item -Path $codexKey -Force | Out-Null
New-ItemProperty -Path $codexKey -Name '(default)' -Value 'URL:codex' -PropertyType String -Force | Out-Null
New-ItemProperty -Path $codexKey -Name 'URL Protocol' -Value '' -PropertyType String -Force | Out-Null

foreach ($sub in @('Application','DefaultIcon','Shell','Shell\open','Shell\open\command')) {
  New-Item -Path (Join-Path $codexKey $sub) -Force | Out-Null
}

$app = Get-ItemProperty -Path (Join-Path $appxKey 'Application')
New-ItemProperty -Path (Join-Path $codexKey 'Application') -Name 'ApplicationName' -Value $app.ApplicationName -PropertyType String -Force | Out-Null
New-ItemProperty -Path (Join-Path $codexKey 'Application') -Name 'ApplicationCompany' -Value $app.ApplicationCompany -PropertyType String -Force | Out-Null
New-ItemProperty -Path (Join-Path $codexKey 'Application') -Name 'ApplicationIcon' -Value $app.ApplicationIcon -PropertyType String -Force | Out-Null
New-ItemProperty -Path (Join-Path $codexKey 'Application') -Name 'ApplicationDescription' -Value $app.ApplicationDescription -PropertyType String -Force | Out-Null
New-ItemProperty -Path (Join-Path $codexKey 'Application') -Name 'AppUserModelID' -Value $app.AppUserModelID -PropertyType String -Force | Out-Null

$open = Get-ItemProperty -Path (Join-Path $appxKey 'Shell\open')
$openPath = Join-Path $codexKey 'Shell\open'
New-ItemProperty -Path $openPath -Name 'AppUserModelID' -Value $open.AppUserModelID -PropertyType String -Force | Out-Null
New-ItemProperty -Path $openPath -Name 'PackageRelativeExecutable' -Value $open.PackageRelativeExecutable -PropertyType String -Force | Out-Null
New-ItemProperty -Path $openPath -Name 'DesktopAppXActivateOptions' -Value $open.DesktopAppXActivateOptions -PropertyType DWord -Force | Out-Null
New-ItemProperty -Path $openPath -Name 'ContractId' -Value $open.ContractId -PropertyType String -Force | Out-Null
New-ItemProperty -Path $openPath -Name 'DesiredInitialViewState' -Value $open.DesiredInitialViewState -PropertyType DWord -Force | Out-Null
New-ItemProperty -Path $openPath -Name 'PackageId' -Value $open.PackageId -PropertyType String -Force | Out-Null

$delegate = (Get-ItemProperty -Path (Join-Path $appxKey 'Shell\open\command')).DelegateExecute
New-ItemProperty -Path (Join-Path $codexKey 'Shell\open\command') -Name 'DelegateExecute' -Value $delegate -PropertyType String -Force | Out-Null
```

如果 `DefaultIcon` 默认值没有写进去，可以单独补：

```powershell
reg add HKCU\Software\Classes\codex\DefaultIcon /ve /t REG_SZ /d "@{OpenAI.Codex_26.527.7698.0_x64__2p2nqsd0c76g0?ms-resource://OpenAI.Codex/Files/assets/Square44x44Logo.png}" /f
```

## 验证

重新运行：

```powershell
codex plugin marketplace list
codex plugin list
```

修复成功后应能看到：

```text
Marketplace `openai-bundled`
C:\Users\xxx\.codex\plugins\cache\openai-bundled\marketplace-source\.agents\plugins\marketplace.json

PLUGIN                       STATUS              VERSION
browser@openai-bundled       installed, enabled  26.527.60818
chrome@openai-bundled        installed, enabled  26.527.60818
computer-use@openai-bundled  installed, enabled  26.527.60818
```

还可以检查关键文件：

```powershell
Test-Path "$env:USERPROFILE\.codex\plugins\cache\openai-bundled\marketplace-source\.agents\plugins\marketplace.json"
Test-Path "$env:USERPROFILE\.codex\plugins\cache\openai-bundled\computer-use\latest\scripts\computer-use-client.mjs"
Test-Path "$env:USERPROFILE\.codex\plugins\cache\openai-bundled\computer-use\latest\node_modules\@oai\sky\bin\windows\codex-computer-use.exe"
Test-Path "$env:USERPROFILE\.codex\plugins\cache\openai-bundled\chrome\latest\extension-host\windows\x64\extension-host.exe"
```

最后完全退出并重新打开 Codex Desktop。重启后再看设置页，Chrome 和 Computer Use 应该会恢复为可用状态。

## 总结

这类问题看起来像 Chrome 扩展或 Computer Use 插件失效，但真正的根通常是 Codex 更新后本地 bundled plugin marketplace 损坏：

- `config.toml` 指向了残缺的 `.tmp\bundled-marketplaces\openai-bundled`。
- `marketplace-source` 缺少 `.agents\plugins\marketplace.json`。
- `browser`、`chrome`、`computer-use` 的 `latest` 指针缺失或指向不完整目录。
- `codex://` 协议注册表残缺，导致点击设置或通知时走到了错误启动路径。

按上面的顺序修复后，`codex plugin list` 能正常显示 `browser`、`chrome`、`computer-use` 都是 `installed, enabled`，桌面端重启后插件状态也会恢复。
