---
name: cleanmymac
description: 'Use esta skill sempre que o usuário invocar `/cleanmymac`, ou pedir pra "limpar o mac", "limpar disco", "liberar espaço", "fazer audit do disco", "analisar storage", "limpar caches", "atacar system data", "investigar o que ocupa espaço", ou variações como "meu disco tá cheio", "tô sem espaço", "que diabos tá ocupando X GB". Também acione quando o usuário compartilhar print do Storage Settings do macOS ou pedir pra investigar uma categoria específica (Documents, System Data, Applications, Photos). Playbook caso-a-caso negociado — diagnóstico read-only primeiro, identifica vilões reais do setup específico do usuário, propõe deleções caso-a-caso, move pra Lixeira quando há risco, barra de progresso real-time pra deleções grandes. Respeita as regras absolutas da memória do usuário (FLUTTER INTOCÁVEL).'
---

# /cleanmymac — Audit e limpeza negociada do macOS

Skill pra fazer auditoria de disco caso-a-caso no Mac do usuário, identificando os vilões REAIS do setup específico dele (não lista genérica) e propondo deleções com confirmação a cada passo. Inspirado em DaisyDisk + CleanMyMac, mas com inteligência situacional.

## Regras absolutas (LEIA PRIMEIRO)

1. **FLUTTER INTOCÁVEL** — Nunca, jamais sugerir ou executar remoção de `~/development/flutter`, `~/.pub-cache`, `.dart_tool/` ou qualquer artefato Flutter. Mesmo se aparecerem em listagens de tamanho, marcar como 🔴 ou omitir. (Regra registrada em memória após incidente 2026-05-11.)
2. **Confirmação obrigatória antes de toda deleção** — apresentar tabela com tamanhos e justificativa, esperar OK do usuário.
3. **Lixeira primeiro pra deleções grandes ou arriscadas** — `mv → ~/.Trash/` em vez de `rm -rf`. Rede de segurança. Apaga definitivo só depois de confirmação.
4. **App fechado antes de apagar Application Support/Caches do app** — usar `pgrep -lf` pra confirmar. Pedir pro usuário fechar via Cmd+Q (nunca matar processo dele).
5. **Dados de usuário NUNCA são tocados** — listados na seção abaixo.
6. **iCloud Photos/Notes**: instruir System Settings → Apple ID → iCloud → toggle OFF → "Delete from Mac". macOS limpa sozinho. Não tocar via terminal.

## Playbook (5 fases)

### Fase 1 — Diagnóstico read-only

Rodar bateria de checks em paralelo. Tudo só leitura, nenhum efeito colateral.

```bash
# Volume Data REAL (não system snapshot)
df -h ~ | head -2

# APFS info + snapshots Time Machine locais
diskutil info / | grep -E "Volume Used|Volume Free|Container Free|Snapshot"
tmutil listlocalsnapshots / 2>/dev/null
tmutil listlocalsnapshotdates / 2>/dev/null

# Caches do usuário
du -sh ~/Library/Caches 2>/dev/null
du -sh ~/Library/Caches/* 2>/dev/null | sort -hr | head -15

# Application Support (geralmente o maior vilão)
du -sh ~/Library/Application\ Support/* 2>/dev/null | sort -hr | head -15

# Group Containers (apps com extensões + iCloud)
du -sh ~/Library/Group\ Containers/* 2>/dev/null | sort -hr | head -15

# Xcode / iOS dev
du -sh ~/Library/Developer/Xcode/DerivedData 2>/dev/null
du -sh ~/Library/Developer/Xcode/Archives 2>/dev/null
du -sh ~/Library/Developer/Xcode/iOS\ DeviceSupport 2>/dev/null
du -sh ~/Library/Developer/CoreSimulator 2>/dev/null

# Containers do sistema
du -sh ~/Library/Containers 2>/dev/null

# Docker (se aplicável)
ls -lah ~/Library/Containers/com.docker.docker/Data/vms/ 2>/dev/null

# Top diretórios em ~/Library
du -sh ~/Library/* 2>/dev/null | sort -hr | head -15

# Home folders
du -sh ~/Downloads 2>/dev/null
du -sh ~/Documents/* 2>/dev/null | sort -hr | head -15
du -sh ~/Movies 2>/dev/null
du -sh ~/Music 2>/dev/null
du -sh ~/Desktop 2>/dev/null
du -sh ~/Pictures/*.photoslibrary 2>/dev/null
```

**Notas críticas:**
- `df -h /` no macOS moderno mostra só o snapshot read-only do sistema. **Sempre use `df -h ~`** para ver o volume Data real.
- Storage Settings inclui iCloud sincronizado em "Documents" — não bate com `du` físico. Se houver discrepância grande, é cache iCloud (`FileProvider` + `CloudDocs`).

### Fase 2 — Análise e categorização

Categorizar os achados em:

| Categoria | Como tratar |
|---|---|
| **Cache puro / regenerável** (caches HTTP, GPU, Service Worker) | Apagar direto com `rm -rf`, sem Lixeira |
| **Cache iCloud** (CloudKit, FileProvider, CloudDocs) | Apagar direto, re-baixa sob demanda |
| **Caches de apps grandes** (Spotify, Discord, etc.) | Limpar pelo próprio app quando possível |
| **Dados de apps com iCloud sync** (Photos, Notes) | System Settings → desativar sync → Delete from Mac |
| **Apps inteiros pra desinstalar** | Padrão de remoção completa (ver abaixo) |
| **iOS Simulators** | `xcrun simctl delete all` (canônico) |
| **Xcode DerivedData/Archives** | Apagar direto |
| **Time Machine local snapshots** | `tmutil deletelocalsnapshots <date>` |
| **Pastas customizadas do usuário** | NUNCA tocar sem confirmação explícita |

### Fase 3 — Apresentar vilões

Mostrar tabela markdown com:
- Path
- Tamanho
- O que é (descrição clara)
- Avaliação (cache puro / dados / requer ação manual / etc.)

Sempre incluir uma seção **"O que NÃO vou tocar"** listando explicitamente os dados de usuário detectados. Transparência > velocidade.

Ranking de prioridade: **GB liberados / risco**. Atacar primeiro o que tem alto ganho + baixo risco.

### Fase 4 — Execução

Para cada vilão aprovado:

#### Caso A — Cache puro (rm direto)
```bash
echo "ANTES:" && du -sh <path>
rm -rf <path>
echo "DEPOIS:" && du -sh <path> 2>/dev/null || echo "Apagado ✓"
echo "Disco livre:" && df -h ~ | awk 'NR==2{print $4}'
```

#### Caso B — Deleção grande ou arriscada (Lixeira primeiro)
```bash
mv <path> ~/.Trash/
# Aguardar dias/sessões. Quando user confirmar, então:
rm -rf ~/.Trash/<basename>
```

#### Caso C — Remoção completa de app (padrão macOS)
Lista de paths a verificar e remover (substituir `<bundle>` pelo bundle ID, ex: `ru.keepcoder.Telegram`):

```bash
# 1. Verificar se app está rodando
pgrep -lf "<NomeApp>" 2>/dev/null

# 2. Mapear todos os artefatos
ls -lah /Applications/<NomeApp>.app
du -sh ~/Library/Group\ Containers/*<bundle>*
du -sh ~/Library/Containers/*<bundle>*
du -sh ~/Library/Application\ Support/*<NomeApp>* ~/Library/Application\ Support/*<bundle>*
du -sh ~/Library/Caches/*<bundle>*
ls -lah ~/Library/Preferences/*<bundle>*.plist
ls -lah ~/Library/Saved\ Application\ State/*<bundle>*
du -sh ~/Library/WebKit/*<bundle>*
du -sh ~/Library/HTTPStorages/*<bundle>*
ls ~/Library/LaunchAgents/*<bundle>* 2>/dev/null

# 3. Remover (após confirmação)
rm -rf /Applications/<NomeApp>.app
rm -rf ~/Library/Group\ Containers/*<bundle>*
rm -rf ~/Library/Containers/*<bundle>*
rm -rf ~/Library/Application\ Support/*<NomeApp>*
rm -rf ~/Library/Caches/*<bundle>*
rm -f  ~/Library/Preferences/*<bundle>*.plist
rm -rf ~/Library/Saved\ Application\ State/*<bundle>*
rm -rf ~/Library/WebKit/*<bundle>*
rm -rf ~/Library/HTTPStorages/*<bundle>*
rm -rf ~/Library/LaunchAgents/*<bundle>*
```

#### Caso D — iOS Simulators
```bash
xcrun simctl list devices available
xcrun simctl shutdown all 2>/dev/null
xcrun simctl delete all  # canônico, limpa metadados internos
```

#### Caso E — Time Machine local snapshots
```bash
tmutil listlocalsnapshots /
# Para cada snapshot:
tmutil deletelocalsnapshots <date>
```

### Fase 5 — Verificação

Após cada operação:
1. Confirmar pasta sumiu (`du -sh` ou `ls`)
2. Mostrar `df -h ~` antes/depois
3. Acumular resumo da sessão (tabela com tudo já liberado)

## Barra de progresso (para deleções > 5 GB)

Para `rm -rf` em pastas grandes, usar Monitor com loop calculando % real-time:

```bash
TARGET="<path absoluto>"
TOTAL_KB=<tamanho original em KB>
WIDTH=30
START_TIME=$(date +%s)

# Lançar rm em background
rm -rf "$TARGET" &

# Monitor:
while true; do
  if [ ! -e "$TARGET" ]; then
    BAR=$(printf '█%.0s' $(seq 1 $WIDTH))
    ELAPSED=$(($(date +%s) - START_TIME))
    AVAIL=$(df -h ~ | awk 'NR==2{print $4}')
    echo "[$BAR] 100% — Concluído em ${ELAPSED}s | Disco livre: ${AVAIL}"
    break
  fi
  CURRENT_KB=$(du -sk "$TARGET" 2>/dev/null | awk '{print $1}')
  CURRENT_KB=${CURRENT_KB:-0}
  REMOVED_KB=$((TOTAL_KB - CURRENT_KB))
  [ $REMOVED_KB -lt 0 ] && REMOVED_KB=0
  PCT=$((REMOVED_KB * 100 / TOTAL_KB))
  [ $PCT -gt 100 ] && PCT=100
  FILLED=$((PCT * WIDTH / 100))
  EMPTY=$((WIDTH - FILLED))
  BAR_F=""; BAR_E=""
  [ $FILLED -gt 0 ] && BAR_F=$(printf '█%.0s' $(seq 1 $FILLED))
  [ $EMPTY -gt 0 ] && BAR_E=$(printf '░%.0s' $(seq 1 $EMPTY))
  REMOVED_GB=$(awk "BEGIN{printf \"%.1f\", $REMOVED_KB/1048576}")
  CURRENT_GB=$(awk "BEGIN{printf \"%.1f\", $CURRENT_KB/1048576}")
  ELAPSED=$(($(date +%s) - START_TIME))
  if [ $PCT -gt 0 ] && [ $PCT -lt 100 ]; then
    ETA=$((ELAPSED * (100 - PCT) / PCT))
    echo "[${BAR_F}${BAR_E}] ${PCT}% — ${REMOVED_GB}/${TOTAL_GB} GB apagados | restam ${CURRENT_GB} GB | ETA ~${ETA}s"
  fi
  sleep 4
done
```

Usar via tool `Monitor` com `timeout_ms` adequado (600000ms = 10 min cobre até ~50 GB).

## Catálogo de vilões conhecidos

### Browsers Chromium-based (Comet, Arc, Chrome, Brave, Edge)

Estrutura típica em `~/Library/Application Support/<Browser>/Default/`:

**SAFE-TO-DELETE (caches puros):**
- `Service Worker/` — pode ter MUITO (3+ GB), é o vilão principal
- `Shared Dictionary/`
- `blob_storage/`
- `GPUCache/`
- `DawnWebGPUCache/`
- `Code Cache/`

**NUNCA TOCAR (dados do usuário):**
- `Login Data` (senhas)
- `Bookmarks` + `Bookmarks.bak`
- `Cookies` (sessões logadas)
- `Web Data` (autofill)
- `Preferences`
- `History`
- `Favicons`
- `Extensions/` (extensões instaladas)
- `Local Extension Settings/`
- `IndexedDB/` (alguns sites guardam drafts/dados aqui — Notion, Figma, etc.)
- `Local Storage/`
- `Sessions/`

⚠️ Browser DEVE estar fechado (Cmd+Q) antes de apagar — SQLite pode corromper.

### iOS Simulators (Xcode)
- `~/Library/Developer/CoreSimulator/Devices` — frequentemente 4-20 GB
- Comando canônico: `xcrun simctl delete all` (limpa metadados internos também)

### Xcode artifacts
- `~/Library/Developer/Xcode/DerivedData` — build artifacts, regenera
- `~/Library/Developer/Xcode/Archives` — só se você não precisar de archives antigos
- `~/Library/Developer/Xcode/iOS DeviceSupport` — debug symbols, regenera

### iCloud caches (SAFE, regeneram)
- `~/Library/Caches/CloudKit`
- `~/Library/Application Support/FileProvider`
- `~/Library/Application Support/CloudDocs`

### Apple Photos / Notes
**Não tocar via terminal**. Caminho correto:
- System Settings → Apple ID → iCloud → Photos/Notes → toggle OFF
- Quando o macOS perguntar, escolher **"Delete from Mac"**
- macOS limpa automaticamente (3-30 GB dependendo do uso)
- Se for Photos e o sync já foi desativado mas a biblioteca local persiste: `~/Pictures/*.photoslibrary` pode ser movida pra Lixeira (após confirmar que tudo está no iCloud)

### Time Machine local snapshots
- `tmutil listlocalsnapshots /` — listar
- `tmutil deletelocalsnapshots <date>` — apagar (precisa data exata do snapshot)
- Pode ter 20-50 GB invisíveis no Storage Settings

### Docker.raw (se aplicável)
- `~/Library/Containers/com.docker.docker/Data/vms/0/data/Docker.raw` — VM image que só cresce
- Pra encolher: Docker Desktop → Settings → Resources → Disk image size → reduzir + Apply (recria a imagem menor)

### Sleep image / swap (root, cuidado)
- `/private/var/vm/sleepimage` — tamanho da RAM, regenera no próximo sleep
- `/private/var/vm/swapfile*` — swap, macOS gerencia
- Tocar nesses requer `sudo` e raramente compensa o risco

### Pastas que parecem caches mas NÃO são
- `~/Library/Biome/` — Siri/intelligence, **deixar quieto**
- `~/Library/Mobile Documents/` — iCloud Drive, dados sincronizados
- `~/Library/Keychains/` — senhas do sistema, **NUNCA**
- `~/Library/Mail/` — emails sincronizados (se IMAP, OK; se POP, são únicos)
- `~/Library/Daemon Containers/` — system daemons

## Tom de comunicação

- Explica o **WHY** antes de cada deleção — quero que o usuário entenda o que está saindo.
- Apresenta tabelas markdown com tamanhos formatados (GB, MB).
- Para deleções grandes (>5 GB), oferece barra de progresso real-time.
- Sempre fecha cada operação com resumo: `df -h ~` antes/depois + tabela acumulada da sessão.
- Pergunta cedo, decide tarde: se houver qualquer dúvida sobre "dados ou cache", pergunta antes de apagar.

## Anti-padrões (NUNCA FAZER)

- ❌ Apagar Flutter ou pub-cache (regra absoluta)
- ❌ `rm -rf` sem confirmação prévia em diretórios > 1 GB
- ❌ Tocar em `~/Library/Keychains/`, `~/Library/Mail/`, `~/Library/Mobile Documents/`
- ❌ Apagar dados de browser (Login Data, Bookmarks, Cookies, etc.)
- ❌ Apagar Photos Library sem confirmar que fotos estão no iCloud
- ❌ Apagar Notes attachments diretamente (sempre via System Settings)
- ❌ Forçar `kill` em processos do usuário sem pedir pra ele dar Cmd+Q
- ❌ Sugerir limpeza genérica sem diagnóstico prévio (o ponto da skill é caso-a-caso)

## Quando encerrar

Quando o usuário ficar satisfeito ou quando os candidatos restantes tiverem GB/risco ruim, encerra com:
1. Resumo final da sessão (tabela com tudo liberado)
2. `df -h ~` final
3. Eventuais pendências (ex: System Settings que ficou pra ele fazer)
4. Tarefas opcionais futuras (apps que ele poderia revisar com calma, Downloads pra limpar manualmente, etc.)
