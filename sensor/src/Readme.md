# 📘 Módulo Sensor – Proyecto: Spectrum Monitoring Platform

## 🛰️ Descripción General

El **Módulo Sensor** es el componente encargado de:

- Capturar el espectro RF mediante **HackRF One**.  
- Procesar la señal para obtener la **PSD (Welch)**.  
- Enviar resultados al servidor central mediante **ZMQ + REST**.  
- Recibir configuraciones remotas desde el Backend (FASTAPI).  
- Registrar métricas internas del sistema (CPU, RAM, disco, tiempos).  

El módulo está compuesto por dos capas:

1. **Orquestador (C Engine)**  
2. **Run Server (Python)**  

---

## 🧩 Arquitectura General

```text
                ┌───────────────────────────┐
                │        Backend FASTAPI     │
                │   (Plataforma central ANE) │
                │       http://.../api/*     │
                └──────────────┬─────────────┘
                               │ REST (JSON)
                               ▼
                      ┌───────────────────────┐
                      │       Run Server       │
                      │        (Python)        │
                      │ - ZMQ pub/sub          │
                      │ - Cliente REST local   │
                      └─────────────┬─────────┘
                                    │ ZMQ (JSON)
                       ▲────────────┘
                       │ publish(data)
                       │ subscribe(acquire)
┌──────────────────────────────────────────────────────────┐
│                 Orquestador (C Engine)                   │
│ - Control HackRF                                          │
│ - Captura IQ                                              │
│ - PSD Welch                                               │
│ - Métricas CPU/RAM/disco                                  │
└──────────────────────────────────────────────────────────┘

```


## ⚙️ Flujo Completo del Sensor

- El Backend FASTAPI solicita una adquisición.  
- El Run Server consulta el endpoint remoto `/configuration/{mac}`.  
- El Backend responde con los parámetros de adquisición PSD.  
- El Run Server envía esa configuración al Orquestador vía **ZMQ topic `"acquire"`**.  
- El Orquestador:  
  - Configura el HackRF  
  - Captura IQ  
  - Calcula la PSD  
  - Publica la PSD por **ZMQ topic `"data"`**  
- El Run Server reenvía la PSD al Backend mediante **POST `/data`**.  
- El Backend almacena y/o visualiza la señal.  

---

# 1. Orquestador (C Engine)

## 1.1 Responsabilidades

- Captura IQ usando HackRF One.  
- Configura el hardware SDR según parámetros remotos.  
- Procesa la señal mediante Welch.  
- Publica la PSD como un JSON por ZMQ.  
- Registra métricas del sistema en CSV.  

---

## 1.2 Recepción de Órdenes (ZMQ topic `"acquire"`)

El Orquestador escucha comandos desde el Run Server:

```c
zsub_init("acquire", handle_psd_message);


Formato del comando recibido:

{
  "center_freq": 98000000,
  "span": 20000000,
  "rbw": 5000,
  "sample_rate": 20000000,
  "overlap": 0.5,
  "window_type": 2,
  "scale": "dBm",
  "lna_gain": 16,
  "vga_gain": 32,
  "amp_enabled": false
}
