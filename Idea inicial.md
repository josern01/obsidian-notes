---
share_link: https://share.note.sx/6ay6fnrf
share_updated: 2026-09-10T15:25:49-06:00
---

- **Topología definida por el usuario:** servidores, clientes, dispositivos de red, servicios, medios/conexiones, etc.
- **Entorno aislado:** evitar que los ataques tengan impacto sobre una red real.
- **Simulación de escenarios de ataque**, observando cómo afectan a la infraestructura.
- Generar **señales/telemetría del ataque**, como:
    - disponibilidad de servicios
    - latencia
    - tráfico de red
    - consumo de CPU/RAM/recursos
    - servicios afectados
    - alcance del ataque
- Un **sistema de evaluación/puntuación** para determinar el impacto o comportamiento del escenario.
- Habíamos considerado implementar la infraestructura mediante **contenedores**, con la posibilidad de utilizar **Kubernetes** para orquestación.
- También habíamos planteado la posibilidad de **importar topologías de Packet Tracer (`.pkt`)**, aunque eso era una cuestión que todavía había que resolver técnicamente.

La idea central no era simplemente hacer un “programa que lanza ataques”, sino construir una especie de **campo de entrenamiento virtual reproducible**, donde puedas definir una infraestructura, ejecutar un escenario y después medir objetivamente qué ocurrió.


### Podemos usar Mininet-sec
[mininet-sec/README.md at main · mininet-sec/mininet-sec · GitHub](https://github.com/mininet-sec/mininet-sec/blob/main/README.md#getting-started)

### ¿Qué es Mininet-Sec?

Es una plataforma de **emulación de redes orientada específicamente a ciberseguridad**. Permite construir escenarios virtuales para experimentar con ataques y defensas en un entorno aislado. Entre sus capacidades están la emulación de servicios como HTTP, SMTP, IMAP, DNS, LDAP y NTP, además de firewalls, routers, switches y generación de tráfico.

Mininet-Sec ya permite crear escenarios de ataques y probar herramientas de seguridad ofensiva en un entorno aislado.

**Usuario → diseña infraestructura → selecciona escenario → ejecuta ataque → observa consecuencias → obtiene métricas/evaluación**

Mientras que Mininet-Sec proporciona principalmente la **infraestructura de emulación** sobre la que realizar experimentos.

Mininet, por ejemplo, crea hosts, switches y enlaces virtuales utilizando namespaces de Linux, interfaces virtuales, etc.

Mininet-Sec añade encima servicios y componentes de seguridad

```
                 TESIS
                     │
          ┌──────────▼──────────┐
          │  Interfaz / API     │
          │  del simulador      │
          └──────────┬──────────┘
                     │
          ┌──────────▼──────────┐
          │ Definición escenario│
          │                      │
          │ Hosts                │
          │ Servicios            │
          │ Red                  │
          │ Vulnerabilidades    │
          │ Ataque               │
          └──────────┬──────────┘
                     │
          ┌──────────▼──────────┐
          │    Motor de         │
          │     ejecución       │
          └──────────┬──────────┘
                     │
              ┌──────▼──────┐
              │ Mininet-Sec  │
              │ / Mininet    │
              └──────┬──────┘
                     │
       ┌─────────────┼─────────────┐
       ▼             ▼             ▼
    tráfico       servicios      ataques
       │             │             │
       └─────────────┼─────────────┘
                     ▼
             ┌───────────────┐
             │ Monitorización│
             └───────┬───────┘
                     ▼
             ┌───────────────┐
             │   Métricas    │
             │               │
             │ disponibilidad│
             │ latencia      │
             │ tráfico       │
             │ recursos      │
             │ impacto       │
             └───────────────┘
```


### No construiríamos el emulador desde cero

Usaríamos Mininet/Mininet-Sec como **motor de emulación**, y nuestro trabajo estaría en desarrollar una **capa de simulación/orquestación/evaluación de escenarios de ciberataque** encima.


```
Escenario: Ataque a servidor web

Red:
    ├── atacante
    ├── firewall
    ├── servidor web
    └── base de datos

Objetivo:
    servidor-web

Vulnerabilidad:
    servicio HTTP vulnerable

Ataque:
    reconocimiento
    ↓
    explotación
    ↓
    acceso
    ↓
    impacto

Métricas:
    ├── latencia
    ├── tráfico
    ├── CPU
    ├── memoria
    ├── disponibilidad
    └── servicios afectados
```

Al finalizar:

```
RESULTADO DEL ESCENARIO
────────────────────────

Ataque: SQL Injection
Objetivo: Web Server

Impacto:
    Disponibilidad     15%
    Recursos           32%
    Servicios afectados 2
    Alcance             40%

Nivel de impacto:
    ██████░░░░ 6.2/10

Servicios afectados:
    HTTP
    MySQL

Tiempo hasta compromiso:
    38.4 segundos
```


---

## Idea de división

```
                    SIMULADOR DE CIBERATAQUES
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
          ▼                   ▼                   ▼
     1. SIMULACIÓN       2. MONITOREO        3. RESPUESTA
          │              Y MÉTRICAS          / NEUTRALIZACIÓN
          │                   │                   │
          ▼                   ▼                   ▼
     Ejecuta ataque      Observa sistema     Detecta condición
     Genera tráfico      Recopila eventos     Ejecuta respuesta
     Manipula servicios  Calcula métricas     Mitiga / bloquea
          │                   │                   │
          └───────────────────┼───────────────────┘
                              ▼
                        RESULTADO FINAL
```

## 1. Simulación

Este sería el **motor ofensivo**.

Su responsabilidad sería construir y ejecutar los escenarios:

- Crear la topología.
- Levantar hosts y servicios.
- Configurar atacante, víctimas, servidores, etc.
- Ejecutar ataques controlados.
- Definir secuencias de ataque.
- Generar tráfico legítimo y malicioso.
- Mantener los escenarios reproducibles.

Aquí **Mininet/Mininet-Sec** podría ser fundamental.

```
                 Internet simulado
                       │
                  [ Atacante ]
                       │
                    [ Router ]
                       │
                  [ Firewall ]
                       │
              ┌────────┴────────┐
              │                 │
          [Web Server]      [DNS Server]
              │
          [Database]
```

## 2. Monitoreo + métricas

**Red**

- paquetes enviados/recibidos
- ancho de banda
- latencia
- conexiones activas
- paquetes perdidos

**Sistema**

- CPU
- RAM
- procesos
- carga
- uso de disco

**Servicios**

- HTTP disponible/no disponible
- DNS disponible/no disponible
- SSH disponible/no disponible
- tiempo de respuesta

**Seguridad**

- conexiones sospechosas
- número de eventos
- hosts afectados
- servicios comprometidos
- duración del ataque

Y después convertirlo en métricas:

```
Impacto
│
├── Disponibilidad     20%
├── Rendimiento        35%
├── Recursos           18%
├── Servicios afectados 40%
└── Alcance             25%
```

Podría incluso generar un **índice de impacto**.

Por ejemplo:

```
Impacto =
    0.30 × disponibilidad
  + 0.25 × recursos
  + 0.20 × servicios
  + 0.15 × tráfico
  + 0.10 × alcance
```


## 3. Módulo de detección y respuesta

```
Ataque
  ↓
Monitoreo
  ↓
Detección
  ↓
Clasificación
  ↓
Respuesta
  ↓
Mitigación
  ↓
Evaluación
```

Por ejemplo:

```
Ataque:
Port Scan
      ↓
Monitor:
Detecta múltiples conexiones
      ↓
Detector:
Comportamiento anómalo
      ↓
Respuesta:
Bloquear IP
      ↓
Verificación:
¿Disminuyó el tráfico?
      ↓
Métricas:
Impacto antes vs después
```

```
             ┌──────────────────┐
             │   ESCENARIO      │
             │   DE ATAQUE      │
             └────────┬─────────┘
                      │
                      ▼
             ┌──────────────────┐
             │    SIMULACIÓN    │
             │                  │
             │ Mininet-Sec      │
             └────────┬─────────┘
                      │
                      │ eventos/tráfico
                      ▼
             ┌──────────────────┐
             │    MONITOREO     │
             │                  │
             │ métricas/eventos │
             └────────┬─────────┘
                      │
                      │ detección
                      ▼
             ┌──────────────────┐
             │     RESPUESTA    │
             │                  │
             │ bloquear/aislar  │
             └────────┬─────────┘
                      │
                      │ modificación
                      ▼
             ┌──────────────────┐
             │    SIMULACIÓN    │
             │     continúa     │
             └──────────────────┘
```


| Integrante | Módulo                    | Responsabilidad principal                                 |
| ---------- | ------------------------- | --------------------------------------------------------- |
| 1          | **Simulación**            | Topología, ataques, servicios y escenarios                |
| 2          | **Monitoreo y métricas**  | Telemetría, detección de eventos y evaluación del impacto |
| 3          | **Detección y respuesta** | Identificación de ataques y mecanismos de mitigación      |
