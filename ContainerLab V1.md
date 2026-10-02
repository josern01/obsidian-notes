---
share_link: https://share.note.sx/y5sbsjmy
share_updated: 2026-10-01T20:21:34-06:00
---

![Screenshot 2026-10-01 185201.png](Screenshot%202026-10-01%20185201.png)

Mediante WSL corremos containerlab, podemos ver la interfaz grafica en VScode

Pequeño ejemplo desde una pagina web donde se el user puede añadir un nodo y crear la topología, esete crea un yamal, el backend el python levanta automaticamente en container lab

Se construyó un ejemplo mínimo funcional para probar el flujo completo antes de meter React Flow.  
backend.py (FastAPI): recibe JSON con 2 nodos, lo traduce a YAML de Containerlab, lo guarda en disco, y ahora ejecuta automáticamente "clab deploy" vía subprocess para desplegar la topología real. 
frontend.html: formulario simple en HTML/JS puro (sin framework todavía) que manda el JSON al backend con fetch y muestra el YAML generado.

![Pasted image 20261001201814.png](Pasted%20image%2020261001201814.png)

Podemos observar que mediante la extension de container lab se construye 

![Pasted image 20261001201845.png](Pasted%20image%2020261001201845.png)

### Frontend simple

```html
<!DOCTYPE html>
<html lang="es">
<head>
<meta charset="UTF-8">
<title>Demo: Editor de topologia -> Containerlab</title>
<style>
  body { font-family: system-ui, sans-serif; max-width: 480px; margin: 40px auto; }
  label { display: block; margin-top: 12px; font-size: 14px; color: #333; }
  input { width: 100%; padding: 6px; box-sizing: border-box; }
  button { margin-top: 16px; padding: 8px 16px; cursor: pointer; }
  pre { background: #111; color: #0f0; padding: 12px; margin-top: 16px; white-space: pre-wrap; }
</style>
</head>
<body>

<h2>Crear topología (ejemplo mínimo: 2 nodos)</h2>

<label>Nombre nodo A
  <input id="nombreA" value="router1">
</label>
<label>Imagen Docker nodo A
  <input id="imagenA" value="alpine:latest">
</label>

<label>Nombre nodo B
  <input id="nombreB" value="webserver1">
</label>
<label>Imagen Docker nodo B
  <input id="imagenB" value="vulnerables/web-dvwa">
</label>

<button onclick="crearTopologia()">Crear topología</button>

<pre id="resultado">(el YAML generado aparecerá aquí)</pre>

<script>
async function crearTopologia() {
  const payload = {
    nodo_a: {
      nombre: document.getElementById("nombreA").value,
      imagen: document.getElementById("imagenA").value
    },
    nodo_b: {
      nombre: document.getElementById("nombreB").value,
      imagen: document.getElementById("imagenB").value
    }
  };

  const resp = await fetch("http://127.0.0.1:8000/topologias", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(payload)
  });

  const data = await resp.json();
  document.getElementById("resultado").textContent =
    data.mensaje + "\n\n" + data.yaml;
}
</script>

</body>
</html>
```

### Backend

```python
"""
Ejemplo MINIMO: recibe una topologia simple (2 nodos, 1 enlace) desde el
frontend, la traduce a YAML de Containerlab y la guarda en disco.

Ejecutar:
    pip install fastapi uvicorn pyyaml
    uvicorn backend:app --reload

Luego abrir frontend.html  en el navegador.
Ejemplol: file://wsl.localhost/Ubuntu/home/joser/tesis/containerlab/app/frontend.html
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


# ---- 1. Lo que llega del frontend -----------------------------------------

class Nodo(BaseModel):
    nombre: str
    imagen: str  # ej: "alpine:latest"


class Topologia(BaseModel):
    nodo_a: Nodo
    nodo_b: Nodo


# ---- 2. Traducir ese JSON al YAML que Containerlab espera -----------------

def construir_yaml_clab(topo: Topologia) -> str:
    estructura = {
        "name": "lab-demo",
        "topology": {
            "nodes": {
                topo.nodo_a.nombre: {"kind": "linux", "image": topo.nodo_a.imagen},
                topo.nodo_b.nombre: {"kind": "linux", "image": topo.nodo_b.imagen},
            },
            "links": [
                {"endpoints": [f"{topo.nodo_a.nombre}:eth1", f"{topo.nodo_b.nombre}:eth1"]}
            ],
        },
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