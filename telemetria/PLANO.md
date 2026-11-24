# Plano â€” Telemetria de agentes (Frente C)

Painel **local** que mostra, ao vivo, tudo o que os agentes estÃ£o fazendo na mÃ¡quina: cada sessÃ£o do
Claude Code (terminais, bots, `claude -p`, loops longos, subagentes) e do Codex CLI.

Status (13/10/2026): **T0â€“T5 prontas e instaladas** nesta mÃ¡quina. T6â€“T8 ainda sÃ£o plano.

| PeÃ§a | Arquivo |
|---|---|
| Coletor por hook | `hook/registra.py` |
| Instalar / remover | `instalar.sh` / `desinstalar.sh` (via `ajusta_hooks.py`, com backup do `settings.json`) |
| ManutenÃ§Ã£o (resumo, limpeza 30 dias, WAL) | `manutencao.py` + `systemd/telemetria-manutencao.timer` (de hora em hora) |
| Painel | `painel/servidor.py` + `painel/painel.html` â†’ **http://127.0.0.1:8766/** e **http://192.168.2.99:8766/** (serviÃ§o `telemetria-painel`; sÃ³ atende 127.0.0.0/8 e 192.168.2.0/24) |
| T4 Â· OpenTelemetry | `painel/coletores.py` â€” receptor OTLP em 127.0.0.1:4318; variÃ¡veis `OTEL_*` no `env` do `settings.json` |
| T5 Â· Codex | `painel/coletores.py` â€” lÃª `~/.codex/sessions/**/*.jsonl` a cada 15 s (Ãºltimos 2 dias, posiÃ§Ã£o guardada por arquivo) |
| Ligar / desligar | `./telemetria.sh ligar` Â· `./telemetria.sh desligar` Â· `./telemetria.sh status` |
| Banco | `~/.local/share/telemetria-agentes/eventos.db` (fora do repo) |

**Resultado da T0:** ~3â€“5 mil chamadas de ferramenta por dia (pico 5,2 mil em 10/10) â†’ ~10 mil eventos/dia,
cerca de 2 MB/dia, ~60 MB em 30 dias. O Claude Code 2.1.296 entrega os eventos `SessionStart`, `SessionEnd`,
`UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `PermissionRequest`, `Notification`,
`Stop`, `SubagentStart`, `SubagentStop`, `PreCompact` e `PostCompact`, com `agent_id`/`agent_type` nos de subagente.
**SessÃµes jÃ¡ abertas passam a registrar no evento seguinte** (o Claude Code relÃª os hooks); as que estÃ£o
paradas aparecem no contador "abertas sem sinal ainda" atÃ© fazerem algo.

**Regra de ouro:** tudo fica na mÃ¡quina. Sem nuvem, sem API, sem conta nova. Guarda **nomes e nÃºmeros**,
nunca o texto de prompt, resposta, comando completo ou conteÃºdo de arquivo.

---

## 1. Perguntas que o painel responde

| Pergunta | De onde sai |
|---|---|
| Quais sessÃµes estÃ£o ativas agora, em que pasta, hÃ¡ quanto tempo? | hooks `SessionStart`/`SessionEnd` + Ãºltimo evento |
| Qual estÃ¡ **esperando resposta minha**? | hook `Notification` / `Stop` sem novo `UserPromptSubmit` |
| Qual estÃ¡ **parada** (sem evento hÃ¡ N min, processo vivo)? | Ãºltimo evento + `pgrep` |
| O que cada uma estÃ¡ fazendo agora? | Ãºltimo `PreToolUse` (ferramenta, skill, MCP) |
| Quais processamentos longos estÃ£o rodando e hÃ¡ quanto tempo? | `PreToolUse` Bash sem `PostToolUse` correspondente |
| O que falhou hoje, por ferramenta / skill / MCP? | `PostToolUse` com erro |
| Quais skills, MCPs e subagentes eu uso de verdade? | contagens do histÃ³rico |
| Quanto cada sessÃ£o consumiu (tokens, custo estimado em preÃ§o de API)? | OpenTelemetry (Fase T4) |

## 2. Arquitetura

```
Claude Code (todas as sessÃµes) â”€â”¬â”€ hooks â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
                                â””â”€ OpenTelemetry (T4) â”€â”¤   (sÃ³ localhost)
Codex CLI â”€â”€ sessÃµes no disco (~/.codex/sessions) â”€â”€â”€â”€â”€â”¤
                                                       â–¼
                         ~/.local/share/telemetria-agentes/eventos.db  (SQLite, modo WAL)
                                                       â”‚
                                     servidor 127.0.0.1 + 192.168.2.99 :8766
                                                       â”‚
                                     painel HTML (atualiza a cada 5 s)
```

Por que funciona para **todas** as sessÃµes: toda sessÃ£o do Claude Code lÃª `~/.claude/settings.json` ao
abrir. Um hook registrado ali vale para qualquer terminal, bot, `claude -p` e subagente.

## 3. Fases

### T0 â€” Medir antes de mexer (sem instalar nada)
- Contar sessÃµes ativas (`pgrep`), eventos por hora no histÃ³rico (`~/.claude/projects/*.jsonl`) e
  estimar o volume de eventos por dia â†’ dimensiona o banco e a retenÃ§Ã£o.
- Listar os hooks jÃ¡ existentes em `settings.json` para o novo **somar**, nÃ£o substituir.
- Conferir quais eventos de hook a versÃ£o instalada entrega e o formato do JSON de entrada.

### T1 â€” Coletor por hook
- Script `telemetria/hook/registra.py` (Python puro, sem dependÃªncia):
  lÃª o JSON do hook na entrada padrÃ£o â†’ filtra pela **lista de campos permitidos** â†’ grava uma linha
  no SQLite â†’ sai com cÃ³digo 0 **sempre** (nunca bloqueia nem altera a sessÃ£o).
- Eventos: `SessionStart`, `SessionEnd`, `UserPromptSubmit`, `PreToolUse`, `PostToolUse`,
  `Notification`, `Stop`, `SubagentStop`, `PreCompact`.
- Tetos: `timeout` 2 s no hook; gravaÃ§Ã£o com `busy_timeout` curto; se o banco estiver travado,
  descarta o evento (perder uma linha Ã© melhor que travar uma sessÃ£o).
- InstalaÃ§Ã£o: script `telemetria/instalar.sh` que **acrescenta** os hooks em `settings.json`
  (com backup datado) e `desinstalar.sh` que remove sÃ³ os dele.

**Campos permitidos** (o resto Ã© descartado):

| Campo | Exemplo |
|---|---|
| hora, evento | `2026-10-13T10:02:11`, `PostToolUse` |
| sessÃ£o, sessÃ£o-mÃ£e | id da sessÃ£o; subagente aponta para a mÃ£e |
| pasta do projeto | sÃ³ o nome da pasta (`ccdashboard`), nÃ£o o caminho |
| origem | `terminal`, `headless`, `bot`, `subagente` (pela variÃ¡vel de ambiente / pai do processo) |
| ferramenta | `Bash`, `Edit`, `Skill`, `mcp__<servidor>__<ferramenta>` |
| programa do Bash | sÃ³ a primeira palavra (`ffmpeg`, `git`, `python3`) |
| skill / tipo de subagente | nome |
| duraÃ§Ã£o, sucesso/falha | ms; `ok`/`erro` |
| tipo do erro | categoria curta (`timeout`, `exitâ‰ 0`, `negado`), sem o texto |

**Nunca grava:** prompt, resposta, comando completo, caminho de arquivo, entrada/saÃ­da de ferramenta,
texto de erro, e-mail, tokens/chaves.

### T2 â€” Banco local
- Tabelas: `eventos` (bruto, 30 dias), `sessoes` (uma linha por sessÃ£o, estado atual),
  `resumo_diario` (permanente).
- Limpeza diÃ¡ria dos brutos com mais de 30 dias (timer `systemd --user`).
- Tamanho esperado: medido na T0; alvo < 200 MB.

### T3 â€” Painel local
- `telemetria/painel/servidor.py` (Python, sÃ³ `http.server` + `sqlite3`) escutando **sÃ³ em
  127.0.0.1**, serviÃ§o `systemd --user` com `MemoryMax`.
- PÃ¡gina Ãºnica, padrÃ£o ARCKHÉ (escuro/Ã¢mbar), atualizaÃ§Ã£o a cada 5 s:

| Aba | Mostra |
|---|---|
| **Agora** | sessÃµes ativas: pasta, origem, tempo, ferramenta atual; selo **esperando vocÃª** / **parada hÃ¡ N min** / **processando** |
| **Processamentos** | comandos longos em andamento e concluÃ­dos hoje (programa, duraÃ§Ã£o, resultado) |
| **Falhas** | erros do dia por ferramenta, skill e MCP; taxa de falha; ligaÃ§Ã£o com `FALHAS.md` |
| **Uso** | skills, MCPs e subagentes mais usados; o que nunca Ã© usado |
| **HistÃ³rico** | sessÃµes e ferramentas por dia (30 dias) |
| **SaÃºde** | Ãºltimo evento recebido, tamanho do banco, hooks instalados |

### T4 â€” OpenTelemetry local (tokens e custo)
- Receptor OTLP em `127.0.0.1:4318` dentro do mesmo servidor, gravando no mesmo SQLite.
- Bloco `env` em `~/.claude/settings.json` apontando para `http://127.0.0.1:4318`
  (mesmas variÃ¡veis de `config/claude-code-otel.example.json`, sem chave e sem nuvem).
- Vale sÃ³ para sessÃµes abertas **depois** da mudanÃ§a.
- Custo exibido = **estimativa em preÃ§o de API**; na assinatura quem manda Ã© o plano.

### T5 â€” Codex
- Leitor das sessÃµes do Codex em `~/.codex/sessions/` (arquivos JSONL), mesma lista de campos.
- Verificar se a versÃ£o instalada do Codex CLI exporta OpenTelemetry; se exportar, usar o mesmo
  receptor da T4.

### T6 â€” HistÃ³rico antigo (opcional)
- Importar `~/.claude/projects/*.jsonl` (sÃ³ nÃºmeros) para o painel jÃ¡ nascer com meses de dados.

### T7 â€” Avisos (opcional)
- Rotina a cada 5 min: sessÃ£o esperando resposta hÃ¡ mais de 15 min, ferramenta com taxa de falha
  acima de 30%, processamento rodando hÃ¡ mais de 1 h â†’ mensagem no bot.

### T8 â€” Ponte para o Claude Dashboards (opcional)
- Mesmo esquema no Supabase (Frente B do `PLANO.md`) quando quiser ver o painel no app do Claude ou de
  outra mÃ¡quina. Exige autorizaÃ§Ã£o explÃ­cita de uso do Supabase.

## 4. CritÃ©rios de pronto (cada fase)

| Fase | Comando â†’ resultado esperado |
|---|---|
| T1 | abrir sessÃ£o nova, rodar 1 `ls` â†’ `sqlite3 eventos.db "select evento,ferramenta from eventos order by id desc limit 3"` mostra `PostToolUse Bash` |
| T1 | hook com banco travado (`flock` no arquivo) â†’ a sessÃ£o segue normal, evento descartado, sem atraso > 2 s |
| T1 | `grep` no banco por um trecho de prompt digitado no teste â†’ **zero** ocorrÃªncias |
| T1 | `desinstalar.sh` â†’ `settings.json` igual ao backup, hooks antigos intactos |
| T3 | `curl -s 127.0.0.1:8766/api/agora` lista as sessÃµes ativas; contagem bate com `pgrep` (Â±1) |
| T3 | o painel nÃ£o abre por outro IP da rede |
| T4 | sessÃ£o nova â†’ aparecem tokens e modelo no painel em atÃ© 60 s |
| T5 | uma execuÃ§Ã£o `codex exec` aparece na aba **Agora** / **HistÃ³rico** |

## 5. Riscos e limites
- **Hook em todo evento:** precisa ser rÃ¡pido (alvo < 50 ms). Medir na T1 com 100 eventos.
- **Hooks globais valem para todos os projetos:** instalaÃ§Ã£o sÃ³ com "sim" do dono; desinstalador testado.
- **Nomes de skills, MCPs e pastas ficam no banco:** Ã© local, mas nÃ£o compartilhar o arquivo.
- **Eventos de hook mudam entre versÃµes do Claude Code:** o coletor ignora campo desconhecido e
  registra o evento mesmo assim.
- **"Parada" Ã© inferÃªncia:** sem evento + processo vivo pode ser sÃ³ um comando longo; a aba mostra o
  Ãºltimo comando para a pessoa julgar.

## 6. Ordem sugerida
T0 â†’ T1 â†’ T2 â†’ T3 (uma tarde, jÃ¡ entrega o painel "Agora") Â· T4 Â· T5 Â· depois T6â€“T8 conforme a necessidade.

## 7. VerificaÃ§Ã£o feita em 13/10/2026 (T1â€“T3)

| CritÃ©rio | Resultado |
|---|---|
| SessÃ£o nova `claude -p` com um `ls` | registrou `SessionStart` (origem `headless`) â†’ `UserPromptSubmit` â†’ `PreToolUse`/`PostToolUse` `Bash ls` 54 ms â†’ `Stop` â†’ `SessionEnd` |
| Frase-sentinela no prompt, no comando, no erro e na saÃ­da | **0** ocorrÃªncias no banco |
| Banco travado por outro processo | hook saiu com 0 em 316 ms, evento descartado, sessÃ£o seguiu |
| Velocidade do hook | 29 ms por evento (mÃ©dia de 100) |
| `instalar.sh` + `desinstalar.sh` numa cÃ³pia do `settings.json` | volta idÃªntica ao original; os hooks antigos ficam intactos |
| `/api/*` (6 rotas) e pÃ¡gina | 200; contagem bate com os processos (`ativas` + `abertas sem sinal` = processos `claude`) |
| Acesso por outro IP da rede | recusado (sÃ³ escuta em 127.0.0.1) |
| Largura de celular (380 px) | sem rolagem horizontal |

Ajuste feito: a porta 8765 jÃ¡ era usada por outro serviÃ§o local; o painel ficou na **8766**.

## 8. T4 e T5 â€” feito em 13/10/2026

- **T4:** o Claude Code manda um evento `api_request` por chamada ao modelo para `127.0.0.1:4318`. Guardamos
  sÃ³ modelo, tokens (entrada, saÃ­da, cache lido, cache criado), custo estimado e duraÃ§Ã£o, na tabela `consumo`.
  Erro de API vira evento `ApiError`. Pacote de outro programa que herdou as variÃ¡veis `OTEL_*` Ã© ignorado
  (sÃ³ entra `service.name` com "claude"). MÃ©tricas e traces: aceitos e descartados.
- **T5:** pelos arquivos de sessÃ£o do Codex (nÃ£o pela telemetria dele): sessÃ£o, modelo, turnos, comandos
  (sÃ³ o programa), duraÃ§Ã£o e tokens. `codex exec` some da aba Agora quando termina; o Codex interativo
  fica "esperando vocÃª". O Codex nÃ£o informa preÃ§o, entÃ£o o custo fica em branco.
- **Aba Consumo:** hoje, por modelo (7 dias), por projeto (hoje) e por dia (30 dias), separado por fonte.

| Teste | Resultado |
|---|---|
| sessÃ£o nova `claude -p` | chegou `claude-opus-5-5`, 2 entrada / 4 saÃ­da / 12.249 cache lido / 45.126 cache criado, US$ 0,36 estimado, 4,6 s |
| frase-sentinela no prompt com OTel ligado | **0** ocorrÃªncias no banco |
| Codex | 784 arquivos de sessÃ£o importados (2 dias), 3.273 registros de tokens |
| acesso por 192.168.2.99 | 200; por wi-fi (192.168.1.172) e Tailscale: recusado (nÃ£o escuta lÃ¡) |
| cliente fora de `TELEMETRIA_REDES` | 403 "fora da rede permitida" |

## 9. Custo de mÃ¡quina (medido em 13/10/2026)

| Item | Custo |
|---|---|
| Hook | ~29 ms por evento; 2 eventos por ferramenta â†’ ~60 ms a mais por chamada de ferramenta. ~10â€“20 mil eventos/dia â‰ˆ 5â€“10 min de CPU por dia, espalhados |
| ServiÃ§o do painel + coletores | ~13 MB de RAM; teto `MemoryMax=256M`, `CPUQuota=50%` |
| Leitura do Codex | 0,02 s a cada 15 s quando nÃ£o hÃ¡ novidade (~2 min de CPU por dia) |
| OTLP | desprezÃ­vel (um POST pequeno a cada 5 s por sessÃ£o ativa) |
| Disco | ~3â€“5 MB/dia; com a limpeza de 30 dias, teto esperado ~100â€“150 MB |
| Rede | nenhuma saÃ­da da mÃ¡quina; sÃ³ localhost e a rede 192.168.2.x |

## 10. Ligar e desligar

```bash
cd ~/projetos/ccdashboard/telemetria
./telemetria.sh status     # o que estÃ¡ ligado
./telemetria.sh desligar   # tira hooks e OTEL_* do settings.json, para painel e manutenÃ§Ã£o (banco fica)
./telemetria.sh ligar      # pÃµe tudo de volta (backup do settings.json a cada vez)
```

SÃ³ pausar o painel (hooks continuam gravando): `systemctl --user stop telemetria-painel.service`.
SÃ³ o Codex ou sÃ³ o OTLP: `TELEMETRIA_CODEX=0` / `TELEMETRIA_OTLP=0` no serviÃ§o. Os hooks e o OTel valem
para sessÃµes novas e para o prÃ³ximo evento das abertas; o OTel sÃ³ pega sessÃµes abertas depois de ligado.

## 11. Rede: upload por sessÃ£o (15/10/2026)

Na live de 15/10 (OBS no PC local, mesma Starlink deste servidor) o YouTube engasgou; TikTok/Insta/Face nÃ£o.
Este servidor subia **16â€“22 Mbps** pela Starlink (`enP7s7`): sessÃµes Claude reenviando o contexto (com imagens)
para a API da Anthropic (`160.79.104.10`, `2607:6bc0::10`) e um upload do yt-pubx (Google). LatÃªncia e perda
estavam normais, entÃ£o o limite era a banda de subida. Pausar o yt-pubx e as sessÃµes derrubou o upload para 0,2â€“8 Mbps.

Ferramentas (ficam no projeto wifi, que Ã© o hub de monitoramento): `~/projetos/wifi/MONITOR-REDE.md`

- `~/projetos/wifi/monitor-upload.sh [iface] [janela]`: sem sudo; IP remoto, serviÃ§o, processo:PID,
  **pasta da sessÃ£o e hÃ¡ quanto tempo estÃ¡ aberta**. SÃ³ TCP do servidor; o resto aparece como "sem atribuiÃ§Ã£o".
- `~/.local/bin/bandwhich` v0.23.1 (ARM64): por pacote, inclui UDP. Liga com
  `sudo setcap cap_sys_ptrace,cap_dac_read_search,cap_net_raw,cap_net_admin+ep ~/.local/bin/bandwhich`
  e desliga com `sudo setcap -r ~/.local/bin/bandwhich`.

Nenhum dos dois roda sozinho. Ideia para depois (nÃ£o aprovada): aba **Rede** no painel com upload por sessÃ£o
(mesmo cruzamento PID â†’ cwd â†’ sessÃ£o que o monitor faz) e um aviso quando a subida passar de um teto durante a live.

## 12. Aba VPS (16/10/2026)

- InventÃ¡rio das 6 VPS (VPS1â€“VPS6: 5 Hetzner + 1 Hostinger) em `painel/vps.json`, escrito Ã  mÃ£o; detalhe em `~/projetos/wifi/VPS.md`.
- `servidor.py` sobe `laco_vps()`: a cada `TELEMETRIA_VPS_MIN` (padrÃ£o 10) min faz SSH `BatchMode` (chave, sem senha) e lÃª disco, RAM e uptime; `/api/vps` sÃ³ devolve o cache. Desligar: `TELEMETRIA_VPS=0`.
- Disco â‰¥ 70% fica amarelo, â‰¥ 85% vermelho; VPS sem resposta aparece "sem acesso" com o erro do SSH.
