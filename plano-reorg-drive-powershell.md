# Plano de Reorganização do Google Drive — Brief para Agente CLI (PowerShell)

> **Objetivo:** reorganizar o Google Drive (montado localmente via *Google Drive para Desktop*) em uma taxonomia limpa, movendo e removendo itens com **boa performance** e **reversibilidade**.
> **Público:** um agente de IA via CLI que vai **gerar e executar scripts PowerShell**.
> **Autor do contexto:** Jezer Portilho. Data de referência: 03/06/2026 (São Paulo / BRT).

---

## 0. Premissas de ambiente (o agente DEVE validar antes de tudo)

1. O Drive está montado pelo **Google Drive para Desktop**. Descobrir a raiz real (`$DriveRoot`), normalmente algo como `G:\Meu Drive` ou `G:\My Drive`. **Não** assumir a letra `G:` — detectar.
2. Identificar o **modo de sincronização**:
   - **Stream (espelho virtual):** arquivos podem estar "somente online" (placeholders). Mover/renomear é operação de **metadados** — rápido e **não baixa** o conteúdo. Excluir vai para a **Lixeira do Drive**.
   - **Mirror (espelho local):** arquivos já estão em disco → moves são *rename* NTFS puro (mais rápido ainda).
   - Em ambos os modos o plano funciona; só muda a velocidade.
3. **Não** marcar pastas como "Disponível offline" só para mover — isso forçaria download de GBs sem necessidade. Mover online-only já basta.
4. Garantir que o ícone do Drive esteja **"Sincronizado"** (sem fila pendente) antes de iniciar e **entre as fases**, para evitar conflitos de sync.
5. Fechar apps que travam arquivos (VS Code, Git, Office) antes de mexer em `DocsIN`/`.git`.

---

## 1. Regras de performance (decisivas para o agente)

| Situação | Método recomendado | Por quê |
|---|---|---|
| Relocar uma **pasta inteira** dentro do mesmo volume | `Move-Item -LiteralPath` | *Rename* atômico no mesmo volume — instantâneo, sem copiar dados |
| Move falhou (caminho longo, arquivo travado, árvore enorme) | `robocopy "src" "dst" /MOVE /E /MT:16 /R:1 /W:1 /NP /NFL /NDL /LOG+:reorg.log` | Multithread, resiliente, com log; `/R:1 /W:1` evita travar em arquivo bloqueado |
| **Excluir árvore grande de ruído** (ex.: `.git` com milhares de objetos) | `Remove-Item -LiteralPath <pastaPai> -Recurse -Force` **uma vez** na pasta-pai | Excluir o nó pai joga a subárvore inteira na Lixeira do Drive de uma vez — muito mais rápido que arquivo a arquivo |
| Mover muitos arquivos soltos para o mesmo destino | Agrupar em **uma** chamada por destino | Reduz round-trips de sync |

Regras gerais:
- **Sempre** usar `-LiteralPath` (nunca `-Path` com wildcard) — há nomes com **acento** (`Março`) e **espaço no fim** (`Conjunto De documentos `).
- Habilitar **long paths**: `New-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem' -Name LongPathsEnabled -Value 1 -PropertyType DWORD -Force` ou usar prefixo `\\?\`.
- Operar em **lotes por fase** e aguardar o sync estabilizar entre fases.

---

## 2. Segurança, reversibilidade e idempotência

1. **Fase de simulação obrigatória:** rodar tudo primeiro com `-WhatIf` (e `robocopy /L`) e **gerar um manifesto CSV** (`origem,destino,acao,tamanho,tipo`) para revisão humana **antes** de executar de verdade.
2. **Exclusões vão para a Lixeira do Drive** (recuperável por ~30 dias). **Não** esvaziar a lixeira automaticamente — deixar para o Jezer confirmar e liberar espaço manualmente.
3. **Logar tudo** (`Start-Transcript` + log do robocopy).
4. **Idempotência:** antes de criar pasta, checar `Test-Path`; antes de mover, checar se já está no destino; se houver colisão de nome no destino, **não sobrescrever** — sufixar (` (dup)`) ou pular e registrar no log.
5. **Arquivos nativos do Google** (`.gdoc`, `.gsheet`, `.gform`, `.gslides`) aparecem localmente como **ponteiros minúsculos**. Mover o ponteiro **move o documento** no Drive (ok). Excluir o ponteiro **manda o documento para a lixeira**. **Nunca** tentar abrir/parsear esses ponteiros como conteúdo.
6. Validar espaço/quarentena: nada de `Remove-Item` permanente (`-Force` aqui só ignora read-only/oculto; a remoção continua indo para a lixeira do Drive).

---

## 3. Taxonomia-alvo (criar na raiz do Drive)

Prefixos numéricos são opcionais (ajudam a ordenação). Ajustar nomes a gosto.

```
$DriveRoot\
├── 00_Pessoal-Familia\
├── 01_Financeiro-IR\
├── 02_Carreira\
├── 03_Condominio-ReservaIpanema\
├── 04_Projetos-Tech\
├── 05_Arquivo-Legado\
└── 99_Revisar-Excluir\        # staging; só ruído óbvio é excluído direto
```

---

## 4. Mapeamento (regras de destino)

> O agente deve **gerar o inventário real** com `Get-ChildItem -LiteralPath $DriveRoot -Recurse -Force` e aplicar estas regras. Caminhos abaixo são relativos a `$DriveRoot` e podem variar — casar por **nome** de pasta/arquivo.

### 4.1 Arquivos soltos na raiz → destino
| Item (raiz) | Ação | Destino |
|---|---|---|
| `Currículo - PT V1` (.gdoc) | MOVE | `02_Carreira` |
| `Cópia de Currículo - Volvo` (.gdoc) | MOVE | `02_Carreira` |
| `Cópia traduzida de Currículo - PT` (.gdoc) | MOVE | `02_Carreira` |
| `ART - Jezer Portilho Dos Santos-assinar.pdf` | MOVE | `02_Carreira` *(revisar: pode ser Condomínio)* |
| `Conversa do WhatsApp com Res. do IPANEMA...` (.zip) | MOVE | `03_Condominio-ReservaIpanema` |
| `semafororealista_transparente.png` | MOVE | `04_Projetos-Tech\Semaforo-assets` |
| `Documento sem título` (44 KB) | MOVE | `99_Revisar-Excluir` *(tem conteúdo → revisar antes de apagar)* |
| `Documento sem título` (~1 KB) | MOVE | `99_Revisar-Excluir` *(vazio → candidato a excluir)* |
| `Formulário sem título` (.gform vazio) | MOVE | `99_Revisar-Excluir` *(candidato a excluir)* |
| `WG-MVPN-SSL.exe` (instalador VPN) | MOVE | `99_Revisar-Excluir` *(candidato a excluir)* |

### 4.2 Pastas → destino
| Origem (relativa) | Ação | Destino |
|---|---|---|
| `Documentos\Particular\IR` (com `IR2023`, `IR2025`) | MOVE (promover) | `01_Financeiro-IR\` |
| `Documentos\copias\Fisica\Fisica jezer\passaporte` | MOVE | `00_Pessoal-Familia\passaporte` |
| `Documentos\copias\Fisica\FILHOS` (MARY, ARTHUR) | MOVE | `00_Pessoal-Familia\FILHOS` |
| `Documentos\ligia` (escritura) | MOVE | `00_Pessoal-Familia\ligia` |
| `Documentos\fotoRosto` | MOVE | `00_Pessoal-Familia\fotoRosto` |
| `Documentos\copias\Fisica\Fisica jefter` | MOVE | `05_Arquivo-Legado` *(revisar)* |
| `Documentos\Conjunto De documentos ` *(espaço no fim!)* | MOVE | `05_Arquivo-Legado` |
| `Documentos\copias` (resto, após extrair `Fisica`) | MOVE | `05_Arquivo-Legado` |
| `Documentos\projetos\FitInsurTheme` | MOVE | `04_Projetos-Tech` |
| `Projetos` (raiz, 2025) | MERGE | `04_Projetos-Tech` *(ver colisão em §5)* |
| `Condominio` (com `GeminiSiteV2`) | MOVE | `03_Condominio-ReservaIpanema` |
| `Documentos` (balde 2010, o que sobrar) | MOVE | `05_Arquivo-Legado` |

### 4.3 Ruído para EXCLUIR (Lixeira do Drive)
| Origem | Ação |
|---|---|
| `Documentos\portilho Silva\DocsIN\.git` (centenas de pastas: `objects`, `refs`, `heads`, `tags`, `hooks`, `pack`, hex `a6/ac/01/...`) | **DELETE recursivo na pasta `.git`** |
| `Documentos\portilho Silva\DocsIN\.vscode` | DELETE |
| Após limpar acima: `Documentos\portilho Silva\DocsIN` (sobra `IN`, `plano de fundo`) | MOVE → `04_Projetos-Tech\DocsIN` |

> ⚠️ **Decisão em aberto (default aplicado):** o repositório `.git` está marcado para **exclusão** por ser artefato de código sincronizado por engano. Se o Jezer quiser preservar histórico, trocar para MOVE → `05_Arquivo-Legado\DocsIN-git-backup`. O agente deve deixar isso como **parâmetro no topo do script** (`$GitAction = 'Delete' | 'Archive'`).

> **Decisão em aberto (default aplicado):** os 3 currículos vão **todos** para `02_Carreira`. Marcar a versão mais recente (`Currículo - PT V1` vs as cópias) como principal e mover as demais para `02_Carreira\versoes-antigas`. Parâmetro: `$ResumeStrategy = 'AllTogether' | 'KeepLatest'`.

---

## 5. Casos especiais que o agente DEVE tratar

1. **Colisão de nomes `projetos` / `Projetos`:** existem duas — uma na raiz e outra dentro de `Documentos`. Ao consolidar em `04_Projetos-Tech`, mesclar conteúdo; se houver subpasta/arquivo de mesmo nome, comparar data/tamanho e sufixar o duplicado, nunca sobrescrever.
2. **Espaço no fim do nome:** `Conjunto De documentos ` — só funciona com `-LiteralPath`.
3. **Acentos:** `Março`, `Currículo`, `Cópia` — garantir console em UTF-8 (`[Console]::OutputEncoding = [Text.Encoding]::UTF8`).
4. **Caminhos longos** na árvore `.git` — habilitar long paths ou `\\?\`.
5. **Ponteiros do Google** (.gdoc/.gform) — mover ok; nunca abrir como conteúdo.
6. **Arquivos travados** — `robocopy /R:1 /W:1` para não pendurar.

---

## 6. Fluxo de execução em fases (o script deve seguir esta ordem)

- **Fase 0 — Inventário & Plano:** detectar `$DriveRoot` e modo de sync; `Get-ChildItem -Recurse -Force`; gerar `manifesto.csv` (origem→destino→ação) e `inventario_antes.csv`. **Parar e pedir aprovação** (ou rodar com `-WhatIf`).
- **Fase 1 — Criar taxonomia:** criar as 7 pastas-alvo (idempotente, `Test-Path`).
- **Fase 2 — Mover pastas** (§4.2) com `Move-Item`; fallback `robocopy /MOVE`. Aguardar sync.
- **Fase 3 — Mover arquivos soltos** (§4.1), agrupados por destino. Aguardar sync.
- **Fase 4 — Excluir ruído** (§4.3) — excluir o nó pai `.git`/`.vscode` de uma vez. **Não** esvaziar lixeira.
- **Fase 5 — Verificação:** novo inventário (`inventario_depois.csv`); assert: raiz contém **só** as 7 pastas + nada solto (exceto o que foi para `99_`); imprimir contagens antes/depois e gerar `relatorio_final.csv`.

---

## 7. Esqueleto sugerido do script (o agente expande)

```powershell
[CmdletBinding(SupportsShouldProcess)]
param(
  [string]$DriveRoot,                         # auto-detectar se vazio
  [ValidateSet('Delete','Archive')] $GitAction = 'Delete',
  [ValidateSet('AllTogether','KeepLatest')] $ResumeStrategy = 'AllTogether',
  [switch]$Execute                            # sem -Execute => apenas simula (-WhatIf)
)

[Console]::OutputEncoding = [Text.Encoding]::UTF8
Start-Transcript -Path ".\reorg_transcript.log" -Append

# Fase 0: detectar raiz, modo de sync, gerar inventário + manifesto CSV
# Fase 1: criar pastas-alvo (Test-Path antes de New-Item)
# Fase 2..4: Move-Safe / Remove-Safe (com -WhatIf quando !$Execute)
# Fase 5: inventário "depois" + asserts + relatório

function Move-Safe {
  param([string]$Src,[string]$Dst)
  if (-not (Test-Path -LiteralPath $Src)) { return }
  if (-not (Test-Path -LiteralPath $Dst)) { New-Item -ItemType Directory -LiteralPath $Dst -Force | Out-Null }
  try   { Move-Item -LiteralPath $Src -Destination $Dst -WhatIf:(!$Execute) -ErrorAction Stop }
  catch { robocopy "$Src" "$(Join-Path $Dst (Split-Path $Src -Leaf))" /MOVE /E /MT:16 /R:1 /W:1 /NP /NFL /NDL /LOG+:.\robocopy.log }
}

function Remove-Noise {
  param([string]$Path)
  if (Test-Path -LiteralPath $Path) { Remove-Item -LiteralPath $Path -Recurse -Force -WhatIf:(!$Execute) }
}

Stop-Transcript
```

---

## 8. Checklist de aceite

- [ ] Raiz do Drive tem apenas as 7 pastas-alvo (sem arquivos soltos fora de `99_`).
- [ ] `IR2023`/`IR2025` acessíveis em `01_Financeiro-IR` (sem 3 níveis de aninhamento).
- [ ] Nenhuma pasta `.git`/`.vscode`/hex de objetos restante fora da lixeira.
- [ ] Manifesto e relatório (antes/depois) gerados.
- [ ] Nada excluído permanentemente — tudo recuperável na Lixeira do Drive.
- [ ] Nenhum documento Google corrompido (ponteiros apenas movidos).
