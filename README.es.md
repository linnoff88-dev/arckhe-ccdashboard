# ðŸ“Š CC Dashboard â€” paneles en vivo en Claude

[![CC Dashboard](guia/assets/banner-es.jpg)](https://linnoff88-dev.github.io/

**ðŸ‡§ðŸ‡· [PortuguÃªs](README.md) Â· ðŸ‡ºðŸ‡¸ [English](README.en.md) Â· ðŸ‡ªðŸ‡¸ [EspaÃ±ol](README.es.md)**

## QuÃ© es

CC Dashboard es un kit en espaÃ±ol para usar los Dashboards de Claude, paneles que guardan las consultas y rehacen las cuentas cada vez que los abres. Sirve para dueÃ±os de pequeÃ±os negocios y para quienes usan Claude Code todos los dÃ­as. Trae prompts listos para armar paneles del negocio (dinero, atrasos, fugas) y un plan para medir tu propio Claude Code (costo estimado, herramientas, fallas). Para usarlo necesitas un plan de pago de Claude y, en las partes con base de datos, una cuenta gratuita en Supabase.

## ðŸ“– GuÃ­a de uso

GuÃ­a completa (landing + paso a paso): **https://linnoff88-dev.github.io/

> Nota: los archivos de prompts (`prompts/*.md`) y el `PLANO.md` estÃ¡n en portuguÃ©s.

## CÃ³mo funciona

Un Dashboard de Claude no guarda nÃºmeros: guarda **consultas**. Cada vez que lo abres o lo actualizas, vuelve a ejecutar las consultas sobre los datos conectados. Al hacer clic en un nÃºmero, ves la consulta y la hora de la Ãºltima actualizaciÃ³n.

**Negocio**

```
Apps (CRM, hoja de cÃ¡lculo, base de datos) â†’ Conector en Claude â†’ [Base central en Supabase] â†’ Dashboard â†’ Copia programada
```

- Una pregunta que cabe en una sola app: conector + prompt y listo.
- Una pregunta que cruza apps (horas Ã— facturas, suscripciones Ã— cobros): los registros deben copiarse a una base central (Supabase = Postgres alojado), que guarda el historial y permite cruzar.

**Claude Code**

```
Claude Code (OpenTelemetry) â†’ Recolector en Supabase â†’ Tablas en bruto (7 dÃ­as) â†’ Resumen diario (10 min) â†’ Dashboard
```

- Claude Code envÃ­a su propia telemetrÃ­a (modelo, tokens, costo estimado, herramientas, duraciÃ³n, fallas).
- El recolector guarda **solo campos de una lista permitida** y descarta el texto de prompts y respuestas, comandos completos, rutas, entradas y salidas de herramientas, texto de error y correo electrÃ³nico. Del shell, guarda solo el nombre del programa.
- Un resumen diario se recalcula cada 10 minutos; los datos en bruto se borran a los 7 dÃ­as.

## CÃ³mo usar

| Paso | QuÃ© hacer | Archivo |
|---|---|---|
| 1 | Activar el conector de la app en *Customize â†’ Connectors* | â€” |
| 2 | Pedir el panel con pregunta, definiciones y "no modifiques datos" | `prompts/01-dashboard-negocio.md` |
| 3 | Verificar un nÃºmero en el origen antes de actuar | `prompts/02-conferir-numero.md` |
| 4 | Cruzar apps en una base central en Supabase | `prompts/03-base-central.md` |
| 5 | Programar la actualizaciÃ³n de la base | `prompts/05-agendar-atualizacao.md` |
| 6 | DiagnÃ³stico del negocio â†’ plan â†’ panel de seguimiento | `prompts/06-diagnostico-negocio.md` |
| 7 | Armar el panel de Claude Code (base de datos, recolector, pestaÃ±as) | `prompts/04-observabilidade-claude-code.md` |
| 8 | Activar la telemetrÃ­a en `~/.claude/settings.json` y **reiniciar** Claude Code | `config/claude-code-otel.example.json` |

Supabase de solo lectura y restringido a un proyecto (conector personalizado):

```
https://mcp.supabase.com/mcp?project_ref=SEU_PROJECT_REF&read_only=true
```

Si no llega nada al recolector: `claude --debug-file /tmp/cc.log` y busca lÃ­neas `[3P telemetry]`. La tabla de sÃ­ntomas (401, 404, 415, tarjetas vacÃ­as) estÃ¡ al final de `prompts/04-observabilidade-claude-code.md`.

## TelemetrÃ­a de agentes (Frente C) â€” local, sin nube

Un panel en tu propia mÃ¡quina con **todas** las sesiones de Claude Code (terminales, bots, `claude -p`, subagentes) y de Codex, en vivo: quÃ© se estÃ¡ procesando, quÃ© te espera, quÃ© fallÃ³, tokens y costo estimado. Sin API, sin cuenta nueva.

![PestaÃ±a Ahora del panel de telemetrÃ­a (interfaz en portuguÃ©s)](guia/assets/telemetria-agora.jpg)

```bash
./help.sh                          # ayuda completa
./help.sh ligar                    # hooks + OpenTelemetry en ~/.claude/settings.json (con copia), panel y mantenimiento
telemetria/telemetria.sh status    # quÃ© estÃ¡ activo ahora
./help.sh desligar                 # quita todo del settings.json y detiene los servicios (la base se conserva)
systemctl --user stop telemetria-painel.service   # solo pausar el panel
```

- **Panel:** `http://127.0.0.1:8766/` y, en la red local, `http://<ip-de-la-maquina>:8766/` (direcciones en `TELEMETRIA_HOSTS`, redes permitidas en `TELEMETRIA_REDES`). PestaÃ±as: Ahora, Procesos, Consumo, Fallas, Uso, Historial, Salud.
- **QuÃ© se guarda:** solo nombres (proyecto, herramienta, skill, programa) y nÃºmeros. Nunca prompts, respuestas, comandos completos, rutas de archivo ni salida de herramientas. Base en `~/.local/share/telemetria-agentes/` (30 dÃ­as de eventos + resumen diario).
- **Costo de mÃ¡quina (medido):** ~60 ms mÃ¡s por llamada de herramienta, ~13 MB de RAM, 3â€“5 MB de disco por dÃ­a (tope ~150 MB). Nada sale de la mÃ¡quina.
- **LÃ­mites:** tokens y costo solo en sesiones de Claude Code abiertas despuÃ©s de activarla; el costo es una estimaciÃ³n a precio de API (Codex no informa precio); el panel no tiene contraseÃ±a â€” cualquier dispositivo de la red permitida lo ve. Probado en Linux.
- Detalles, pruebas y fases: [`telemetria/PLANO.md`](telemetria/PLANO.md) (en portuguÃ©s).

## Requisitos previos

- Plan de pago de Claude: Pro, Max, Team o Enterprise (funciÃ³n en beta, cuenta en el uso del plan).
- Cuenta de Supabase gratuita (500 MB; el proyecto gratuito se pausa tras una semana sin uso) â€” para la base central y Claude Code.
- Claude Code â€” solo para el panel de observabilidad.

## LÃ­mites

- **Muestra, no avisa.** Una alerta exige una rutina programada aparte.
- **El costo es una estimaciÃ³n** a precio de API; en Pro/Max vale la suscripciÃ³n.
- **Cada persona necesita su propia conexiÃ³n** al proyecto de Supabase para ver las tarjetas.
- **Los nombres quedan guardados** (skills, repositorios, servidores MCP) â€” piensa en quiÃ©n lo ve.
- **Solo Claude Code**; el chat de la app y Cowork no entran.
- **PruÃ©balo en tu sistema** (Linux, Windows, Mac) antes de confiar en el flujo.

## Estructura

```
PLANO.md                               plan por fases (Frente A negocio, Frente B Claude Code)
telemetria/                            Frente C: telemetrÃ­a local de agentes (Claude Code + Codex)
help.sh                                ayuda + activar/desactivar telemetrÃ­a
prompts/01-dashboard-negocio.md        panel a partir de una app
prompts/02-conferir-numero.md          checklist para verificar un nÃºmero
prompts/03-base-central.md             base central en Supabase
prompts/04-observabilidade-claude-code.md  prompt Ãºnico del panel de Claude Code + diagnÃ³stico
prompts/05-agendar-atualizacao.md      copia programada
prompts/06-diagnostico-negocio.md      diagnÃ³stico de solo lectura â†’ plan â†’ seguimiento
config/claude-code-otel.example.json   bloque env de Claude Code
guia/                                  pÃ¡gina landing + guÃ­a (PT, EN, ES)
```

## Estado

La **telemetrÃ­a de agentes (Frente C)** ya estÃ¡ implementada y probada en Linux (`telemetria/`). Plan + kit de prompts. El panel de Claude Code aÃºn no se ha armado de punta a punta a partir de este kit; trata la primera ejecuciÃ³n como piloto y verifica cada paso antes de dar el OK a Claude.

---

[ARCKHÉ AI](https://ARCKHÉ AI)
