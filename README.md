# ESXi MCP Server

An MCP server for administering VMware ESXi or vCenter through `pyVmomi`. It includes native SSE transport and a gateway that adapts it to **Streamable HTTP**, enabling integration with current MCP clients such as APX.

> **Status:** connectivity, MCP initialization, and tool discovery have been verified. Real administrative operations against ESXi (`tools/call`) still require an explicitly authorized test on a non-critical VM.

## Architecture

```text
MCP client (APX)
  └─ POST /mcp :8009       Streamable HTTP
       └─ Supergateway
            └─ GET /sse :8080 and POST /sse/messages  SSE
                 └─ ESXi MCP Server (Python/Uvicorn/pyVmomi)
                      └─ VMware SOAP HTTPS :443
                           └─ ESXi or vCenter
```

| Component | Endpoint / purpose |
|---|---|
| Streamable HTTP client | `http://<host>:8009/mcp` |
| Legacy SSE | `http://<host>:8008/sse` |
| SSE messages | `/sse/messages?session_id=<id>` |
| Hypervisor | `https://<esxi-or-vcenter>` |

## Implemented fixes and improvements

- Added **Supergateway** to Docker Compose to translate native SSE into Streamable HTTP at `/mcp` (port `8009`). This fixes the `Not Found` response from clients that send `POST /mcp`.
- Fixed the SSE flow: `connect_sse()` now manages the ASGI response without a manual `http.response.start`. The endpoint correctly returns the event containing the message URL and a per-connection `session_id`.
- Replaced the nonexistent `handle_post()` call with `handle_post_message(scope, receive, send)`, which is compatible with the installed MCP SDK.
- Registered the MCP `tools/list` and `tools/call` handlers with `add_request_handler`, replacing decorators unsupported by the installed version.
- Fixed permissions for the `./logs` bind mount so the non-privileged container user can write logs.
- Added a health check, resource limits, a dedicated bridge network, and `no-new-privileges` to both Compose services.
- Real configuration, secrets, logs, and backups are excluded from Git. Use `config.yaml.sample` as the starting point.

## Available MCP tools

The `tools/list` discovery response contains the following eight tools:

| Tool | Purpose |
|---|---|
| `authenticate` | MCP authentication when an API key is configured |
| `listVMs` | List virtual machines |
| `powerOn` | Power on a VM |
| `powerOff` | Force power off a VM |
| `createVM` | Create a VM |
| `cloneVM` | Clone a VM |
| `deleteVM` | Permanently delete a VM |
| `vmStats` | Retrieve performance statistics |

Registering a tool and seeing it in `tools/list` **does not prove** that execution against ESXi has been validated. In particular, `powerOff` is a forced shutdown.

## Requirements

- Docker Engine and Docker Compose v2 (recommended), or Python 3.7+ for local execution.
- HTTPS network access to ESXi/vCenter.
- A VMware account with the minimum required privileges.

Python dependencies: `pyvmomi`, `pyyaml`, `uvicorn`, and `mcp`.

## Deployment with Docker Compose

1. Create a local configuration from the sample:

```bash
cp config.yaml.sample config.yaml
mkdir -p logs
```

2. Edit `config.yaml` and/or set the required environment variables. Do not add credentials to the repository:

```yaml
vcenter_host: "<esxi-or-vcenter>"
vcenter_user: "<username>"
vcenter_password: "<password>"
datacenter: ""                 # Optional; empty for standalone ESXi
cluster: ""                    # Optional; empty for standalone ESXi
datastore: "<datastore>"
network: "VM Network"
insecure: false                 # Set true only if you understand the risk
api_key: "<optional-api-key>"
log_file: "./logs/vmware_mcp.log"
log_level: "INFO"
```

3. Set the log directory ownership to the container UID/GID, then start the services:

```bash
MCP_UID=$(docker compose run --rm --no-deps --entrypoint id esxi-mcp-server -u)
MCP_GID=$(docker compose run --rm --no-deps --entrypoint id esxi-mcp-server -g)
sudo chown -R "${MCP_UID}:${MCP_GID}" ./logs
sudo chmod -R u+rwX ./logs

docker compose up -d --build
```

4. Configure the MCP client with the Streamable HTTP URL:

```text
http://<docker-host>:8009/mcp
```

The direct SSE endpoint (`http://<docker-host>:8008/sse`) remains available for clients that still use the legacy SSE transport.

## Validation and troubleshooting

After changing `server.py`:

```bash
python3 -m py_compile server.py
docker compose up -d --build --force-recreate esxi-mcp-server
docker compose restart esxi-mcp-gateway
```

Operational commands:

```bash
docker compose ps
docker compose logs --tail=100 esxi-mcp-server
docker compose logs --tail=100 esxi-mcp-gateway
```

The following checks were completed during the fixes:

- `GET /sse` returns the `endpoint` event with `/sse/messages?session_id=...`.
- Supergateway translates initialization traffic from `/mcp` to SSE.
- `initialize` and `tools/list` return successful JSON-RPC responses.
- `tools/list` discovers the eight tools listed above.

## Security and operations

- Do not commit `config.yaml`, `.env` files, logs, tokens, or passwords.
- Prefer valid TLS (`insecure: false`), API key authentication, network segmentation, and a firewall that exposes the service only to authorized clients.
- Pin MCP SDK and Supergateway versions before production use.
- Apply least privilege in ESXi/vCenter. VMware remains responsible for authentication, authorization, and effective hypervisor privileges.
- Before enabling operational administration, inspect gateway logs during an authorized `tools/call` and test with a non-critical VM.

## License

MIT License.

## Credits

- VMware pyVmomi
- MCP Protocol
- Supergateway
