# Plano ê¢â¬â Telemetria de agentes (Frente C)

Painel **local** que mostra, ao vivo, tudo o que os agentes estêÂ£o fazendo na mêÂ¡quina: cada sessêÂ£o do
Claude Code (terminais, bots, `claude -p`, loops longos, subagentes) e do Codex CLI.

Status (13/10/2026): **T0ê¢â¬âT5 prontas e instaladas** nesta mêÂ¡quina. T6ê¢â¬âT8 ainda sêÂ£o plano.

| PeêÂ§a | Arquivo |
|---|---|
| Coletor por hook | `hook/registra.py` |
| Instalar / remover | `instalar.sh` / `desinstalar.sh` (via `ajusta_hooks.py`, com backup do `settings.json`) |
| ManutenêÂ§êÂ£o (resumo, limpeza 30 dias, WAL) | `manutencao.py` + `systemd/telemetria-manutencao.timer` (de hora em hora) |
| Painel | `painel/servidor.py` + `painel/painel.html` ê¢â â **http://127.0.0.1:8766/** e **http://192.168.2.99:8766/** (serviêÂ§o `telemetria-painel`; sêÂ³ atende 127.0.0.0/8 e 192.168.2.0/24) |
| T4 êÂ· OpenTelemetry | `painel/coletores.py` ê¢â¬â receptor OTLP em 127.0.0.1:4318; variêÂ¡veis `OTEL_*` no `env` do `settings.json` |
| T5 êÂ· Codex | `painel/coletores.py` ê¢â¬â lêÂª `~/.codex/sessions/**/*.jsonl` a cada 15 s (êÂºltimos 2 dias, posiêÂ§êÂ£o guardada por arquivo) |
| Ligar / desligar | `./telemetria.sh ligar` êÂ· `./telemetria.sh desligar` êÂ· `./telemetria.sh status` |
| Banco | `~/.local/share/telemetria-agentes/eventos.db` (fora do repo) |

**Resultado da T0:** ~3ê¢â¬â5 mil chamadas de ferramenta por dia (pico 5,2 mil em 10/10) ê¢â â ~10 mil eventos/dia,
cerca de 2 MB/dia, ~60 MB em 30 dias. O Claude Code 2.1.296 entrega os eventos `SessionStart`, `SessionEnd`,
`UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `PermissionRequest`, `Notification`,
`Stop`, `SubagentStart`, `SubagentStop`, `PreCompact` e `PostCompact`, com `agent_id`/`agent_type` nos de subagente.
**SessêÂµes jêÂ¡ abertas passam a registrar no evento seguinte** (o Claude Code relêÂª os hooks); as que estêÂ£o
paradas aparecem no contador "abertas sem sinal ainda" atêÂ© fazerem algo.

**Regra de ouro:** tudo fica na mêÂ¡quina. Sem nuvem, sem API, sem conta nova. Guarda **nomes e nêÂºmeros**,
nunca o texto de prompt, resposta, comando completo ou conteêÂºdo de arquivo.

---

## 1. Perguntas que o painel responde

| Pergunta | De onde sai |
|---|---|
| Quais sessêÂµes estêÂ£o ativas agora, em que pasta, hêÂ¡ quanto tempo? | hooks `SessionStart`/`SessionEnd` + êÂºltimo evento |
| Qual estêÂ¡ **esperando resposta minha**? | hook `Notification` / `Stop` sem novo `UserPromptSubmit` |
| Qual estêÂ¡ **parada** (sem evento hêÂ¡ N min, processo vivo)? | êÂºltimo evento + `pgrep` |
| O que cada uma estêÂ¡ fazendo agora? | êÂºltimo `PreToolUse` (ferramenta, skill, MCP) |
| Quais processamentos longos estêÂ£o rodando e hêÂ¡ quanto tempo? | `PreToolUse` Bash sem `PostToolUse` correspondente |
| O que falhou hoje, por ferramenta / skill / MCP? | `PostToolUse` com erro |
| Quais skills, MCPs e subagentes eu uso de verdade? | contagens do histêÂ³rico |
| Quanto cada sessêÂ£o consumiu (tokens, custo estimado em preêÂ§o de API)? | OpenTelemetry (Fase T4) |

## 2. Arquitetura

```
Claude Code (todas as sessêÂµes) ê¢ââ¬ê¢âÂ¬ê¢ââ¬ hooks ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢âÂ
                                ê¢ââê¢ââ¬ OpenTelemetry (T4) ê¢ââ¬ê¢âÂ¤   (sêÂ³ localhost)
Codex CLI ê¢ââ¬ê¢ââ¬ sessêÂµes no disco (~/.codex/sessions) ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢âÂ¤
                                                       ê¢âÂ¼
                         ~/.local/share/telemetria-agentes/eventos.db  (SQLite, modo WAL)
                                                       ê¢ââ
                                     servidor 127.0.0.1 + 192.168.2.99 :8766
                                                       ê¢ââ
                                     painel HTML (atualiza a cada 5 s)
```

Por que funciona para **todas** as sessêÂµes: toda sessêÂ£o do Claude Code lêÂª `~/.claude/settings.json` ao
abrir. Um hook registrado ali vale para qualquer terminal, bot, `claude -p` e subagente.

## 3. Fases

### T0 ê¢â¬â Medir antes de mexer (sem instalar nada)
- Contar sessêÂµes ativas (`pgrep`), eventos por hora no histêÂ³rico (`~/.claude/projects/*.jsonl`) e
  estimar o volume de eventos por dia ê¢â â dimensiona o banco e a retenêÂ§êÂ£o.
- Listar os hooks jêÂ¡ existentes em `settings.json` para o novo **somar**, nêÂ£o substituir.
- Conferir quais eventos de hook a versêÂ£o instalada entrega e o formato do JSON de entrada.

### T1 ê¢â¬â Coletor por hook
- Script `telemetria/hook/registra.py` (Python puro, sem dependêÂªncia):
  lêÂª o JSON do hook na entrada padrêÂ£o ê¢â â filtra pela **lista de campos permitidos** ê¢â â grava uma linha
  no SQLite ê¢â â sai com cêÂ³digo 0 **sempre** (nunca bloqueia nem altera a sessêÂ£o).
- Eventos: `SessionStart`, `SessionEnd`, `UserPromptSubmit`, `PreToolUse`, `PostToolUse`,
  `Notification`, `Stop`, `SubagentStop`, `PreCompact`.
- Tetos: `timeout` 2 s no hook; gravaêÂ§êÂ£o com `busy_timeout` curto; se o banco estiver travado,
  descarta o evento (perder uma linha êÂ© melhor que travar uma sessêÂ£o).
- InstalaêÂ§êÂ£o: script `telemetria/instalar.sh` que **acrescenta** os hooks em `settings.json`
  (com backup datado) e `desinstalar.sh` que remove sêÂ³ os dele.

**Campos permitidos** (o resto êÂ© descartado):

| Campo | Exemplo |
|---|---|
| hora, evento | `2026-10-13T10:02:11`, `PostToolUse` |
| sessêÂ£o, sessêÂ£o-mêÂ£e | id da sessêÂ£o; subagente aponta para a mêÂ£e |
| pasta do projeto | sêÂ³ o nome da pasta (`ccdashboard`), nêÂ£o o caminho |
| origem | `terminal`, `headless`, `bot`, `subagente` (pela variêÂ¡vel de ambiente / pai do processo) |
| ferramenta | `Bash`, `Edit`, `Skill`, `mcp__<servidor>__<ferramenta>` |
| programa do Bash | sêÂ³ a primeira palavra (`ffmpeg`, `git`, `python3`) |
| skill / tipo de subagente | nome |
| duraêÂ§êÂ£o, sucesso/falha | ms; `ok`/`erro` |
| tipo do erro | categoria curta (`timeout`, `exitê¢â°Â 0`, `negado`), sem o texto |

**Nunca grava:** prompt, resposta, comando completo, caminho de arquivo, entrada/saêÂ­da de ferramenta,
texto de erro, e-mail, tokens/chaves.

### T2 ê¢â¬â Banco local
- Tabelas: `eventos` (bruto, 30 dias), `sessoes` (uma linha por sessêÂ£o, estado atual),
  `resumo_diario` (permanente).
- Limpeza diêÂ¡ria dos brutos com mais de 30 dias (timer `systemd --user`).
- Tamanho esperado: medido na T0; alvo < 200 MB.

### T3 ê¢â¬â Painel local
- `telemetria/painel/servidor.py` (Python, sêÂ³ `http.server` + `sqlite3`) escutando **sêÂ³ em
  127.0.0.1**, serviêÂ§o `systemd --user` com `MemoryMax`.
- PêÂ¡gina êÂºnica, padrêÂ£o ARCKHê (escuro/êÂ¢mbar), atualizaêÂ§êÂ£o a cada 5 s:

| Aba | Mostra |
|---|---|
| **Agora** | sessêÂµes ativas: pasta, origem, tempo, ferramenta atual; selo **esperando vocêÂª** / **parada hêÂ¡ N min** / **processando** |
| **Processamentos** | comandos longos em andamento e concluêÂ­dos hoje (programa, duraêÂ§êÂ£o, resultado) |
| **Falhas** | erros do dia por ferramenta, skill e MCP; taxa de falha; ligaêÂ§êÂ£o com `FALHAS.md` |
| **Uso** | skills, MCPs e subagentes mais usados; o que nunca êÂ© usado |
| **HistêÂ³rico** | sessêÂµes e ferramentas por dia (30 dias) |
| **SaêÂºde** | êÂºltimo evento recebido, tamanho do banco, hooks instalados |

### T4 ê¢â¬â OpenTelemetry local (tokens e custo)
- Receptor OTLP em `127.0.0.1:4318` dentro do mesmo servidor, gravando no mesmo SQLite.
- Bloco `env` em `~/.claude/settings.json` apontando para `http://127.0.0.1:4318`
  (mesmas variêÂ¡veis de `config/claude-code-otel.example.json`, sem chave e sem nuvem).
- Vale sêÂ³ para sessêÂµes abertas **depois** da mudanêÂ§a.
- Custo exibido = **estimativa em preêÂ§o de API**; na assinatura quem manda êÂ© o plano.

### T5 ê¢â¬â Codex
- Leitor das sessêÂµes do Codex em `~/.codex/sessions/` (arquivos JSONL), mesma lista de campos.
- Verificar se a versêÂ£o instalada do Codex CLI exporta OpenTelemetry; se exportar, usar o mesmo
  receptor da T4.

### T6 ê¢â¬â HistêÂ³rico antigo (opcional)
- Importar `~/.claude/projects/*.jsonl` (sêÂ³ nêÂºmeros) para o painel jêÂ¡ nascer com meses de dados.

### T7 ê¢â¬â Avisos (opcional)
- Rotina a cada 5 min: sessêÂ£o esperando resposta hêÂ¡ mais de 15 min, ferramenta com taxa de falha
  acima de 30%, processamento rodando hêÂ¡ mais de 1 h ê¢â â mensagem no bot.

### T8 ê¢â¬â Ponte para o Claude Dashboards (opcional)
- Mesmo esquema no Supabase (Frente B do `PLANO.md`) quando quiser ver o painel no app do Claude ou de
  outra mêÂ¡quina. Exige autorizaêÂ§êÂ£o explêÂ­cita de uso do Supabase.

## 4. CritêÂ©rios de pronto (cada fase)

| Fase | Comando ê¢â â resultado esperado |
|---|---|
| T1 | abrir sessêÂ£o nova, rodar 1 `ls` ê¢â â `sqlite3 eventos.db "select evento,ferramenta from eventos order by id desc limit 3"` mostra `PostToolUse Bash` |
| T1 | hook com banco travado (`flock` no arquivo) ê¢â â a sessêÂ£o segue normal, evento descartado, sem atraso > 2 s |
| T1 | `grep` no banco por um trecho de prompt digitado no teste ê¢â â **zero** ocorrêÂªncias |
| T1 | `desinstalar.sh` ê¢â â `settings.json` igual ao backup, hooks antigos intactos |
| T3 | `curl -s 127.0.0.1:8766/api/agora` lista as sessêÂµes ativas; contagem bate com `pgrep` (êÂ±1) |
| T3 | o painel nêÂ£o abre por outro IP da rede |
| T4 | sessêÂ£o nova ê¢â â aparecem tokens e modelo no painel em atêÂ© 60 s |
| T5 | uma execuêÂ§êÂ£o `codex exec` aparece na aba **Agora** / **HistêÂ³rico** |

## 5. Riscos e limites
- **Hook em todo evento:** precisa ser rêÂ¡pido (alvo < 50 ms). Medir na T1 com 100 eventos.
- **Hooks globais valem para todos os projetos:** instalaêÂ§êÂ£o sêÂ³ com "sim" do dono; desinstalador testado.
- **Nomes de skills, MCPs e pastas ficam no banco:** êÂ© local, mas nêÂ£o compartilhar o arquivo.
- **Eventos de hook mudam entre versêÂµes do Claude Code:** o coletor ignora campo desconhecido e
  registra o evento mesmo assim.
- **"Parada" êÂ© inferêÂªncia:** sem evento + processo vivo pode ser sêÂ³ um comando longo; a aba mostra o
  êÂºltimo comando para a pessoa julgar.

## 6. Ordem sugerida
T0 ê¢â â T1 ê¢â â T2 ê¢â â T3 (uma tarde, jêÂ¡ entrega o painel "Agora") êÂ· T4 êÂ· T5 êÂ· depois T6ê¢â¬âT8 conforme a necessidade.

## 7. VerificaêÂ§êÂ£o feita em 13/10/2026 (T1ê¢â¬âT3)

| CritêÂ©rio | Resultado |
|---|---|
| SessêÂ£o nova `claude -p` com um `ls` | registrou `SessionStart` (origem `headless`) ê¢â â `UserPromptSubmit` ê¢â â `PreToolUse`/`PostToolUse` `Bash ls` 54 ms ê¢â â `Stop` ê¢â â `SessionEnd` |
| Frase-sentinela no prompt, no comando, no erro e na saêÂ­da | **0** ocorrêÂªncias no banco |
| Banco travado por outro processo | hook saiu com 0 em 316 ms, evento descartado, sessêÂ£o seguiu |
| Velocidade do hook | 29 ms por evento (mêÂ©dia de 100) |
| `instalar.sh` + `desinstalar.sh` numa cêÂ³pia do `settings.json` | volta idêÂªntica ao original; os hooks antigos ficam intactos |
| `/api/*` (6 rotas) e pêÂ¡gina | 200; contagem bate com os processos (`ativas` + `abertas sem sinal` = processos `claude`) |
| Acesso por outro IP da rede | recusado (sêÂ³ escuta em 127.0.0.1) |
| Largura de celular (380 px) | sem rolagem horizontal |

Ajuste feito: a porta 8765 jêÂ¡ era usada por outro serviêÂ§o local; o painel ficou na **8766**.

## 8. T4 e T5 ê¢â¬â feito em 13/10/2026

- **T4:** o Claude Code manda um evento `api_request` por chamada ao modelo para `127.0.0.1:4318`. Guardamos
  sêÂ³ modelo, tokens (entrada, saêÂ­da, cache lido, cache criado), custo estimado e duraêÂ§êÂ£o, na tabela `consumo`.
  Erro de API vira evento `ApiError`. Pacote de outro programa que herdou as variêÂ¡veis `OTEL_*` êÂ© ignorado
  (sêÂ³ entra `service.name` com "claude"). MêÂ©tricas e traces: aceitos e descartados.
- **T5:** pelos arquivos de sessêÂ£o do Codex (nêÂ£o pela telemetria dele): sessêÂ£o, modelo, turnos, comandos
  (sêÂ³ o programa), duraêÂ§êÂ£o e tokens. `codex exec` some da aba Agora quando termina; o Codex interativo
  fica "esperando vocêÂª". O Codex nêÂ£o informa preêÂ§o, entêÂ£o o custo fica em branco.
- **Aba Consumo:** hoje, por modelo (7 dias), por projeto (hoje) e por dia (30 dias), separado por fonte.

| Teste | Resultado |
|---|---|
| sessêÂ£o nova `claude -p` | chegou `claude-opus-5-5`, 2 entrada / 4 saêÂ­da / 12.249 cache lido / 45.126 cache criado, US$ 0,36 estimado, 4,6 s |
| frase-sentinela no prompt com OTel ligado | **0** ocorrêÂªncias no banco |
| Codex | 784 arquivos de sessêÂ£o importados (2 dias), 3.273 registros de tokens |
| acesso por 192.168.2.99 | 200; por wi-fi (192.168.1.172) e Tailscale: recusado (nêÂ£o escuta lêÂ¡) |
| cliente fora de `TELEMETRIA_REDES` | 403 "fora da rede permitida" |

## 9. Custo de mêÂ¡quina (medido em 13/10/2026)

| Item | Custo |
|---|---|
| Hook | ~29 ms por evento; 2 eventos por ferramenta ê¢â â ~60 ms a mais por chamada de ferramenta. ~10ê¢â¬â20 mil eventos/dia ê¢â°Ë 5ê¢â¬â10 min de CPU por dia, espalhados |
| ServiêÂ§o do painel + coletores | ~13 MB de RAM; teto `MemoryMax=256M`, `CPUQuota=50%` |
| Leitura do Codex | 0,02 s a cada 15 s quando nêÂ£o hêÂ¡ novidade (~2 min de CPU por dia) |
| OTLP | desprezêÂ­vel (um POST pequeno a cada 5 s por sessêÂ£o ativa) |
| Disco | ~3ê¢â¬â5 MB/dia; com a limpeza de 30 dias, teto esperado ~100ê¢â¬â150 MB |
| Rede | nenhuma saêÂ­da da mêÂ¡quina; sêÂ³ localhost e a rede 192.168.2.x |

## 10. Ligar e desligar

```bash
cd ~/projetos/ccdashboard/telemetria
./telemetria.sh status     # o que estêÂ¡ ligado
./telemetria.sh desligar   # tira hooks e OTEL_* do settings.json, para painel e manutenêÂ§êÂ£o (banco fica)
./telemetria.sh ligar      # pêÂµe tudo de volta (backup do settings.json a cada vez)
```

SêÂ³ pausar o painel (hooks continuam gravando): `systemctl --user stop telemetria-painel.service`.
SêÂ³ o Codex ou sêÂ³ o OTLP: `TELEMETRIA_CODEX=0` / `TELEMETRIA_OTLP=0` no serviêÂ§o. Os hooks e o OTel valem
para sessêÂµes novas e para o prêÂ³ximo evento das abertas; o OTel sêÂ³ pega sessêÂµes abertas depois de ligado.

## 11. Rede: upload por sessêÂ£o (15/10/2026)

Na live de 15/10 (OBS no PC local, mesma Starlink deste servidor) o YouTube engasgou; TikTok/Insta/Face nêÂ£o.
Este servidor subia **16ê¢â¬â22 Mbps** pela Starlink (`enP7s7`): sessêÂµes Claude reenviando o contexto (com imagens)
para a API da Anthropic (`160.79.104.10`, `2607:6bc0::10`) e um upload do yt-pubx (Google). LatêÂªncia e perda
estavam normais, entêÂ£o o limite era a banda de subida. Pausar o yt-pubx e as sessêÂµes derrubou o upload para 0,2ê¢â¬â8 Mbps.

Ferramentas (ficam no projeto wifi, que êÂ© o hub de monitoramento): `~/projetos/wifi/MONITOR-REDE.md`

- `~/projetos/wifi/monitor-upload.sh [iface] [janela]`: sem sudo; IP remoto, serviêÂ§o, processo:PID,
  **pasta da sessêÂ£o e hêÂ¡ quanto tempo estêÂ¡ aberta**. SêÂ³ TCP do servidor; o resto aparece como "sem atribuiêÂ§êÂ£o".
- `~/.local/bin/bandwhich` v0.23.1 (ARM64): por pacote, inclui UDP. Liga com
  `sudo setcap cap_sys_ptrace,cap_dac_read_search,cap_net_raw,cap_net_admin+ep ~/.local/bin/bandwhich`
  e desliga com `sudo setcap -r ~/.local/bin/bandwhich`.

Nenhum dos dois roda sozinho. Ideia para depois (nêÂ£o aprovada): aba **Rede** no painel com upload por sessêÂ£o
(mesmo cruzamento PID ê¢â â cwd ê¢â â sessêÂ£o que o monitor faz) e um aviso quando a subida passar de um teto durante a live.

## 12. Aba VPS (16/10/2026)

- InventêÂ¡rio das 6 VPS (VPS1ê¢â¬âVPS6: 5 Hetzner + 1 Hostinger) em `painel/vps.json`, escrito êÂ  mêÂ£o; detalhe em `~/projetos/wifi/VPS.md`.
- `servidor.py` sobe `laco_vps()`: a cada `TELEMETRIA_VPS_MIN` (padrêÂ£o 10) min faz SSH `BatchMode` (chave, sem senha) e lêÂª disco, RAM e uptime; `/api/vps` sêÂ³ devolve o cache. Desligar: `TELEMETRIA_VPS=0`.
- Disco ê¢â°Â¥ 70% fica amarelo, ê¢â°Â¥ 85% vermelho; VPS sem resposta aparece "sem acesso" com o erro do SSH.
