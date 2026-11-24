# ðŸ“Š CC Dashboard â€” painÃ©is ao vivo no Claude

<div align="center">
  <img src="capa/logo.svg" alt="Arckhé AI" height="48" />
  <p><strong>Arckhé AI</strong></p>
</div>


[![CC Dashboard](guia/assets/banner.jpg)](https://linnoff88-dev.github.io/

**ðŸ‡§ðŸ‡· [PortuguÃªs](README.md) Â· ðŸ‡ºðŸ‡¸ [English](README.en.md) Â· ðŸ‡ªðŸ‡¸ [EspaÃ±ol](README.es.md)**

## O que Ã©

O CC Dashboard Ã© um kit em portuguÃªs para usar os Dashboards do Claude, painÃ©is que guardam as consultas e refazem as contas sempre que vocÃª abre. Serve para donos de pequenos negÃ³cios e para quem usa o Claude Code todo dia. Ele traz prompts prontos para montar painÃ©is do negÃ³cio (dinheiro, atrasos, vazamentos) e um plano para medir o prÃ³prio Claude Code (custo estimado, ferramentas, falhas). Para usar, vocÃª precisa de um plano pago do Claude e, nas partes com banco de dados, de uma conta gratuita no Supabase.

## ðŸ“– Guia de uso

Guia completo (landing + passo a passo): **https://linnoff88-dev.github.io/

## Como funciona

Um Dashboard do Claude nÃ£o grava nÃºmeros: grava **consultas**. Toda vez que vocÃª abre ou atualiza, ele roda as consultas de novo nos dados conectados. Clicando num nÃºmero, vocÃª vÃª a consulta e a hora da Ãºltima atualizaÃ§Ã£o.

**NegÃ³cio**

```
Apps (CRM, planilha, banco) â†’ Conector no Claude â†’ [Base central no Supabase] â†’ Dashboard â†’ CÃ³pia agendada
```

- Uma pergunta que cabe num app sÃ³: conector + prompt e pronto.
- Uma pergunta que cruza apps (horas Ã— faturas, assinaturas Ã— cobranÃ§as): os registros precisam ser copiados para uma base central (Supabase = Postgres hospedado), que guarda histÃ³rico e permite cruzar.

**Claude Code**

```
Claude Code (OpenTelemetry) â†’ Coletor no Supabase â†’ Tabelas brutas (7 dias) â†’ Resumo diÃ¡rio (10 min) â†’ Dashboard
```

- O Claude Code envia a prÃ³pria telemetria (modelo, tokens, custo estimado, ferramentas, duraÃ§Ã£o, falhas).
- O coletor guarda **sÃ³ campos de uma lista permitida** e descarta texto de prompts e respostas, comandos completos, caminhos, entradas e saÃ­das de ferramentas, texto de erro e e-mail. Do shell, guarda sÃ³ o nome do programa.
- Um resumo diÃ¡rio Ã© recalculado a cada 10 minutos; os brutos sÃ£o apagados apÃ³s 7 dias.

## Como usar

| Passo | O que fazer | Arquivo |
|---|---|---|
| 1 | Ligar o conector do app em *Customize â†’ Connectors* | â€” |
| 2 | Pedir o painel com pergunta, definiÃ§Ãµes e "nÃ£o altere dados" | `prompts/01-dashboard-negocio.md` |
| 3 | Conferir um nÃºmero na origem antes de agir | `prompts/02-conferir-numero.md` |
| 4 | Cruzar apps numa base central no Supabase | `prompts/03-base-central.md` |
| 5 | Agendar a atualizaÃ§Ã£o da base | `prompts/05-agendar-atualizacao.md` |
| 6 | DiagnÃ³stico do negÃ³cio â†’ plano â†’ painel de acompanhamento | `prompts/06-diagnostico-negocio.md` |
| 7 | Montar o painel do Claude Code (banco, coletor, abas) | `prompts/04-observabilidade-claude-code.md` |
| 8 | Ligar a telemetria no `~/.claude/settings.json` e **reiniciar** o Claude Code | `config/claude-code-otel.example.json` |

Supabase sÃ³ leitura e restrito a um projeto (conector personalizado):

```
https://mcp.supabase.com/mcp?project_ref=SEU_PROJECT_REF&read_only=true
```

Se nada chegar no coletor: `claude --debug-file /tmp/cc.log` e procure linhas `[3P telemetry]`. A tabela de sintomas (401, 404, 415, cartÃµes vazios) estÃ¡ no fim de `prompts/04-observabilidade-claude-code.md`.

## Telemetria de agentes (Frente C) â€” local, sem nuvem

Painel na sua prÃ³pria mÃ¡quina com **todas** as sessÃµes do Claude Code (terminais, bots, `claude -p`, subagentes) e do Codex, ao vivo: o que estÃ¡ processando, o que espera vocÃª, o que falhou, tokens e custo estimado. Sem API, sem conta nova.

![Aba Agora do painel de telemetria](guia/assets/telemetria-agora.jpg)

```bash
./help.sh                          # ajuda completa
./help.sh ligar                    # hooks + OpenTelemetry no ~/.claude/settings.json (com backup), painel e manutenÃ§Ã£o
telemetria/telemetria.sh status    # o que estÃ¡ ligado agora
./help.sh desligar                 # tira tudo do settings.json e para os serviÃ§os (o banco fica)
systemctl --user stop telemetria-painel.service   # sÃ³ pausar o painel
```

- **Painel:** `http://127.0.0.1:8766/` e, na rede local, `http://<ip-da-maquina>:8766/` (endereÃ§os em `TELEMETRIA_HOSTS`, redes liberadas em `TELEMETRIA_REDES`). Abas: Agora, Processamentos, Consumo, Falhas, Uso, HistÃ³rico, SaÃºde.
- **O que Ã© guardado:** sÃ³ nomes (projeto, ferramenta, skill, programa) e nÃºmeros. Nunca prompt, resposta, comando completo, caminho de arquivo nem saÃ­da de ferramenta. Banco em `~/.local/share/telemetria-agentes/` (30 dias de eventos + resumo diÃ¡rio).
- **Custo de mÃ¡quina (medido):** ~60 ms a mais por chamada de ferramenta, ~13 MB de RAM, 3â€“5 MB de disco por dia (teto ~150 MB). Nada sai da mÃ¡quina.
- **Limites:** tokens e custo sÃ³ nas sessÃµes do Claude Code abertas depois de ligar; custo Ã© estimativa em preÃ§o de API (o Codex nÃ£o informa preÃ§o); o painel nÃ£o tem senha â€” qualquer aparelho da rede liberada vÃª. Testado no Linux.
- Detalhes, testes e fases: [`telemetria/PLANO.md`](telemetria/PLANO.md).

## PrÃ©-requisitos

- Plano pago do Claude: Pro, Max, Team ou Enterprise (recurso em beta, conta no uso do plano).
- Conta Supabase grÃ¡tis (500 MB; projeto grÃ¡tis pausa apÃ³s uma semana sem uso) â€” para base central e Claude Code.
- Claude Code â€” sÃ³ para o painel de observabilidade.

## Limites

- **Mostra, nÃ£o avisa.** Alerta exige uma rotina agendada separada.
- **Custo Ã© estimativa** em preÃ§o de API; em Pro/Max vale a assinatura.
- **Cada pessoa precisa da prÃ³pria conexÃ£o** ao projeto Supabase para ver os cartÃµes.
- **Nomes ficam guardados** (skills, repositÃ³rios, servidores MCP) â€” pense em quem vÃª.
- **SÃ³ Claude Code**; o chat do app e o Cowork nÃ£o entram.
- **Teste no seu sistema** (Linux, Windows, Mac) antes de confiar no fluxo.

## Estrutura

```
PLANO.md                               plano por fases (Frente A negÃ³cio, Frente B Claude Code)
telemetria/PLANO.md                    Frente C: telemetria local de agentes (Claude Code + Codex); ligar/desligar: telemetria/telemetria.sh
help.sh                                ajuda + ligar/desligar a telemetria
prompts/01-dashboard-negocio.md        painel a partir de um app
prompts/02-conferir-numero.md          checklist para conferir um nÃºmero
prompts/03-base-central.md             base central no Supabase
prompts/04-observabilidade-claude-code.md  prompt Ãºnico do painel do Claude Code + diagnÃ³stico
prompts/05-agendar-atualizacao.md      cÃ³pia agendada
prompts/06-diagnostico-negocio.md      diagnÃ³stico sÃ³ leitura â†’ plano â†’ acompanhamento
config/claude-code-otel.example.json   bloco env do Claude Code
guia/                                  pÃ¡gina landing + guia (PT, EN, ES)
```

## Status

A **telemetria de agentes (Frente C)** jÃ¡ estÃ¡ implementada e testada no Linux (`telemetria/`). Plano + kit de prompts. O painel do Claude Code ainda nÃ£o foi montado de ponta a ponta a partir deste kit; trate a primeira execuÃ§Ã£o como piloto e confira cada passo antes de dar OK ao Claude.

---

[ARCKHÉ AI](https://ARCKHÉ AI)

---

<div align="center">
  <img src="capa/logo.svg" alt="Arckhé AI" height="32" />
  <p><strong>Arckhé AI</strong></p>
</div>