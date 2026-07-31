---
read_when:
    - Captura de registros de macOS o investigación del registro de datos privados
    - Depuración de problemas de activación por voz y del ciclo de vida de las sesiones
summary: 'Registro de OpenClaw: registro de archivo de diagnóstico rotativo + opciones de privacidad del registro unificado'
title: Registro en macOS
x-i18n:
    generated_at: "2026-07-26T04:47:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ef0fd91bd7fc0a8b5f598cfe8f5de551795a4badd0f6634c5bcbd4f3916bfc64
    source_path: platforms/mac/logging.md
    workflow: 16
---

# Registro (macOS)

## Registro de archivo de diagnóstico rotativo (panel Debug)

La aplicación de macOS registra mediante swift-log (registro unificado de forma predeterminada) y también puede escribir un registro de archivo local rotativo para una captura persistente (`DiagnosticsFileLog`).

- Activación: **Debug pane -> Logs -> App logging -> "Write rolling diagnostics log (JSONL)"** (desactivado de forma predeterminada).
- Nivel de detalle: selector **Debug pane -> Logs -> App logging -> Verbosity**.
- Ubicación: `~/Library/Logs/OpenClaw/diagnostics.jsonl`.
- Rotación: rota al alcanzar 5 MB; conserva hasta 5 copias de seguridad con los sufijos `.1`...`.5` (se descarta la más antigua).
- Borrado: **Debug pane -> Logs -> App logging -> "Clear"** elimina el archivo activo y todas las copias de seguridad.

El archivo debe tratarse como información confidencial; no debe compartirse sin revisarlo.

## Datos privados del registro unificado en macOS

El registro unificado oculta la mayoría de las cargas útiles, salvo que un subsistema habilite `privacy -off`. Esto se controla mediante un plist en `/Library/Preferences/Logging/Subsystems/`, cuya clave es el nombre del subsistema. La opción solo se aplica a las nuevas entradas del registro, por lo que debe habilitarse antes de reproducir un problema. Más información: [peculiaridades de la privacidad del registro de macOS](https://steipete.me/posts/2025/logging-privacy-shenanigans).

## Habilitación para OpenClaw (`ai.openclaw`)

Primero se escribe el plist en un archivo temporal y, después, se instala de forma atómica como root:

```bash
cat <<'EOF' >/tmp/ai.openclaw.plist
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>DEFAULT-OPTIONS</key>
    <dict>
        <key>Enable-Private-Data</key>
        <true/>
    </dict>
</dict>
</plist>
EOF
sudo install -m 644 -o root -g wheel /tmp/ai.openclaw.plist /Library/Preferences/Logging/Subsystems/ai.openclaw.plist
```

No es necesario reiniciar; logd detecta el archivo rápidamente, pero solo las nuevas líneas del registro incluyen cargas útiles privadas. La salida más detallada puede consultarse con `./scripts/clawlog.sh --category WebChat --last 5m` (`--last`/`-l` establece el intervalo de tiempo, cuyo valor predeterminado es `5m`; `--category`/`-c` filtra por categoría).

## Desactivación después de la depuración

- Elimine la anulación: `sudo rm /Library/Preferences/Logging/Subsystems/ai.openclaw.plist`.
- Opcionalmente, ejecute `sudo log config --reload` para forzar que logd descarte la anulación de inmediato.
- Esta superficie puede incluir números de teléfono y cuerpos de mensajes; mantenga el plist instalado solo mientras sea necesario.

## Contenido relacionado

- [Aplicación para macOS](/es/platforms/macos)
- [Registro del Gateway](/es/gateway/logging)
