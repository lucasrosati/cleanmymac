---
name: cleanmypc
description: 'Use esta skill sempre que o usuário invocar `/cleanmypc`, ou pedir pra "limpar o pc", "limpar windows", "limpar disco", "liberar espaço", "fazer audit do disco", "analisar storage", "limpar caches", "investigar o que ocupa espaço", ou variações como "meu C: tá cheio", "tô sem espaço no windows", "que diabos tá ocupando X GB". Também acione quando o usuário compartilhar print de Storage Sense / Disk Management / Settings → System → Storage, ou pedir pra investigar uma categoria específica (Apps & features, Temporary files, System reserved). Playbook caso-a-caso negociado — diagnóstico read-only primeiro via PowerShell, identifica vilões reais do setup específico do usuário (Windows Update cache, hiberfil.sys, WSL VHDX, Docker, browser caches, Visual Studio), propõe deleções caso-a-caso, manda pra Recycle Bin quando há risco, barra de progresso real-time pra deleções grandes. Respeita as regras absolutas da memória do usuário.'
---

# /cleanmypc — Audit e limpeza negociada do Windows

Versão Windows do `/cleanmymac`. Skill pra fazer auditoria de disco caso-a-caso no Windows do usuário, identificando os vilões REAIS do setup específico dele (não lista genérica) e propondo deleções com confirmação a cada passo. Inspirado em CCleaner + WizTree, mas com inteligência situacional via PowerShell.

## Regras absolutas (LEIA PRIMEIRO)

1. **Respeitar regras absolutas do usuário** — Se o usuário tem regras tipo "nunca apague pasta X" salvas em memória, honrar mesmo se aparecerem em rankings de tamanho. Ex: dev tools (CUDA, .NET SDK), games (Steam library), WSL distros que ele queira preservar.
2. **Confirmação obrigatória antes de toda deleção** — apresentar tabela com tamanhos e justificativa, esperar OK do usuário.
3. **Recycle Bin primeiro pra deleções grandes ou arriscadas** — usar shell COM ou mover pra pasta temp em vez de `Remove-Item -Force` direto. Rede de segurança.
4. **App fechado antes de apagar AppData do app** — usar `Get-Process` pra confirmar. Pedir pro usuário fechar via X (nunca matar processo dele sem permissão).
5. **Dados de usuário NUNCA são tocados** — listados na seção abaixo.
6. **Operações em `C:\Windows\` exigem admin elevado** — sempre avisar e pedir confirmação extra antes de qualquer DISM, cleanmgr /sageset, ou modificação de hiberfil/pagefile.
7. **OneDrive / iCloud sync**: instruir Settings → desativar sync → escolher manter no cloud. Não tocar via PowerShell.

## Playbook (5 fases)

### Fase 1 — Diagnóstico read-only

Rodar bateria de checks em paralelo. Tudo só leitura, nenhum efeito colateral. Usar PowerShell 5.1+ ou PowerShell 7+ (Core).

```powershell
# Espaço em disco — todas as drives
Get-PSDrive -PSProvider FileSystem | Format-Table Name, @{N='Used(GB)';E={[math]::Round($_.Used/1GB,1)}}, @{N='Free(GB)';E={[math]::Round($_.Free/1GB,1)}}, @{N='Total(GB)';E={[math]::Round(($_.Used+$_.Free)/1GB,1)}}

# Hiberfil + Pagefile (geralmente invisíveis em rankings, mas ocupam giga)
Get-Item C:\hiberfil.sys, C:\pagefile.sys, C:\swapfile.sys -Force -ErrorAction SilentlyContinue | Select Name, @{N='GB';E={[math]::Round($_.Length/1GB,2)}}

# Windows.old (sobras de updates do Windows)
if (Test-Path C:\Windows.old) { Get-ChildItem C:\Windows.old -Recurse -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum | % { "Windows.old: $([math]::Round($_.Sum/1GB,2)) GB" } }

# Volume Shadow Copies (System Restore points, equivalente a Time Machine snapshots)
vssadmin list shadowstorage 2>$null

# Top 20 maiores pastas em %LOCALAPPDATA%
Get-ChildItem $env:LOCALAPPDATA -Directory | ForEach-Object {
  $size = (Get-ChildItem $_.FullName -Recurse -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum).Sum
  [PSCustomObject]@{ Name = $_.Name; GB = [math]::Round($size/1GB,2) }
} | Sort-Object GB -Descending | Select -First 20 | Format-Table

# Top 20 maiores pastas em %APPDATA% (Roaming)
Get-ChildItem $env:APPDATA -Directory | ForEach-Object {
  $size = (Get-ChildItem $_.FullName -Recurse -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum).Sum
  [PSCustomObject]@{ Name = $_.Name; GB = [math]::Round($size/1GB,2) }
} | Sort-Object GB -Descending | Select -First 20 | Format-Table

# Windows Update cache
Get-ChildItem C:\Windows\SoftwareDistribution\Download -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum | % { "WU cache: $([math]::Round($_.Sum/1GB,2)) GB" }

# Component store (WinSxS) — leitura via DISM (read-only quando só Analyze)
DISM /Online /Cleanup-Image /AnalyzeComponentStore

# Temp folders
Get-ChildItem $env:TEMP, C:\Windows\Temp -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum | % { "Temp: $([math]::Round($_.Sum/1GB,2)) GB" }

# Recycle Bin total
(New-Object -ComObject Shell.Application).Namespace(0xA).Items() | Measure-Object -Property Size -Sum | % { "Recycle Bin: $([math]::Round($_.Sum/1GB,2)) GB" }

# WSL VHDX (Linux subsystem disks)
Get-ChildItem $env:LOCALAPPDATA\Packages -Filter "*.vhdx" -Recurse -ErrorAction SilentlyContinue | Select FullName, @{N='GB';E={[math]::Round($_.Length/1GB,2)}}
Get-ChildItem $env:LOCALAPPDATA\Docker -Filter "*.vhdx" -Recurse -ErrorAction SilentlyContinue | Select FullName, @{N='GB';E={[math]::Round($_.Length/1GB,2)}}

# Browser caches Chromium (Chrome/Edge/Brave/Comet/Opera/Vivaldi)
$browsers = @(
  "$env:LOCALAPPDATA\Google\Chrome\User Data",
  "$env:LOCALAPPDATA\Microsoft\Edge\User Data",
  "$env:LOCALAPPDATA\BraveSoftware\Brave-Browser\User Data",
  "$env:LOCALAPPDATA\Perplexity\Comet\User Data",
  "$env:APPDATA\Opera Software\Opera Stable",
  "$env:LOCALAPPDATA\Vivaldi\User Data"
)
foreach ($b in $browsers) {
  if (Test-Path $b) {
    $size = (Get-ChildItem $b -Recurse -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum).Sum
    Write-Output "$b — $([math]::Round($size/1GB,2)) GB"
  }
}

# Visual Studio / .NET / NuGet
Get-ChildItem $env:USERPROFILE\.nuget\packages -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum | % { "NuGet packages: $([math]::Round($_.Sum/1GB,2)) GB" }
Get-ChildItem "$env:LOCALAPPDATA\Microsoft\VisualStudio" -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum | % { "VS data: $([math]::Round($_.Sum/1GB,2)) GB" }

# Android SDK / emulators
if (Test-Path "$env:USERPROFILE\.android") {
  Get-ChildItem "$env:USERPROFILE\.android" -Recurse -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum | % { "Android emulators/SDK: $([math]::Round($_.Sum/1GB,2)) GB" }
}

# Steam / Epic / game caches (se aplicável)
$steamPaths = @("C:\Program Files (x86)\Steam\steamapps\downloading", "C:\Program Files (x86)\Steam\steamapps\shadercache")
foreach ($p in $steamPaths) {
  if (Test-Path $p) {
    $size = (Get-ChildItem $p -Recurse -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum).Sum
    Write-Output "$p — $([math]::Round($size/1GB,2)) GB"
  }
}

# Processos rodando (pra checar antes de mexer em AppData de app)
# Substituir <AppName>: Get-Process <AppName> -ErrorAction SilentlyContinue
```

**Notas críticas:**
- Storage Settings (`ms-settings:storagesense`) inclui OneDrive sincronizado — não bate com soma física. Se houver discrepância grande, é cache OneDrive.
- `WinSxS` parece gigantesco mas a maior parte são hard links — **NUNCA** mexer manualmente, usar DISM.
- `hiberfil.sys` só pode ser apagado desativando hibernação (`powercfg /h off`).

### Fase 2 — Análise e categorização

| Categoria | Como tratar |
|---|---|
| **Cache puro / regenerável** (browser, GPU, blob storage) | `Remove-Item -Recurse -Force` direto |
| **Temp folders** (`%TEMP%`, `C:\Windows\Temp`) | Limpar via `Remove-Item`; arquivos em uso são pulados automaticamente |
| **Windows Update cache** (`SoftwareDistribution\Download`) | Parar serviço `wuauserv` → apagar → reiniciar |
| **Component store** (`WinSxS`) | `DISM /Online /Cleanup-Image /StartComponentCleanup /ResetBase` (admin) |
| **Hibernation file** | `powercfg /h off` desativa e remove (admin) |
| **Page file** | Configurar via System Properties → Performance Options (não apagar manualmente) |
| **Windows.old** | `cleanmgr.exe /sageset` ou Storage Sense |
| **WSL VHDX** | `wsl --shutdown` + `Optimize-VHD` (precisa Hyper-V module) |
| **Docker WSL2 VHDX** | Docker Desktop → Settings → Resources → "Clean / Purge data" |
| **Volume Shadow Copies** | `vssadmin delete shadows /for=C: /oldest` ou `/all` |
| **Browser dados de usuário** | NUNCA tocar (Login Data, Bookmarks, Cookies, etc.) |
| **Steam shadercache** | Pode apagar; Steam recria. NUNCA `steamapps\common` (jogos). |
| **OneDrive cache** | Settings → desativar sync → "Free up space" |
| **App inteiro** | `winget uninstall` ou Settings → Apps → Installed apps |

### Fase 3 — Apresentar vilões

Tabela markdown com Path, Tamanho, Descrição, Avaliação. Sempre incluir seção **"O que NÃO vou tocar"**.

Ranking: GB liberados / risco. Atacar primeiro alto ganho + baixo risco.

### Fase 4 — Execução

#### Caso A — Cache puro
```powershell
$path = "C:\path\to\cache"
$before = (Get-ChildItem $path -Recurse -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum).Sum
Write-Host "ANTES: $([math]::Round($before/1GB,2)) GB"
Remove-Item $path\* -Recurse -Force -ErrorAction SilentlyContinue
Write-Host "DEPOIS: Apagado ✓"
Get-PSDrive C | Format-Table Name, @{N='Free(GB)';E={[math]::Round($_.Free/1GB,1)}}
```

#### Caso B — Recycle Bin primeiro (deleções grandes/arriscadas)
```powershell
# Mover pra Recycle Bin via COM
$shell = New-Object -ComObject Shell.Application
$item = $shell.Namespace(0).ParseName("C:\path\to\folder")
$item.InvokeVerb("delete")
```

Ou usar `Microsoft.VisualBasic`:
```powershell
Add-Type -AssemblyName Microsoft.VisualBasic
[Microsoft.VisualBasic.FileIO.FileSystem]::DeleteDirectory(
  "C:\path\to\folder",
  'OnlyErrorDialogs',
  'SendToRecycleBin'
)
```

Depois de dias de confirmação:
```powershell
Clear-RecycleBin -Force
```

#### Caso C — Remoção completa de app

```powershell
# 1. Verificar app não está rodando
Get-Process <AppName> -ErrorAction SilentlyContinue

# 2. Desinstalar via winget (preferido)
winget uninstall <AppId>

# Ou via legacy:
Get-Package -Name "*<AppName>*" | Uninstall-Package

# 3. Mapear leftovers
$candidates = @(
  "$env:LOCALAPPDATA\<Vendor>",
  "$env:LOCALAPPDATA\<AppName>",
  "$env:APPDATA\<Vendor>",
  "$env:APPDATA\<AppName>",
  "$env:PROGRAMDATA\<Vendor>",
  "$env:USERPROFILE\.<AppName>"
)
foreach ($p in $candidates) {
  if (Test-Path $p) {
    $size = (Get-ChildItem $p -Recurse -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum).Sum
    Write-Output "$p — $([math]::Round($size/1MB,1)) MB"
  }
}

# 4. Após confirmação:
foreach ($p in $candidates) {
  if (Test-Path $p) { Remove-Item $p -Recurse -Force -ErrorAction SilentlyContinue }
}

# 5. Registry leftovers (cuidado, só sugerir, não executar sem extra confirmação)
# HKCU:\Software\<Vendor>
# HKLM:\Software\<Vendor>
# Recomendação: usar Revo Uninstaller ou IObit Uninstaller pra GUI mais segura
```

#### Caso D — Windows Update cache
```powershell
# Admin requerido
Stop-Service wuauserv -Force
Stop-Service bits -Force
Remove-Item C:\Windows\SoftwareDistribution\Download\* -Recurse -Force -ErrorAction SilentlyContinue
Start-Service wuauserv
Start-Service bits
```

#### Caso E — Component store cleanup
```powershell
# Admin requerido
DISM /Online /Cleanup-Image /AnalyzeComponentStore
# Se "Component Store Cleanup Recommended : Yes":
DISM /Online /Cleanup-Image /StartComponentCleanup /ResetBase
```

#### Caso F — Hibernation file
```powershell
# Admin requerido
# CUIDADO: desativa Fast Startup. Avisar usuário.
powercfg /h off
# Pra reativar:
# powercfg /h on
```

#### Caso G — Volume Shadow Copies (System Restore)
```powershell
# Lista
vssadmin list shadows /for=C:

# Apaga só o mais velho (mantém alguns checkpoints recentes)
vssadmin delete shadows /for=C: /oldest

# Apaga TODOS (sem volta — perde System Restore points)
# Confirmar com usuário antes!
vssadmin delete shadows /for=C: /all
```

#### Caso H — WSL VHDX shrink
```powershell
# 1. Encontrar
$vhdx = (Get-ChildItem $env:LOCALAPPDATA\Packages -Filter "ext4.vhdx" -Recurse).FullName

# 2. Shutdown WSL
wsl --shutdown

# 3. Shrink (precisa Hyper-V module — Windows Pro/Enterprise)
Optimize-VHD -Path $vhdx -Mode Full

# Alternativa via diskpart (Home):
# diskpart
#   select vdisk file="<caminho>"
#   attach vdisk readonly
#   compact vdisk
#   detach vdisk
#   exit
```

#### Caso I — Storage Sense (built-in macOS-like)
```powershell
# Abrir Storage Sense Settings
start ms-settings:storagesense

# Rodar Storage Sense agora (PowerShell)
# Não tem API direta, mas pode disparar via:
Get-ScheduledTask -TaskName "StorageSense" | Start-ScheduledTask
```

### Fase 5 — Verificação

Após cada operação:
1. Confirmar pasta sumiu (`Test-Path` ou `Get-ChildItem`)
2. Mostrar drive C: antes/depois
3. Acumular resumo da sessão

## Barra de progresso (para deleções > 5 GB)

```powershell
$target = "C:\path\to\big"
$totalKB = [int]((Get-ChildItem $target -Recurse -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum).Sum / 1KB)
$width = 30
$startTime = Get-Date

# Lança remove em background
$job = Start-Job -ScriptBlock { param($p) Remove-Item $p -Recurse -Force -ErrorAction SilentlyContinue } -ArgumentList $target

while ((Get-Job -Id $job.Id).State -eq "Running") {
  if (-not (Test-Path $target)) { break }
  $currentKB = [int]((Get-ChildItem $target -Recurse -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum).Sum / 1KB)
  $removedKB = $totalKB - $currentKB
  if ($removedKB -lt 0) { $removedKB = 0 }
  $pct = [math]::Min(100, [math]::Floor($removedKB * 100 / $totalKB))
  $filled = [math]::Floor($pct * $width / 100)
  $empty = $width - $filled
  $bar = ('█' * $filled) + ('░' * $empty)
  $removedGB = [math]::Round($removedKB / 1MB, 1)
  $currentGB = [math]::Round($currentKB / 1MB, 1)
  $elapsed = ((Get-Date) - $startTime).TotalSeconds
  $eta = if ($pct -gt 0 -and $pct -lt 100) { [math]::Floor($elapsed * (100 - $pct) / $pct) } else { 0 }
  Write-Host "[$bar] $pct% — $removedGB GB apagados | restam $currentGB GB | ETA ~${eta}s"
  Start-Sleep -Seconds 4
}

Remove-Job $job
$avail = [math]::Round((Get-PSDrive C).Free / 1GB, 1)
Write-Host "[$('█' * $width)] 100% — Concluído | Disco livre: ${avail} GB"
```

## Catálogo de vilões conhecidos

### Browsers Chromium-based (Chrome, Edge, Brave, Comet, Opera, Vivaldi)

Estrutura típica em `%LOCALAPPDATA%\<Vendor>\<Browser>\User Data\Default\`:

**SAFE-TO-DELETE (caches puros):**
- `Service Worker\` — vilão principal, pode ter 3+ GB
- `Cache\`
- `Code Cache\`
- `GPUCache\`
- `DawnWebGPUCache\`
- `Shared Dictionary\`
- `blob_storage\`

**NUNCA TOCAR (dados do usuário):**
- `Login Data` (senhas)
- `Bookmarks` + `Bookmarks.bak`
- `Cookies`
- `Web Data` (autofill)
- `Preferences`
- `History`
- `Favicons`
- `Extensions\`
- `Local Extension Settings\`
- `IndexedDB\` (drafts de sites como Notion, Figma)
- `Local Storage\`
- `Sessions\`

⚠️ Browser DEVE estar fechado antes de apagar — SQLite pode corromper.

### Windows-específicos

| Vilão | Path | Como limpar |
|---|---|---|
| **Windows Update cache** | `C:\Windows\SoftwareDistribution\Download` | Parar `wuauserv` → apagar → reiniciar |
| **Windows.old** | `C:\Windows.old` | Storage Sense ou `cleanmgr` (admin) |
| **Hibernation file** | `C:\hiberfil.sys` | `powercfg /h off` (desativa Fast Startup) |
| **Component store** | `C:\Windows\WinSxS` | `DISM /Online /Cleanup-Image /StartComponentCleanup /ResetBase` |
| **Volume Shadow Copies** | (invisível) | `vssadmin delete shadows /for=C: /oldest` |
| **Temp folders** | `%TEMP%`, `C:\Windows\Temp` | `Remove-Item` direto |
| **Memory dumps** | `C:\Windows\Minidump`, `C:\Windows\MEMORY.DMP` | `Remove-Item` (admin) |
| **Prefetch** | `C:\Windows\Prefetch` | Raramente vale; SO regenera |
| **Recycle Bin** | `C:\$Recycle.Bin` | `Clear-RecycleBin -Force` |

### Dev tools

- **Visual Studio caches:** `%LOCALAPPDATA%\Microsoft\VisualStudio\<version>\ComponentModelCache`, `Designer`, `Extensions`
- **NuGet global packages:** `%USERPROFILE%\.nuget\packages` (re-baixa, mas pode demorar)
- **MSBuild output:** `bin\`, `obj\` em projetos (regenera no próximo build)
- **JetBrains caches:** `%LOCALAPPDATA%\JetBrains\<IDE>\caches`
- **node_modules:** projetos abandonados — `Get-ChildItem -Path C:\projects -Filter node_modules -Recurse -Directory`
- **Yarn/npm cache:** `%LOCALAPPDATA%\Yarn\Cache`, `%APPDATA%\npm-cache`

### WSL / Docker

- **WSL distros VHDX:** `%LOCALAPPDATA%\Packages\<distro>\LocalState\ext4.vhdx` — usar `Optimize-VHD` após `wsl --shutdown`
- **Docker Desktop:** `%LOCALAPPDATA%\Docker\wsl\data\ext4.vhdx` — Settings → Resources → Disk image size
- **Docker images/containers:** `docker system prune -a --volumes` (CUIDADO: remove tudo não usado)

### Games

- **Steam shader cache:** `C:\Program Files (x86)\Steam\steamapps\shadercache` — pode apagar, regenera
- **Steam downloading:** `C:\Program Files (x86)\Steam\steamapps\downloading` — só se download travado
- **Epic Games cache:** `%LOCALAPPDATA%\EpicGamesLauncher\Saved\webcache`
- **Battle.net cache:** `%PROGRAMDATA%\Battle.net\Cache`

### Mobile dev

- **Android emulators:** `%USERPROFILE%\.android\avd` — apagar AVDs não usados via Android Studio AVD Manager
- **Android SDK:** `%LOCALAPPDATA%\Android\Sdk\system-images` — system images antigas

### Cloud sync

- **OneDrive cache:** desativar sync via app → "Free up space" no menu de contexto da pasta
- **Google Drive cache:** `%LOCALAPPDATA%\Google\DriveFS`
- **Dropbox cache:** `%LOCALAPPDATA%\Dropbox\instance1\cache`

### Pastas que parecem cache mas NÃO são

- `C:\Windows\WinSxS\` — component store; usar DISM, nunca apagar direto
- `C:\Windows\System32\` — sistema
- `C:\ProgramData\Microsoft\Windows Defender\` — antivírus
- `%APPDATA%\Microsoft\Windows\` — perfil do usuário, configs
- `C:\Users\<user>\OneDrive\` — dados sincronizados (sem placeholder)
- `C:\Users\<user>\Documents\` — geralmente dados do usuário

## Tom de comunicação

- Explica o **WHY** antes de cada deleção.
- Apresenta tabelas markdown com tamanhos formatados (GB, MB).
- Para deleções grandes (>5 GB), oferece barra de progresso real-time.
- Sempre fecha cada operação com resumo: drive C: antes/depois + tabela acumulada.
- Pergunta cedo, decide tarde: dúvida = pergunta.
- Sempre avisa quando operação requer admin elevado.

## Anti-padrões (NUNCA FAZER)

- ❌ Apagar `C:\Windows\WinSxS` manualmente (usar DISM)
- ❌ Apagar `C:\Windows\System32` ou qualquer subpasta de sistema
- ❌ Apagar `pagefile.sys` manualmente (configurar via System Properties)
- ❌ `Remove-Item -Force` sem confirmação prévia em diretórios > 1 GB
- ❌ Apagar dados de browser (Login Data, Bookmarks, Cookies, etc.)
- ❌ Apagar `OneDrive`, `Google Drive`, `Dropbox` folders (são sincronizados — usar app pra desativar sync)
- ❌ Apagar `steamapps\common` (jogos instalados)
- ❌ Apagar registries manualmente sem backup (`reg export` antes)
- ❌ Forçar `Stop-Process` em processos do usuário sem pedir pra ele fechar
- ❌ Sugerir limpeza genérica sem diagnóstico prévio
- ❌ Apagar System Restore points sem confirmação (vssadmin /all)

## Quando encerrar

Quando o usuário ficar satisfeito ou quando os candidatos restantes tiverem GB/risco ruim:
1. Resumo final (tabela com tudo liberado)
2. `Get-PSDrive C` final
3. Pendências (ex: Storage Sense que ficou pra ele rodar manualmente)
4. Tarefas opcionais futuras (apps que poderiam ser revisados, Downloads, Steam library)
