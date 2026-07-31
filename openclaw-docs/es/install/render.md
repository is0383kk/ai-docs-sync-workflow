---
read_when:
    - Implementación de OpenClaw en Render
    - Se desea un despliegue declarativo en la nube con Render Blueprints
summary: Despliega OpenClaw en Render con infraestructura como código
title: Render
x-i18n:
    generated_at: "2026-07-26T05:44:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a5fbb3c6df04e186df958a62a6130da4e3e485acfeecc7e85fee0d5b69a0438f
    source_path: install/render.mdx
    workflow: 16
---

Implementa OpenClaw en [Render](https://render.com) mediante el Blueprint `render.yaml` del repositorio. Este declara el servicio, el disco y las variables de entorno en un solo archivo.

## Requisitos previos

- Una [cuenta de Render](https://render.com) (nivel gratuito disponible)
- Una clave de API de su [proveedor de modelos](/es/providers) preferido

## Implementación

[Implementar en Render](https://render.com/deploy?repo=https://github.com/openclaw/openclaw)

Esto crea un servicio de Render a partir de `render.yaml`, compila la imagen de Docker y la implementa. La URL del servicio sigue el patrón `https://<service-name>.onrender.com`.

## El Blueprint

```yaml
services:
  - type: web
    name: openclaw
    runtime: docker
    plan: starter
    healthCheckPath: /health
    envVars:
      - key: OPENCLAW_GATEWAY_PORT
        value: "8080"
      - key: OPENCLAW_STATE_DIR
        value: /data/.openclaw
      - key: OPENCLAW_WORKSPACE_DIR
        value: /data/workspace
      - key: OPENCLAW_GATEWAY_TOKEN
        generateValue: true # genera automáticamente un token seguro
    disk:
      name: openclaw-data
      mountPath: /data
      sizeGB: 1
```

| Característica               | Propósito                                                    |
| --------------------- | ---------------------------------------------------------- |
| `runtime: docker`     | Compila a partir del Dockerfile del repositorio                          |
| `healthCheckPath`     | Render supervisa `/health` y reinicia las instancias con problemas |
| `generateValue: true` | Genera automáticamente un valor criptográficamente seguro            |
| `disk`                | Almacenamiento persistente que se conserva tras las reimplementaciones                 |

## Elección de un plan

| Plan      | Suspensión         | Disco          | Ideal para                      |
| --------- | ----------------- | ------------- | ----------------------------- |
| Free      | Tras 15 min de inactividad | No disponible | Pruebas, demostraciones                |
| Starter   | Nunca             | 1GB+          | Uso personal, equipos pequeños     |
| Standard+ | Nunca             | 1GB+          | Producción, varios canales |

El Blueprint utiliza `starter` de forma predeterminada. Para usar el nivel gratuito, cambie `plan: free` en el archivo `render.yaml` de su fork. Tenga en cuenta que, al no haber un disco persistente, el estado de OpenClaw se restablece en cada implementación.

## Después de la implementación

### Acceder a la interfaz de control

El panel web está disponible en `https://<your-service>.onrender.com/`. Conéctese mediante el secreto compartido: el valor `OPENCLAW_GATEWAY_TOKEN` generado automáticamente (se encuentra en **Dashboard → your service → Environment**), o mediante su contraseña si cambió a la autenticación por contraseña.

### Registros

**Dashboard → your service → Logs** muestra los registros de compilación (creación de la imagen de Docker), los registros de implementación (inicio del servicio) y los registros de ejecución (salida de la aplicación).

### Acceso al shell

**Dashboard → your service → Shell** abre una sesión de shell. El disco persistente está montado en `/data`.

### Variables de entorno

Edite las variables en **Dashboard → your service → Environment**. Los cambios activan una reimplementación automática.

### Implementación automática

Render vuelve a implementar automáticamente cuando la rama del repositorio conectado recibe un nuevo commit. Si realizó la implementación directamente desde `openclaw/openclaw` en lugar de desde su propio fork, no tendrá acceso de escritura para activarla; para actualizar, ejecute una sincronización manual del Blueprint desde Dashboard o dirija el servicio a su propio fork.

## Dominio personalizado

1. **Dashboard → your service → Settings → Custom Domains**
2. Añada su dominio
3. Configure el DNS según las instrucciones (CNAME a `*.onrender.com`)
4. Render aprovisiona automáticamente un certificado TLS

## Escalado

- **Vertical**: cambie el plan para obtener más CPU/RAM. Suele ser suficiente para OpenClaw.
- **Horizontal**: aumente el número de instancias (plan Standard o superior). Requiere sesiones persistentes o gestión externa del estado, ya que OpenClaw mantiene el estado de ejecución en el disco local.

## Copias de seguridad y migración

Desde el shell de Render Dashboard, exporte en cualquier momento el estado, la configuración, los perfiles de autenticación y el espacio de trabajo:

```bash
openclaw backup create
```

Esto crea un archivo de copia de seguridad portátil. Consulte [Copia de seguridad](/es/cli/backup).

## Solución de problemas

### El servicio no se inicia

Revise los registros de implementación en Render Dashboard. Problemas habituales:

- Falta `OPENCLAW_GATEWAY_TOKEN`: compruebe que esté configurado en **Dashboard → Environment**
- Discrepancia de puertos: asegúrese de que `OPENCLAW_GATEWAY_PORT=8080` para que el Gateway se vincule al puerto que Render espera

### Inicios en frío lentos (nivel gratuito)

Los servicios del nivel gratuito se suspenden tras 15 minutos de inactividad; la primera solicitud después de la suspensión tarda unos segundos mientras se inicia el contenedor. Cambie al plan Starter para mantener el servicio siempre activo.

### Pérdida de datos tras una reimplementación

Ocurre en el nivel gratuito (sin disco persistente). Cambie a un plan de pago o exporte periódicamente una copia de seguridad mediante `openclaw backup create` desde el shell de Render.

### Fallos de la comprobación de estado

Si las compilaciones se completan correctamente, pero las implementaciones fallan, es posible que el servicio tarde demasiado en iniciarse o que no se pueda acceder a `/health`. Compruebe:

- Los registros de compilación en busca de errores
- Si el contenedor se ejecuta localmente con `docker build && docker run`

## Siguientes pasos

- Configure los canales de mensajería: [Canales](/es/channels)
- Configure el Gateway: [Configuración del Gateway](/es/gateway/configuration)
- Mantenga OpenClaw actualizado: [Actualización](/es/install/updating)
