---
share_link: https://share.note.sx/k24nlhzf
share_updated: 2026-09-15T15:32:56-06:00
---

### Módulo 1 — Simulación (topología + ataques)

**Para la topología virtual:**

- **Containerlab** — el más práctico hoy en día. Define topologías completas (routers, hosts, switches) en YAML y las levanta con contenedores Docker en segundos. Ideal porque tú ya tienes experiencia con Docker.

	
> [!Importante] Importante
> Por ahora el que mejor se adapta sería Containerlab



- **GNS3** o **EVE-NG** — más pesados pero con interfaz gráfica de "arrastrar y soltar" nodos, útiles si quieres que el usuario diseñe la topología visualmente. GNS3 tiene API REST, así que podrías construir tu propio frontend web encima.
- **Vagrant + VirtualBox/libvirt** — si necesitas VMs completas (no solo contenedores) para ciertos servicios vulnerables.

**Para ejecutar los ataques:**

- **MITRE Caldera** — framework de emulación de adversarios (ATT&CK), con API REST y "abilities" ya catalogadas por técnica. Es probablemente tu mejor punto de partida porque ya viene con un motor de orquestación y planificación de ataques.
- **Atomic Red Team** — biblioteca de pruebas atómicas mapeadas a MITRE ATT&CK, fácil de invocar por script.
- **Metasploit Framework** — para explotación clásica (tiene msfrpcd, un daemon RPC que puedes controlar programáticamente).
- **Infection Monkey** (Guardicore) — simulador de "breach and attack" que ya reporta resultados en JSON, bueno como referencia de diseño.

### Módulo 2 — Monitoreo y métricas

- **Prometheus + Grafana** — el estándar de facto. Prometheus recolecta métricas de recursos (CPU, RAM, red) vía _exporters_ (node_exporter, cAdvisor para contenedores); Grafana las visualiza en dashboards en tiempo real.
- **Zeek** (antes Bro) — captura y analiza tráfico de red a nivel de protocolo, genera logs estructurados perfectos para medir latencia, conexiones, servicios afectados.
- **ntopng** — monitoreo de tráfico con métricas de disponibilidad y flujos, tiene API REST.
- **cAdvisor + node_exporter** — si tu topología corre en contenedores (Containerlab), estos te dan consumo de recursos por nodo automáticamente.

### Módulo 3 — Detección y neutralización

- **Suricata** o **Snort** — IDS/IPS: detectan comportamiento malicioso por firmas/reglas y pueden bloquear tráfico (modo IPS) automáticamente.
- **Wazuh** — SIEM/EDR open source, correlaciona eventos, tiene reglas de detección y **active response** (scripts que ejecutan la neutralización: bloquear IP, matar proceso, aislar host). Este es clave porque ya integra detección + respuesta en un solo motor con API.
- **TheHive + Cortex** — si quieres un flujo de "caso → análisis → respuesta" más formal (orquestación tipo SOAR ligero).

```
┌─────────────────────────────────────────────┐
│         Backend orquestador (FastAPI/Node)    │
│  - Expone API REST para el frontend           │
│  - Coordina el ciclo completo                 │
└───────┬───────────┬───────────┬──────────────┘
        │            │           │
   Containerlab   Caldera/    Wazuh/Suricata
   (topología)   Metasploit   (detección+resp.)
        │         (ataques)        │
        └────────────┬─────────────┘
                      │
            Prometheus + Zeek
              (métricas)
                      │
                  Grafana
            (dashboard/frontend)
```

La pieza que realmente **une** todo es tu propio backend orquestador: un servicio (Python con FastAPI es buena opción

1. Recibe la topología del usuario → la traduce a YAML de Containerlab y la despliega.
2. Recibe el tipo de ataque seleccionado → llama a la API de Caldera (o ejecuta un script Atomic Red Team) contra el nodo objetivo.
3. En paralelo, consulta las APIs de Prometheus y Zeek para ir registrando métricas del impacto.
4. Escucha las alertas de Wazuh/Suricata (webhooks o su API) y dispara la neutralización.
5. Al final del ciclo, junta todos los datos (ataque ejecutado, métricas durante el ataque, tiempo de detección, acción de neutralización) y genera el reporte de evaluación.

---
### Backend Orquestador



- **Gestión de escenarios de ataque** — recibe "ataque X contra nodo Y", lo traduce a una llamada a la API de Caldera (o a un script de Atomic Red Team) y dispara la ejecución.
- **Recolección de métricas** — hace polling (o se suscribe) a Prometheus y Zeek durante la ventana de tiempo del ataque, y guarda esas métricas asociadas a esa ejecución específica.
- **Gestión de detección/respuesta** — escucha alertas de Wazuh/Suricata (vía webhook o su API), y cuando detecta una, dispara la acción de neutralización y registra el tiempo que tardó en detectarse.
- **Máquina de estados del ciclo** — mantiene el estado de cada "corrida": `desplegando → atacando → monitoreando → detectado → neutralizado → evaluado`. Esto es importante para tu tesis porque es literalmente el ciclo que quieres demostrar.
- **Persistencia** — guarda en una base de datos (Postgres o incluso SQLite para empezar) cada ejecución: topología usada, ataque, métricas recolectadas, tiempo de detección, acción tomada. Esto es tu fuente de datos para evaluar resultados y generar gráficas comparativas en la tesis.
- **API REST (y WebSocket)** — expone endpoints para que el frontend consuma todo esto, y idealmente un canal WebSocket para enviar eventos en tiempo real (ej. "ataque detectado" aparece al instante sin que el frontend tenga que refrescar).


### Ejemplo de endpoints

```
POST   /topologies          → crear/desplegar topología
GET    /topologies/:id      → estado de la topología
POST   /attacks             → lanzar ataque {tipo, objetivo}
GET    /runs/:id/metrics    → métricas de una ejecución
GET    /runs/:id/events     → línea de tiempo del ciclo
WS     /runs/:id/live       → eventos en tiempo real
```

### Frontend

Es la interfaz donde el usuario interactúa con todo el sistema sin tener que tocar YAML de Containerlab o la consola de Wazuh directamente.

**Pantallas principales:**

1. **Editor de topología** — un lienzo donde el usuario arrastra nodos (host, router, servidor web, base de datos) y los conecta. Puede ser tan simple como un editor tipo diagrama (librerías como React Flow te dan esto casi listo) que al final genera el JSON/YAML que el backend traduce a Containerlab.
2. **Selector de ataques** — catálogo de escenarios disponibles (ej. "fuerza bruta SSH", "escaneo de puertos", "ransomware simulado", "DoS"), con descripción de qué hace y contra qué tipo de nodo aplica. Al seleccionar, llama a `POST /attacks`.
3. **Dashboard de métricas en vivo** — aquí es donde puedes ahorrarte mucho trabajo **embebiendo paneles de Grafana** (vía iframe con snapshot/URL pública) en lugar de graficar tú mismo con D3/Chart.js. Se actualiza con lo que llega por WebSocket.
4. **Línea de tiempo del ciclo** — vista tipo "stepper" (ataque → monitoreo → detección → neutralización → evaluación) mostrando en qué fase va la ejecución actual, con timestamps.
5. **Reporte final** — al terminar una corrida, resumen comparativo: impacto antes/después de la neutralización, tiempo de detección, servicios afectados, etc. — esto es literalmente el material que vas a usar como evidencia en tu tesis.


**Stack sugerido:** **React** con **React Flow** (para el editor de topología) es la combinación más usada para este tipo de herramientas — de hecho es lo que usan proyectos similares como algunos plugins de GNS3 web y editores de Kubernetes. Tailwind para estilos rápido, y un cliente WebSocket simple para los eventos en vivo.


### Que haría ReactFlow

Containerlab **no tiene API REST propia** — es una herramienta de línea de comandos (`clab deploy -t topo.yaml`). Así que tu backend orquestador es quien hace de "traductor + ejecutor de CLI". Vamos paso por paso.

#### 1. Lo que React Flow te da

Cuando el usuario arrastra nodos y los conecta, React Flow internamente mantiene dos arrays: `nodes` y `edges`. Al guardar, envías algo así al backend:

json

```json
{
  "nodes": [
    { "id": "n1", "data": { "label": "router1", "kind": "linux", "image": "alpine:latest" } },
    { "id": "n2", "data": { "label": "webserver1", "kind": "linux", "image": "vulnerables/web-dvwa" } }
  ],
  "edges": [
    { "source": "n1", "target": "n2" }
  ]
}
```

`kind` e `image` los defines tú en el editor (ej. un dropdown: "router genérico", "servidor web vulnerable", "base de datos") — cada opción mapea a una imagen Docker predefinida en tu catálogo.

#### 2. El backend traduce ese JSON al YAML de Containerlab

El formato que espera Containerlab es este:

yaml

```yaml
name: lab-usuario123
topology:
  nodes:
    router1:
      kind: linux
      image: alpine:latest
    webserver1:
      kind: linux
      image: vulnerables/web-dvwa
  links:
    - endpoints: ["router1:eth1", "webserver1:eth1"]
```

Y tu backend (Python) simplemente arma ese diccionario y lo vuelca a YAML:

python

```python
import yaml

def build_clab_topology(nodes: list, edges: list, lab_name: str) -> str:
    topo = {
        "name": lab_name,
        "topology": {
            "nodes": {},
            "links": []
        }
    }

    # mapa id de React Flow -> nombre legible del nodo
    id_to_name = {}

    for node in nodes:
        name = node["data"]["label"]
        id_to_name[node["id"]] = name
        topo["topology"]["nodes"][name] = {
            "kind": node["data"].get("kind", "linux"),
            "image": node["data"]["image"]
        }

    for i, edge in enumerate(edges):
        src = id_to_name[edge["source"]]
        dst = id_to_name[edge["target"]]
        topo["topology"]["links"].append({
            "endpoints": [f"{src}:eth{i+1}", f"{dst}:eth{i+1}"]
        })

    return yaml.dump(topo, sort_keys=False)
```

#### 3. El backend ejecuta Containerlab como subproceso

Esto es lo importante: no hay "API de Containerlab" que llamar, así que tu backend guarda el YAML en disco y corre el comando por `subprocess`:

python

```python
import subprocess, uuid, pathlib

def deploy_topology(nodes, edges, user_id) -> dict:
    lab_name = f"lab-{user_id}-{uuid.uuid4().hex[:6]}"
    yaml_content = build_clab_topology(nodes, edges, lab_name)

    path = pathlib.Path(f"/tmp/topologies/{lab_name}.yaml")
    path.parent.mkdir(parents=True, exist_ok=True)
    path.write_text(yaml_content)

    result = subprocess.run(
        ["sudo", "clab", "deploy", "-t", str(path), "--format", "json"],
        capture_output=True, text=True
    )

    if result.returncode != 0:
        raise RuntimeError(result.stderr)

    return {"lab_name": lab_name, "deploy_output": result.stdout}
```

Nota: `clab deploy --format json` devuelve la info de los contenedores desplegados (nombres, IPs, interfaces) en JSON — esa salida se la guardas en tu base de datos porque después el **módulo de ataques** necesita saber la IP de `webserver1` para apuntarle Caldera/Metasploit.

#### 4. Endpoint que conecta todo esto al frontend

python

```python
from fastapi import FastAPI
app = FastAPI()

@app.post("/topologies")
async def create_topology(payload: TopologyRequest, user=Depends(get_user)):
    result = deploy_topology(payload.nodes, payload.edges, user.id)
    save_to_db(user.id, result["lab_name"], payload.nodes, payload.edges)
    return result
```

El frontend llama a `POST /topologies` con el JSON de React Flow, espera la respuesta (o escucha por WebSocket si el despliegue tarda), y cuando llega `deploy_output` puede mostrar "topología lista" y habilitar el selector de ataques.

#### Puntos importantes a cuidar

- **Permisos**: Containerlab necesita privilegios (namespaces de red), así que tu backend probablemente corre como root o con `sudo` sin password para ese comando específico — considera correr el backend dentro de una VM aislada por seguridad, dado que además vas a lanzar ataques reales.
- **Validación**: antes de traducir, valida que el grafo sea válido (nodos sin conexión huérfanos, ciclos no soportados, imágenes permitidas) — si no, `clab deploy` falla con errores crípticos.
- **Destrucción**: guarda también un endpoint `DELETE /topologies/:id` que corra `clab destroy -t archivo.yaml` para limpiar cuando el usuario termine.

