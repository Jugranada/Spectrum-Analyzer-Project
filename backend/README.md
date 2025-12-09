# 📘 Módulo Backend – Proyecto: Spectrum Monitoring Platform

## 🛰️ Descripción General

El **Módulo Backend** es la capa central de la plataforma. Su función es coordinar, almacenar, procesar y servir toda la información proveniente de los sensores distribuidos.

Este módulo está construido con **FASTAPI** y provee:

- API REST para comunicación con los sensores (Run Server).
- API REST para visualización, consulta y configuración desde UI web.
- Gestión de configuraciones remotas para cada sensor.
- Recepción y almacenamiento de PSDs publicadas por los sensores.
- Registro de eventos, métricas y estados.
- Servidor central que integra la red completa de sensores ANE.

---

## 🧩 Arquitectura General

```text
 ┌───────────────────────────────┐
 │         UI Web / Cliente       │
 │       (Dashboard ANE)          │
 └──────────────┬────────────────┘
                │ REST (JSON)
                ▼
      ┌──────────────────────────────┐
      │        Backend FASTAPI       │
      │  - /configuration/{mac}      │
      │  - /data                     │
      │  - /status                   │
      │  - /sensors                  │
      └──────────────┬──────────────┘
                     │ REST (JSON)
                     ▼
           ┌──────────────────────┐
           │      Run Server      │
           │       (Python)       │
           └───────────┬─────────┘
                       │ ZMQ "acquire" / "data"
                       ▼
           ┌──────────────────────┐
           │   Orquestador (C)    │
           │ Captura + PSD Welch  │
           └──────────────────────┘

```
## ⚙️ Flujo Completo del Backend

Un sensor solicita su configuración actual:

GET /configuration/{mac}

El Backend consulta la base de datos:

Frecuencia central

Span

Ganancias

RBW

Escala, ventana, overlap

Estado del sensor

Devuelve la configuración al Run Server.

El Run Server envía esa configuración al Orquestador para iniciar la captura.

El Orquestador procesa la PSD y la devuelve vía ZMQ.

El Run Server publica la PSD al Backend:

POST /data

El Backend:

Valida el contenido.

Almacena PSD, timestamp, MAC, parámetros.

Notifica al dashboard si aplica.

---

# 1. Endpoints Principales

## 1.1 GET /configuration/{mac}
El sensor consulta su configuración actual.

Ejemplo de respuesta:

{
  "center_freq": 91500000,
  "span": 10000000,
  "rbw": 5000,
  "sample_rate": 2000000,
  "overlap": 0.5,
  "window_type": 1,
  "scale": "dBm",
  "lna_gain": 16,
  "vga_gain": 32,
  "amp_enabled": false
}

Internamente:

Se carga el perfil del sensor según MAC.

Se aplican políticas por región, canal o UI.

Se validan parámetros antes de enviar.

## 1.2 POST /data
El Run Server envía una PSD procesada por el Orquestador.

Ejemplo del JSON recibido:

{
  "start_freq_hz": 88000000,
  "end_freq_hz": 108000000,
  "center_freq_hz": 98000000,
  "timestamp": "2025-01-21T12:30:12.120",
  "Pxx": [-120.5, -119.0, -110.2],
  "mac": "AA:BB:CC:11:22:33"
}

El Backend realiza:

Validación de tamaños y rangos.

Registro de la muestra en base de datos.

Actualización de estado del sensor.

Envío de evento a dashboard (si aplica).

## 1.3 GET /sensors
Devuelve la lista de sensores registrados + estado actual.

[
  {
    "mac": "AA:BB:CC:11:22:33",
    "last_seen": "2025-01-21T12:30:12.120",
    "status": "online",
    "last_center_freq": 98000000
  }
]

## 1.4 POST /status/{mac}
El Run Server envía métricas internas:
{
  "cpu": 23.1,
  "ram": 58.7,
  "disk": 70.2,
  "uptime": 8123
}
El Backend usa esta info para:

Detectar sobrecargas.

Alertar fallos.

Actualizar panel de monitoreo.

# 2. Modelos (Pydantic)

## 2.1 Modelo de Configuración

class ConfigModel(BaseModel):
    center_freq: int
    span: int
    rbw: int
    sample_rate: int
    overlap: float
    window_type: int
    scale: str
    lna_gain: int
    vga_gain: int
    amp_enabled: bool
    
## 2.2 Modelo de Configuración

class PSDModel(BaseModel):
    start_freq_hz: int
    end_freq_hz: int
    center_freq_hz: int
    timestamp: datetime
    Pxx: List[float]
    mac: str

# 3. Flujo de Datos Backend ↔ Sensor

Backend genera configuración según MAC.

Run Server la solicita por GET.

Run Server la envía al Orquestador.

Orquestador captura, procesa y publica PSD.

Run Server envía PSD al Backend.

Backend almacena y actualiza estado.

UI visualiza la PSD o historial.

# 4. Base de Datos

El Backend almacena:

Tabla sensors

mac

ubicación (opcional)

estado

última configuración

Tabla configurations

parámetros PSD

timestamp

sensor_mac

Tabla psd_data

sensor_mac

start_freq

end_freq

Pxx[]

timestamp

Tabla metrics

cpu

ram

uptime

timestamp

sensor_mac

---
