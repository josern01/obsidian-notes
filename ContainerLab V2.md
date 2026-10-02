---
share_link: https://share.note.sx/7p5txcnt
share_updated: 2026-10-01T22:15:08-06:00
---

Creamos un versión 2 usando React Flow, backend con python y frontend con nodejs
Son dos aplicaciones completamente independientes que se comunican por HTTP: el backend corre en `http://127.0.0.1:8000` (FastAPI/uvicorn) y el frontend en `http://localhost:5173` (servidor de desarrollo de Vite). No comparten proceso ni código — solo hablan vía `fetch`.


```
app.v2/
├── backend/
│   ├── backend.py          ← (nodos/enlaces dinámicos)
│   └── requirements.txt
└── frontend/               ← se crea con 'npm create vite@latest'
    └── src/
        ├── App.jsx          ← importa y renderiza TopologyEditor
        └── TopologyEditor.jsx
```


### Cómo se conecta todo con Containerlab — el flujo completo

```
Usuario arrastra nodos/edges en el lienzo (React Flow, en memoria del navegador)
            │
            ▼  clic en "Desplegar topología"
Frontend convierte {nodes, edges} de React Flow → JSON {nodos, enlaces}
            │
            ▼  fetch POST http://127.0.0.1:8000/topologias
Backend (FastAPI) recibe y valida el JSON con Pydantic
            │
            ▼
construir_yaml_clab() arma el diccionario topology.nodes / topology.links
            │
            ▼
yaml.dump() → texto YAML → se guarda en disco como lab-demo.yaml
            │
            ▼
subprocess.run(["sudo", "clab", "deploy", "-t", "lab-demo.yaml", ...])
            │
            ▼
Containerlab lee el YAML, crea los contenedores Docker y los conecta en red
            │
            ▼
clab devuelve JSON por stdout (nombres de contenedor, IPs asignadas)
            │
            ▼
Backend reenvía ese resultado al frontend como respuesta HTTP
            │
            ▼
Frontend muestra el YAML + la salida de clab en pantalla
```



# Frontend

Usa la librería **React Flow**, que resuelve toda la parte visual de "editor de diagramas" (arrastrar nodos, conectar con líneas, mover el lienzo) para que tú no tengas que programar eso desde cero.

#### **Conceptos clave de React Flow que usa el componente:**

- `useNodesState` / `useEdgesState` — dos hooks que mantienen el estado del lienzo: un arreglo de `nodes` (cada uno con `id`, `position {x,y}`, y `data` con lo que quieras guardar) y un arreglo de `edges` (`{source, target}`). React Flow se encarga de redibujar el lienzo cada vez que estos arreglos cambian.
- `onNodesChange` / `onEdgesChange` — callbacks que React Flow llama automáticamente cuando el usuario arrastra, mueve o borra algo; tú solo los conectas a los hooks de arriba y el estado se actualiza solo.
- `onConnect` — se dispara cuando el usuario arrastra desde el borde (_handle_) de un nodo hasta otro. En el componente, esto llama a `addEdge(params, eds)`, que agrega el nuevo enlace al arreglo de `edges`.
- `<ReactFlow>` — el componente que realmente dibuja el lienzo, recibe `nodes`, `edges` y los callbacks de arriba, más `<Background />` (la cuadrícula de fondo) y `<Controls />` (los botones de zoom).



#### **Flujo dentro del componente:**

1. Los botones de arriba (`router`, `servidor_web`, `base_datos`) llaman a `agregarNodo(tipo)`, que crea un nuevo objeto nodo con un `id` único y lo mete en `data.imagen` ya mapeado desde `CATALOGO_IMAGENES` — así el usuario nunca escribe el nombre de una imagen Docker a mano.
2. El usuario conecta nodos arrastrando — React Flow genera los `edges`.
3. Al hacer clic en "Desplegar topología", la función `desplegarTopologia` recorre `nodes` y `edges` y los traduce al formato exacto que el backend espera (`{nodos: [...], enlaces: [...]}`, usando `origen`/`destino` en vez de `source`/`target`).
4. Hace `fetch(POST /topologias)` con ese JSON, y muestra la respuesta (YAML + salida de `clab`, o el error) en el `<pre>` de abajo.

![[Pasted image 20261001214753.png]]

---

### TopologyEditor.jsx

```jsx
import { useCallback, useState } from "react";
import ReactFlow, {
  Background,
  Controls,
  addEdge,
  useNodesState,
  useEdgesState,
} from "reactflow";
import "reactflow/dist/style.css";

// Catalogo simple de "tipos de nodo" -> que imagen Docker usar.
// Esto es lo que en el editor real seria un dropdown al crear cada nodo.
const CATALOGO_IMAGENES = {
  router: "alpine:latest",
  servidor_web: "vulnerables/web-dvwa",
  base_datos: "mysql:8",
};

let contadorId = 0;
const nuevoId = () => `n${contadorId++}`;

const BACKEND_URL = "http://127.0.0.1:8000";

export default function TopologyEditor() {
  const [nodes, setNodes, onNodesChange] = useNodesState([]);
  const [edges, setEdges, onEdgesChange] = useEdgesState([]);
  const [resultado, setResultado] = useState(null);
  const [cargando, setCargando] = useState(false);

  // Conectar dos nodos arrastrando desde un handle -> crea un edge
  const onConnect = useCallback(
    (params) => setEdges((eds) => addEdge(params, eds)),
    [setEdges]
  );

  // Agrega un nodo nuevo del tipo elegido, en una posicion mas o menos libre
  const agregarNodo = (tipo) => {
    const id = nuevoId();
    const nombre = `${tipo}${contadorId}`;
    setNodes((nds) => [
      ...nds,
      {
        id,
        position: { x: 80 + nds.length * 160, y: 120 },
        data: { label: nombre, imagen: CATALOGO_IMAGENES[tipo] },
      },
    ]);
  };

  // Traduce el estado de React Flow al JSON que espera el backend
  // (ver Topologia/Nodo/Enlace en backend.py)
  const desplegarTopologia = async () => {
    setCargando(true);
    setResultado(null);

    const payload = {
      nodos: nodes.map((n) => ({
        id: n.id,
        nombre: n.data.label,
        imagen: n.data.imagen,
      })),
      enlaces: edges.map((e) => ({ origen: e.source, destino: e.target })),
    };

    try {
      const resp = await fetch(`${BACKEND_URL}/topologias`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(payload),
      });
      const data = await resp.json();
      if (!resp.ok) throw new Error(data.detail || "Error desconocido");
      setResultado({ ok: true, data });
    } catch (err) {
      setResultado({ ok: false, error: err.message });
    } finally {
      setCargando(false);
    }
  };

  return (
    <div style={{ display: "flex", flexDirection: "column", height: "100vh" }}>
      {/* Barra de herramientas */}
      <div style={{ padding: 10, display: "flex", gap: 8, borderBottom: "1px solid #ddd" }}>
        {Object.keys(CATALOGO_IMAGENES).map((tipo) => (
          <button key={tipo} onClick={() => agregarNodo(tipo)}>
            + {tipo}
          </button>
        ))}
        <button
          onClick={desplegarTopologia}
          disabled={cargando || nodes.length === 0}
          style={{ marginLeft: "auto", fontWeight: "bold" }}
        >
          {cargando ? "Desplegando..." : "Desplegar topología"}
        </button>
      </div>

      {/* Lienzo del editor */}
      <div style={{ flex: 1 }}>
        <ReactFlow
          nodes={nodes.map((n) => ({ ...n, data: { label: n.data.label } }))}
          edges={edges}
          onNodesChange={onNodesChange}
          onEdgesChange={onEdgesChange}
          onConnect={onConnect}
          fitView
        >
          <Background />
          <Controls />
        </ReactFlow>
      </div>

      {/* Resultado del deploy */}
      {resultado && (
        <pre
          style={{
            background: resultado.ok ? "#111" : "#400",
            color: resultado.ok ? "#0f0" : "#f88",
            padding: 12,
            maxHeight: 200,
            overflow: "auto",
            margin: 0,
          }}
        >
          {resultado.ok
            ? `${resultado.data.mensaje}\n\n${resultado.data.yaml}`
            : `Error: ${resultado.error}`}
        </pre>
      )}
    </div>
  );
}
```

### App.jsx

```jsx
import TopologyEditor from "./TopologyEditor";

function App() {
  return <TopologyEditor />;
}

export default App;
```

---

# Backend

Es el "traductor + ejecutor". No contiene lógica de red real  delega todo el trabajo pesado a Containerlab por línea de comandos. Tiene tres piezas:
Tenemos el script en python 

```python
"""
Ejemplo MINIMO: recibe una topologia simple (2 nodos, 1 enlace) desde el
frontend, la traduce a YAML de Containerlab y la guarda en disco.

Ejecutar:
    pip install fastapi uvicorn pyyaml
    uvicorn backend:app --reload


"""

import pathlib
import subprocess
import yaml
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel

app = FastAPI()

# Permite que frontend.html (abierto directo en el navegador) llame al backend
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)


# ---- 1. Lo que llega del frontend (ahora dinamico: N nodos, N enlaces) ----
#
# React Flow maneja internamente "nodes" (con id unico, ej "n1", "n2"...) y
# "edges" (source -> target por id). Replicamos esa misma forma aqui para
# no tener que traducir nada raro del lado del frontend.

class Nodo(BaseModel):
    id: str           # id interno de React Flow, ej "n1"
    nombre: str        # nombre legible, ej "router1" (sera el nombre del contenedor)
    imagen: str         # ej: "alpine:latest"


class Enlace(BaseModel):
    origen: str   # id del nodo origen (coincide con Nodo.id)
    destino: str  # id del nodo destino


class Topologia(BaseModel):
    nodos: list[Nodo]
    enlaces: list[Enlace]


# ---- 2. Traducir ese JSON al YAML que Containerlab espera -----------------

def construir_yaml_clab(topo: Topologia) -> str:
    if not topo.nodos:
        raise HTTPException(status_code=400, detail="La topologia necesita al menos un nodo")

    # mapa id de React Flow -> nombre legible (el nombre es lo que usa clab)
    id_a_nombre = {nodo.id: nodo.nombre for nodo in topo.nodos}

    nodes_yaml = {
        nodo.nombre: {"kind": "linux", "image": nodo.imagen} for nodo in topo.nodos
    }

    # contador de interfaz por nodo, porque cada link usa ethN distinto
    siguiente_eth = {nodo.nombre: 1 for nodo in topo.nodos}
    links_yaml = []

    for enlace in topo.enlaces:
        origen = id_a_nombre.get(enlace.origen)
        destino = id_a_nombre.get(enlace.destino)
        if origen is None or destino is None:
            raise HTTPException(
                status_code=400,
                detail=f"Enlace invalido: {enlace.origen} -> {enlace.destino} (id no encontrado)",
            )
        eth_origen = siguiente_eth[origen]
        eth_destino = siguiente_eth[destino]
        links_yaml.append({"endpoints": [f"{origen}:eth{eth_origen}", f"{destino}:eth{eth_destino}"]})
        siguiente_eth[origen] += 1
        siguiente_eth[destino] += 1

    estructura = {
        "name": "lab-demo",
        "topology": {"nodes": nodes_yaml, "links": links_yaml},
    }
    return yaml.dump(estructura, sort_keys=False)


# ---- 3. Endpoint que el frontend llama -------------------------------------

@app.post("/topologias")
def crear_topologia(topo: Topologia):
    yaml_texto = construir_yaml_clab(topo)

    # 1. Guardamos el YAML en disco
    archivo_yaml = pathlib.Path("lab-demo.yaml")
    archivo_yaml.write_text(yaml_texto)

    # 2. Corremos 'clab deploy' automaticamente como subproceso.
    #    Containerlab necesita permisos de red (namespaces), por eso 'sudo'.
    #    Para que esto funcione sin pedir password cada vez, configura en
    #    el servidor: sudo visudo  ->  agrega una linea tipo
    #    "usuario ALL=(ALL) NOPASSWD: /usr/bin/clab"
    comando = ["sudo", "clab", "deploy", "-t", str(archivo_yaml), "--format", "json"]

    try:
        resultado = subprocess.run(
            comando,
            capture_output=True,
            text=True,
            timeout=120,  # evita que el request se quede colgado si algo falla
        )
    except FileNotFoundError:
        raise HTTPException(
            status_code=500,
            detail="No se encontro el comando 'clab'. ¿Esta instalado Containerlab y en el PATH?",
        )
    except subprocess.TimeoutExpired:
        raise HTTPException(status_code=504, detail="clab deploy tardo demasiado (timeout).")

    if resultado.returncode != 0:
        # clab devuelve el error util en stderr (ej. imagen no encontrada,
        # interfaz ya en uso, falta sudo, etc.)
        raise HTTPException(status_code=500, detail=resultado.stderr.strip())

    return {
        "mensaje": "Topologia desplegada automaticamente con clab deploy",
        "archivo_yaml": str(archivo_yaml.resolve()),
        "yaml": yaml_texto,
        "clab_stdout": resultado.stdout,  # contiene nombres de contenedores, IPs, etc.
    }


@app.delete("/topologias")
def destruir_topologia():
    """Limpia el laboratorio desplegado (util mientras pruebas)."""
    archivo_yaml = pathlib.Path("lab-demo.yaml")
    if not archivo_yaml.exists():
        raise HTTPException(status_code=404, detail="No hay topologia desplegada (no existe lab-demo.yaml)")

    resultado = subprocess.run(
        ["sudo", "clab", "destroy", "-t", str(archivo_yaml)],
        capture_output=True,
        text=True,
    )
    if resultado.returncode != 0:
        raise HTTPException(status_code=500, detail=resultado.stderr.strip())

    return {"mensaje": "Topologia destruida", "clab_stdout": resultado.stdout}
```


#### **1. Modelos de datos (Pydantic)**


```python
class Nodo(BaseModel):
    id: str       # id interno de React Flow, ej "n1"
    nombre: str    # nombre que tendrá el contenedor
    imagen: str     # imagen Docker, ej "alpine:latest"

class Enlace(BaseModel):
    origen: str   # id del nodo origen
    destino: str  # id del nodo destino

class Topologia(BaseModel):
    nodos: list[Nodo]
    enlaces: list[Enlace]
```

Esto define exactamente qué JSON espera recibir del frontend. FastAPI valida automáticamente la forma del dato — si el frontend manda algo mal formado, rechaza la petición antes de que tu código corra.


#### **2. Traducción a YAML (`construir_yaml_clab`)**  
Recorre la lista de nodos y enlaces, y arma el diccionario con la estructura exacta que Containerlab necesita (`topology.nodes`, `topology.links` con `endpoints`), llevando un contador de interfaz (`eth1`, `eth2`...) por cada nodo para no repetir interfaces si un nodo tiene varias conexiones. Al final, `yaml.dump()` lo convierte a texto YAML real.

#### **3. Endpoints**

- `POST /topologias` — recibe la topología, genera el YAML, lo guarda en disco como `lab-demo.yaml`, y ejecuta `sudo clab deploy -t lab-demo.yaml --format json` como subproceso (`subprocess.run`). Devuelve el YAML generado y la salida de `clab` (que incluye nombres de contenedores e IPs asignadas).
- `DELETE /topologias` — corre `sudo clab destroy -t lab-demo.yaml` para limpiar el laboratorio.

El backend solo arma el archivo correcto y le delega la ejecución a la herramienta de línea de comandos `clab`, capturando su salida o su error.


### requirements.txt

```
fastapi
uvicorn
pyyaml
```


---


![[Pasted image 20261001214753.png]]

Vemos que despliega el yaml

```yaml
Topologia desplegada automaticamente con clab deploy

name: lab-demo
topology:
  nodes:
    router1:
      kind: linux
      image: alpine:latest
    router2:
      kind: linux
      image: alpine:latest
  links:
  - endpoints:
    - router2:eth1
    - router1:eth1
```

Esto se despliega automáticamente a containerlab

![[Pasted image 20261001214853.png]]

Podemos ver que incluso con la extension en visible la topología que hicimos a partir de React Flow 

![[Pasted image 20261001214949.png]]

Tenemos corriendo dos servicios, el backend y frontend

![[Pasted image 20261001215033.png]]

![[Pasted image 20261001215045.png]]


> [!Iimpo] Resumen
> EN ESTE MOMENTO TENEMOS UNA ESTRUCTURA INICIAL DE COMO EL BACKEND SE PUEDE COMUNICAR CON EL REACT FLOW E INTERACTUAR CON CONTAINERLAB, A PARTIR DE AQUI ES MEJORAR EL LIENZO PARA QUE EL USARIO PUEDA ESTRUCTURAR MEJOR LA TOPLOGIA DE RED, Y POSTERIORMENTE EMPEZAR A CONECTAR LA TOPOLOGIA YA DESPLEGADA CON **MITRE Caldera** PARA LANZAR EL ATAQUE CONTRA UN NODO ESPECIFICO
