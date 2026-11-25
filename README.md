# ê°Å¸âÅ  CC Dashboard ê¢â¬â painêÂ©is ao vivo no Claude

<div align="center">
  <img src="capa/logo.svg" alt="Arckhé AI" height="48" />
  <p><strong>Arckhé AI</strong></p>
</div>


[![CC Dashboard](guia/assets/banner.jpg)](https://linnoff88-dev.github.io/

**ê°Å¸â¡Â§ê°Å¸â¡Â· [PortuguêÂªs](README.md) êÂ· ê°Å¸â¡Âºê°Å¸â¡Â¸ [English](README.en.md) êÂ· ê°Å¸â¡Âªê°Å¸â¡Â¸ [EspaêÂ±ol](README.es.md)**

## O que êÂ©

O CC Dashboard êÂ© um kit em portuguêÂªs para usar os Dashboards do Claude, painêÂ©is que guardam as consultas e refazem as contas sempre que vocêÂª abre. Serve para donos de pequenos negêÂ³cios e para quem usa o Claude Code todo dia. Ele traz prompts prontos para montar painêÂ©is do negêÂ³cio (dinheiro, atrasos, vazamentos) e um plano para medir o prêÂ³prio Claude Code (custo estimado, ferramentas, falhas). Para usar, vocêÂª precisa de um plano pago do Claude e, nas partes com banco de dados, de uma conta gratuita no Supabase.

## ê°Å¸ââ Guia de uso

Guia completo (landing + passo a passo): **https://linnoff88-dev.github.io/

## Como funciona

Um Dashboard do Claude nêÂ£o grava nêÂºmeros: grava **consultas**. Toda vez que vocêÂª abre ou atualiza, ele roda as consultas de novo nos dados conectados. Clicando num nêÂºmero, vocêÂª vêÂª a consulta e a hora da êÂºltima atualizaêÂ§êÂ£o.

**NegêÂ³cio**

```
Apps (CRM, planilha, banco) ê¢â â Conector no Claude ê¢â â [Base central no Supabase] ê¢â â Dashboard ê¢â â CêÂ³pia agendada
```

- Uma pergunta que cabe num app sêÂ³: conector + prompt e pronto.
- Uma pergunta que cruza apps (horas êâ faturas, assinaturas êâ cobranêÂ§as): os registros precisam ser copiados para uma base central (Supabase = Postgres hospedado), que guarda histêÂ³rico e permite cruzar.

**Claude Code**

```
Claude Code (OpenTelemetry) ê¢â â Coletor no Supabase ê¢â â Tabelas brutas (7 dias) ê¢â â Resumo diêÂ¡rio (10 min) ê¢â â Dashboard
```

- O Claude Code envia a prêÂ³pria telemetria (modelo, tokens, custo estimado, ferramentas, duraêÂ§êÂ£o, falhas).
- O coletor guarda **sêÂ³ campos de uma lista permitida** e descarta texto de prompts e respostas, comandos completos, caminhos, entradas e saêÂ­das de ferramentas, texto de erro e e-mail. Do shell, guarda sêÂ³ o nome do programa.
- Um resumo diêÂ¡rio êÂ© recalculado a cada 10 minutos; os brutos sêÂ£o apagados apêÂ³s 7 dias.

## Como usar

| Passo | O que fazer | Arquivo |
|---|---|---|
| 1 | Ligar o conector do app em *Customize ê¢â â Connectors* | ê¢â¬â |
| 2 | Pedir o painel com pergunta, definiêÂ§êÂµes e "nêÂ£o altere dados" | `prompts/01-dashboard-negocio.md` |
| 3 | Conferir um nêÂºmero na origem antes de agir | `prompts/02-conferir-numero.md` |
| 4 | Cruzar apps numa base central no Supabase | `prompts/03-base-central.md` |
| 5 | Agendar a atualizaêÂ§êÂ£o da base | `prompts/05-agendar-atualizacao.md` |
| 6 | DiagnêÂ³stico do negêÂ³cio ê¢â â plano ê¢â â painel de acompanhamento | `prompts/06-diagnostico-negocio.md` |
| 7 | Montar o painel do Claude Code (banco, coletor, abas) | `prompts/04-observabilidade-claude-code.md` |
| 8 | Ligar a telemetria no `~/.claude/settings.json` e **reiniciar** o Claude Code | `config/claude-code-otel.example.json` |

Supabase sêÂ³ leitura e restrito a um projeto (conector personalizado):

```
https://mcp.supabase.com/mcp?project_ref=SEU_PROJECT_REF&read_only=true
```

Se nada chegar no coletor: `claude --debug-file /tmp/cc.log` e procure linhas `[3P telemetry]`. A tabela de sintomas (401, 404, 415, cartêÂµes vazios) estêÂ¡ no fim de `prompts/04-observabilidade-claude-code.md`.

## Telemetria de agentes (Frente C) ê¢â¬â local, sem nuvem

Painel na sua prêÂ³pria mêÂ¡quina com **todas** as sessêÂµes do Claude Code (terminais, bots, `claude -p`, subagentes) e do Codex, ao vivo: o que estêÂ¡ processando, o que espera vocêÂª, o que falhou, tokens e custo estimado. Sem API, sem conta nova.

![Aba Agora do painel de telemetria](guia/assets/telemetria-agora.jpg)

```bash
./help.sh                          # ajuda completa
./help.sh ligar                    # hooks + OpenTelemetry no ~/.claude/settings.json (com backup), painel e manutenêÂ§êÂ£o
telemetria/telemetria.sh status    # o que estêÂ¡ ligado agora
./help.sh desligar                 # tira tudo do settings.json e para os serviêÂ§os (o banco fica)
systemctl --user stop telemetria-painel.service   # sêÂ³ pausar o painel
```

- **Painel:** `http://127.0.0.1:8766/` e, na rede local, `http://<ip-da-maquina>:8766/` (endereêÂ§os em `TELEMETRIA_HOSTS`, redes liberadas em `TELEMETRIA_REDES`). Abas: Agora, Processamentos, Consumo, Falhas, Uso, HistêÂ³rico, SaêÂºde.
- **O que êÂ© guardado:** sêÂ³ nomes (projeto, ferramenta, skill, programa) e nêÂºmeros. Nunca prompt, resposta, comando completo, caminho de arquivo nem saêÂ­da de ferramenta. Banco em `~/.local/share/telemetria-agentes/` (30 dias de eventos + resumo diêÂ¡rio).
- **Custo de mêÂ¡quina (medido):** ~60 ms a mais por chamada de ferramenta, ~13 MB de RAM, 3ê¢â¬â5 MB de disco por dia (teto ~150 MB). Nada sai da mêÂ¡quina.
- **Limites:** tokens e custo sêÂ³ nas sessêÂµes do Claude Code abertas depois de ligar; custo êÂ© estimativa em preêÂ§o de API (o Codex nêÂ£o informa preêÂ§o); o painel nêÂ£o tem senha ê¢â¬â qualquer aparelho da rede liberada vêÂª. Testado no Linux.
- Detalhes, testes e fases: [`telemetria/PLANO.md`](telemetria/PLANO.md).

## PrêÂ©-requisitos

- Plano pago do Claude: Pro, Max, Team ou Enterprise (recurso em beta, conta no uso do plano).
- Conta Supabase grêÂ¡tis (500 MB; projeto grêÂ¡tis pausa apêÂ³s uma semana sem uso) ê¢â¬â para base central e Claude Code.
- Claude Code ê¢â¬â sêÂ³ para o painel de observabilidade.

## Limites

- **Mostra, nêÂ£o avisa.** Alerta exige uma rotina agendada separada.
- **Custo êÂ© estimativa** em preêÂ§o de API; em Pro/Max vale a assinatura.
- **Cada pessoa precisa da prêÂ³pria conexêÂ£o** ao projeto Supabase para ver os cartêÂµes.
- **Nomes ficam guardados** (skills, repositêÂ³rios, servidores MCP) ê¢â¬â pense em quem vêÂª.
- **SêÂ³ Claude Code**; o chat do app e o Cowork nêÂ£o entram.
- **Teste no seu sistema** (Linux, Windows, Mac) antes de confiar no fluxo.

## Estrutura

```
PLANO.md                               plano por fases (Frente A negêÂ³cio, Frente B Claude Code)
telemetria/PLANO.md                    Frente C: telemetria local de agentes (Claude Code + Codex); ligar/desligar: telemetria/telemetria.sh
help.sh                                ajuda + ligar/desligar a telemetria
prompts/01-dashboard-negocio.md        painel a partir de um app
prompts/02-conferir-numero.md          checklist para conferir um nêÂºmero
prompts/03-base-central.md             base central no Supabase
prompts/04-observabilidade-claude-code.md  prompt êÂºnico do painel do Claude Code + diagnêÂ³stico
prompts/05-agendar-atualizacao.md      cêÂ³pia agendada
prompts/06-diagnostico-negocio.md      diagnêÂ³stico sêÂ³ leitura ê¢â â plano ê¢â â acompanhamento
config/claude-code-otel.example.json   bloco env do Claude Code
guia/                                  pêÂ¡gina landing + guia (PT, EN, ES)
```

## Status

A **telemetria de agentes (Frente C)** jêÂ¡ estêÂ¡ implementada e testada no Linux (`telemetria/`). Plano + kit de prompts. O painel do Claude Code ainda nêÂ£o foi montado de ponta a ponta a partir deste kit; trate a primeira execuêÂ§êÂ£o como piloto e confira cada passo antes de dar OK ao Claude.

---

[ARCKHê AI](https://ARCKHê AI)

---

<div align="center">
  <img src="capa/logo.svg" alt="Arckhé AI" height="32" />
  <p><strong>Arckhé AI</strong></p>
</div>