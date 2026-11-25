# ê°Å¸âÅ  CC Dashboard ê¢â¬â paneles en vivo en Claude

[![CC Dashboard](guia/assets/banner-es.jpg)](https://linnoff88-dev.github.io/

**ê°Å¸â¡Â§ê°Å¸â¡Â· [PortuguêÂªs](README.md) êÂ· ê°Å¸â¡Âºê°Å¸â¡Â¸ [English](README.en.md) êÂ· ê°Å¸â¡Âªê°Å¸â¡Â¸ [EspaêÂ±ol](README.es.md)**

## QuêÂ© es

CC Dashboard es un kit en espaêÂ±ol para usar los Dashboards de Claude, paneles que guardan las consultas y rehacen las cuentas cada vez que los abres. Sirve para dueêÂ±os de pequeêÂ±os negocios y para quienes usan Claude Code todos los dêÂ­as. Trae prompts listos para armar paneles del negocio (dinero, atrasos, fugas) y un plan para medir tu propio Claude Code (costo estimado, herramientas, fallas). Para usarlo necesitas un plan de pago de Claude y, en las partes con base de datos, una cuenta gratuita en Supabase.

## ê°Å¸ââ GuêÂ­a de uso

GuêÂ­a completa (landing + paso a paso): **https://linnoff88-dev.github.io/

> Nota: los archivos de prompts (`prompts/*.md`) y el `PLANO.md` estêÂ¡n en portuguêÂ©s.

## CêÂ³mo funciona

Un Dashboard de Claude no guarda nêÂºmeros: guarda **consultas**. Cada vez que lo abres o lo actualizas, vuelve a ejecutar las consultas sobre los datos conectados. Al hacer clic en un nêÂºmero, ves la consulta y la hora de la êÂºltima actualizaciêÂ³n.

**Negocio**

```
Apps (CRM, hoja de cêÂ¡lculo, base de datos) ê¢â â Conector en Claude ê¢â â [Base central en Supabase] ê¢â â Dashboard ê¢â â Copia programada
```

- Una pregunta que cabe en una sola app: conector + prompt y listo.
- Una pregunta que cruza apps (horas êâ facturas, suscripciones êâ cobros): los registros deben copiarse a una base central (Supabase = Postgres alojado), que guarda el historial y permite cruzar.

**Claude Code**

```
Claude Code (OpenTelemetry) ê¢â â Recolector en Supabase ê¢â â Tablas en bruto (7 dêÂ­as) ê¢â â Resumen diario (10 min) ê¢â â Dashboard
```

- Claude Code envêÂ­a su propia telemetrêÂ­a (modelo, tokens, costo estimado, herramientas, duraciêÂ³n, fallas).
- El recolector guarda **solo campos de una lista permitida** y descarta el texto de prompts y respuestas, comandos completos, rutas, entradas y salidas de herramientas, texto de error y correo electrêÂ³nico. Del shell, guarda solo el nombre del programa.
- Un resumen diario se recalcula cada 10 minutos; los datos en bruto se borran a los 7 dêÂ­as.

## CêÂ³mo usar

| Paso | QuêÂ© hacer | Archivo |
|---|---|---|
| 1 | Activar el conector de la app en *Customize ê¢â â Connectors* | ê¢â¬â |
| 2 | Pedir el panel con pregunta, definiciones y "no modifiques datos" | `prompts/01-dashboard-negocio.md` |
| 3 | Verificar un nêÂºmero en el origen antes de actuar | `prompts/02-conferir-numero.md` |
| 4 | Cruzar apps en una base central en Supabase | `prompts/03-base-central.md` |
| 5 | Programar la actualizaciêÂ³n de la base | `prompts/05-agendar-atualizacao.md` |
| 6 | DiagnêÂ³stico del negocio ê¢â â plan ê¢â â panel de seguimiento | `prompts/06-diagnostico-negocio.md` |
| 7 | Armar el panel de Claude Code (base de datos, recolector, pestaêÂ±as) | `prompts/04-observabilidade-claude-code.md` |
| 8 | Activar la telemetrêÂ­a en `~/.claude/settings.json` y **reiniciar** Claude Code | `config/claude-code-otel.example.json` |

Supabase de solo lectura y restringido a un proyecto (conector personalizado):

```
https://mcp.supabase.com/mcp?project_ref=SEU_PROJECT_REF&read_only=true
```

Si no llega nada al recolector: `claude --debug-file /tmp/cc.log` y busca lêÂ­neas `[3P telemetry]`. La tabla de sêÂ­ntomas (401, 404, 415, tarjetas vacêÂ­as) estêÂ¡ al final de `prompts/04-observabilidade-claude-code.md`.

## TelemetrêÂ­a de agentes (Frente C) ê¢â¬â local, sin nube

Un panel en tu propia mêÂ¡quina con **todas** las sesiones de Claude Code (terminales, bots, `claude -p`, subagentes) y de Codex, en vivo: quêÂ© se estêÂ¡ procesando, quêÂ© te espera, quêÂ© fallêÂ³, tokens y costo estimado. Sin API, sin cuenta nueva.

![PestaêÂ±a Ahora del panel de telemetrêÂ­a (interfaz en portuguêÂ©s)](guia/assets/telemetria-agora.jpg)

```bash
./help.sh                          # ayuda completa
./help.sh ligar                    # hooks + OpenTelemetry en ~/.claude/settings.json (con copia), panel y mantenimiento
telemetria/telemetria.sh status    # quêÂ© estêÂ¡ activo ahora
./help.sh desligar                 # quita todo del settings.json y detiene los servicios (la base se conserva)
systemctl --user stop telemetria-painel.service   # solo pausar el panel
```

- **Panel:** `http://127.0.0.1:8766/` y, en la red local, `http://<ip-de-la-maquina>:8766/` (direcciones en `TELEMETRIA_HOSTS`, redes permitidas en `TELEMETRIA_REDES`). PestaêÂ±as: Ahora, Procesos, Consumo, Fallas, Uso, Historial, Salud.
- **QuêÂ© se guarda:** solo nombres (proyecto, herramienta, skill, programa) y nêÂºmeros. Nunca prompts, respuestas, comandos completos, rutas de archivo ni salida de herramientas. Base en `~/.local/share/telemetria-agentes/` (30 dêÂ­as de eventos + resumen diario).
- **Costo de mêÂ¡quina (medido):** ~60 ms mêÂ¡s por llamada de herramienta, ~13 MB de RAM, 3ê¢â¬â5 MB de disco por dêÂ­a (tope ~150 MB). Nada sale de la mêÂ¡quina.
- **LêÂ­mites:** tokens y costo solo en sesiones de Claude Code abiertas despuêÂ©s de activarla; el costo es una estimaciêÂ³n a precio de API (Codex no informa precio); el panel no tiene contraseêÂ±a ê¢â¬â cualquier dispositivo de la red permitida lo ve. Probado en Linux.
- Detalles, pruebas y fases: [`telemetria/PLANO.md`](telemetria/PLANO.md) (en portuguêÂ©s).

## Requisitos previos

- Plan de pago de Claude: Pro, Max, Team o Enterprise (funciêÂ³n en beta, cuenta en el uso del plan).
- Cuenta de Supabase gratuita (500 MB; el proyecto gratuito se pausa tras una semana sin uso) ê¢â¬â para la base central y Claude Code.
- Claude Code ê¢â¬â solo para el panel de observabilidad.

## LêÂ­mites

- **Muestra, no avisa.** Una alerta exige una rutina programada aparte.
- **El costo es una estimaciêÂ³n** a precio de API; en Pro/Max vale la suscripciêÂ³n.
- **Cada persona necesita su propia conexiêÂ³n** al proyecto de Supabase para ver las tarjetas.
- **Los nombres quedan guardados** (skills, repositorios, servidores MCP) ê¢â¬â piensa en quiêÂ©n lo ve.
- **Solo Claude Code**; el chat de la app y Cowork no entran.
- **PruêÂ©balo en tu sistema** (Linux, Windows, Mac) antes de confiar en el flujo.

## Estructura

```
PLANO.md                               plan por fases (Frente A negocio, Frente B Claude Code)
telemetria/                            Frente C: telemetrêÂ­a local de agentes (Claude Code + Codex)
help.sh                                ayuda + activar/desactivar telemetrêÂ­a
prompts/01-dashboard-negocio.md        panel a partir de una app
prompts/02-conferir-numero.md          checklist para verificar un nêÂºmero
prompts/03-base-central.md             base central en Supabase
prompts/04-observabilidade-claude-code.md  prompt êÂºnico del panel de Claude Code + diagnêÂ³stico
prompts/05-agendar-atualizacao.md      copia programada
prompts/06-diagnostico-negocio.md      diagnêÂ³stico de solo lectura ê¢â â plan ê¢â â seguimiento
config/claude-code-otel.example.json   bloque env de Claude Code
guia/                                  pêÂ¡gina landing + guêÂ­a (PT, EN, ES)
```

## Estado

La **telemetrêÂ­a de agentes (Frente C)** ya estêÂ¡ implementada y probada en Linux (`telemetria/`). Plan + kit de prompts. El panel de Claude Code aêÂºn no se ha armado de punta a punta a partir de este kit; trata la primera ejecuciêÂ³n como piloto y verifica cada paso antes de dar el OK a Claude.

---

[ARCKHê AI](https://ARCKHê AI)
