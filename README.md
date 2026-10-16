# ESXi MCP Server

Servidor MCP para administrar VMware ESXi o vCenter mediante `pyVmomi`. Incluye transporte SSE nativo y un gateway que lo adapta a **Streamable HTTP**, para integrarlo con clientes MCP actuales como APX.

> **Estado:** la conectividad, inicialización MCP y descubrimiento de herramientas fueron verificados. Las operaciones administrativas reales contra ESXi (`tools/call`) requieren una prueba explícitamente autorizada en una VM no crítica.

## Arquitectura

```text
Cliente MCP (APX)
  └─ POST /mcp :8009       Streamable HTTP
       └─ Supergateway
            └─ GET /sse :8080 y POST /sse/messages  SSE
                 └─ ESXi MCP Server (Python/Uvicorn/pyVmomi)
                      └─ VMware SOAP HTTPS :443
                           └─ ESXi o vCenter
```

| Componente | Endpoint / función |
|---|---|
| Cliente Streamable HTTP | `http://<host>:8009/mcp` |
| SSE legado | `http://<host>:8008/sse` |
| Mensajes SSE | `/sse/messages?session_id=<id>` |
| Hipervisor | `https://<esxi-o-vcenter>` |

## Correcciones e implementación realizadas

- Se añadió **Supergateway** en Docker Compose para traducir el SSE nativo a Streamable HTTP en `/mcp` (puerto `8009`). Esto resuelve el `Not Found` de clientes que envían `POST /mcp`.
- Se corrigió el flujo SSE: `connect_sse()` gestiona ahora la respuesta ASGI sin un `http.response.start` manual. El endpoint entrega correctamente el evento con la URL de mensajes y el `session_id` por conexión.
- Se sustituyó la llamada inexistente `handle_post()` por `handle_post_message(scope, receive, send)`, compatible con el SDK MCP instalado.
- Se registraron los handlers MCP de `tools/list` y `tools/call` mediante `add_request_handler`, en vez de decoradores no soportados por la versión instalada.
- Se corrigieron permisos del bind mount `./logs` para que el usuario no privilegiado del contenedor pueda escribir el log.
- Se incorporaron healthcheck, límites de recursos, red bridge dedicada y `no-new-privileges` para ambos servicios de Compose.
- La configuración real, secretos, logs y copias de respaldo están excluidos de Git. Use `config.yaml.sample` como plantilla.

## Herramientas MCP disponibles

El descubrimiento `tools/list` devuelve estas ocho herramientas:

| Herramienta | Uso |
|---|---|
| `authenticate` | Autenticación MCP si se configuró una API key |
| `listVMs` | Enumerar máquinas virtuales |
| `powerOn` | Encender una VM |
| `powerOff` | Apagado forzado de una VM |
| `createVM` | Crear una VM |
| `cloneVM` | Clonar una VM |
| `deleteVM` | Eliminar permanentemente una VM |
| `vmStats` | Consultar estadísticas de rendimiento |

El registro de una herramienta y su aparición en `tools/list` **no demuestra** que su ejecución contra ESXi esté validada. En particular, `powerOff` es un apagado forzado.

## Requisitos

- Docker Engine y Docker Compose v2 (recomendado), o Python 3.7+ para ejecución local.
- Acceso de red HTTPS al ESXi/vCenter.
- Cuenta de VMware con los permisos mínimos necesarios.

Dependencias Python: `pyvmomi`, `pyyaml`, `uvicorn` y `mcp`.

## Despliegue con Docker Compose

1. Cree una configuración local a partir de la plantilla:

```bash
cp config.yaml.sample config.yaml
mkdir -p logs
```

2. Edite `config.yaml` y/o defina las variables necesarias, sin añadir credenciales al repositorio:

```yaml
vcenter_host: "<esxi-o-vcenter>"
vcenter_user: "<usuario>"
vcenter_password: "<contraseña>"
datacenter: ""                 # Opcional; vacío para ESXi standalone
cluster: ""                    # Opcional; vacío para ESXi standalone
datastore: "<datastore>"
network: "VM Network"
insecure: false                 # Use true solo si comprende el riesgo
api_key: "<api-key-opcional>"
log_file: "./logs/vmware_mcp.log"
log_level: "INFO"
```

3. Ajuste los permisos del directorio de logs al UID/GID del contenedor y levante los servicios:

```bash
MCP_UID=$(docker compose run --rm --no-deps --entrypoint id esxi-mcp-server -u)
MCP_GID=$(docker compose run --rm --no-deps --entrypoint id esxi-mcp-server -g)
sudo chown -R "${MCP_UID}:${MCP_GID}" ./logs
sudo chmod -R u+rwX ./logs

docker compose up -d --build
```

4. Configure el cliente MCP con la URL Streamable HTTP:

```text
http://<host-docker>:8009/mcp
```

El endpoint SSE directo (`http://<host-docker>:8008/sse`) queda disponible para clientes que aún utilizan el transporte SSE legado.

## Validación y diagnóstico

Después de modificar `server.py`:

```bash
python3 -m py_compile server.py
docker compose up -d --build --force-recreate esxi-mcp-server
docker compose restart esxi-mcp-gateway
```

Comandos operativos:

```bash
docker compose ps
docker compose logs --tail=100 esxi-mcp-server
docker compose logs --tail=100 esxi-mcp-gateway
```

Validaciones realizadas durante la corrección:

- `GET /sse` devuelve el evento `endpoint` con `/sse/messages?session_id=...`.
- Supergateway traduce la inicialización desde `/mcp` hacia SSE.
- `initialize` y `tools/list` responden exitosamente por JSON-RPC.
- `tools/list` descubre las ocho herramientas descritas arriba.

## Seguridad y operación

- No versionar `config.yaml`, `.env`, logs, tokens ni contraseñas.
- Prefiera TLS válido (`insecure: false`), API key, segmentación de red y un firewall que exponga el servicio solo a clientes autorizados.
- Fije versiones de las dependencias MCP y de Supergateway antes de usarlo en producción.
- Aplique permisos mínimos en ESXi/vCenter. El control de autenticación, autorización y privilegios efectivos permanece en VMware.
- Antes de habilitar administración operativa, revise los logs del gateway durante un `tools/call` autorizado y pruebe con una VM no crítica.

## Licencia

MIT License.

## Créditos

- VMware pyVmomi
- MCP Protocol
- Supergateway
