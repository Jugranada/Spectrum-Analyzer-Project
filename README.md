Oscar Andres Gutierrez Estepa - Juan Esteban Granada Cardona

# 📡 Spectrum Monitoring Platform  
## 🛰️ Plataforma de Monitoreo Distribuido del Espectro RF 

La **Spectrum Monitoring Platform** es un sistema completo diseñado para la **captura, procesamiento, almacenamiento, visualización y administración del espectro radioeléctrico (RF)** mediante una red distribuida de sensores basados en **HackRF One**.

El proyecto está compuesto por **tres módulos principales**:

1. **Sensor Node** → Captura RF + PSD Welch  
2. **Backend Server** → Gestión central, API, almacenamiento y control  
3. **Frontend Web** → Dashboard en tiempo real

---

# 🧩 Arquitectura General del Sistema

```text
                         ┌─────────────────────────────┐
                         │       UI Web / Frontend      │
                         │   (Dashboard ANE - Vite JS)  │
                         └───────────────┬──────────────┘
                                         │ REST (JSON)
                                         ▼
                   ┌─────────────────────────────────────────────┐
                   │                 Backend FASTAPI              │
                   │ - /configuration/{mac}                      │
                   │ - /data                                     │
                   │ - /sensors                                  │
                   │ - /status                                   │
                   └───────────────┬─────────────────────────────┘
                                   │ REST + ZMQ coordination
                                   ▼
                     ┌────────────────────────────────────┐
                     │            Run Server (Python)       │
                     │ - ZMQ Pub/Sub                       │
                     │ - Interfaz con orquestador          │
                     └─────────────┬───────────────────────┘
                                   │ ZMQ "acquire" / "data"
                                   ▼
                ┌─────────────────────────────────────────────────┐
                │              Orquestador (C Engine)             │
                │  - Control HackRF                              │
                │  - Captura IQ                                  │
                │  - Welch PSD                                   │
                │  - Métricas CPU/RAM                            │
                └─────────────────────────────────────────────────┘


```
# 🎯 Objetivo General de la Plataforma
monitorear de forma distribuida bandas de RF mediante una red de sensores que:

- Capturan señales RF con HackRF
- Procesan PSD 
- Envían las PSD a un servidor 
- Permiten visualización en tiempo real
- Reciben configuraciones remotas
- Registran métricas y estado del sensor

# 🎯 Módulo Sensor (C Engine + Run Server)

## ⚙️ Subcomponentes

## Orquestador (C)

- Control del HackRF

- Captura IQ

- Algoritmo de PSD Welch

- Publicación ZMQ

- Métricas internas

## Run Server (Python)

- Comunicación REST con Backend

- Comunicación ZMQ con el C Engine

- Envía PSDs al Backend

- Recibe configuraciones remotas

## Flujo del Sensor

- Backend entrega configuración a través de GET /configuration/{mac}

- Run Server reenvía la configuración vía ZMQ

- Orquestador captura IQ + calcula PSD

- Orquestador publica PSD → Run Server

- Run Server envía PSD a Backend vía POST /data

# 🎯 Módulo Backend (FASTAPI)

## Funcionalidades

- API para sensores (configuración + recepción de PSDs)

- API para frontend (consulta + estado)

- Almacenamiento (sensors, psd_data, metrics)

- Validación y limpieza de datos

- Actúa como control tower del sistema

 # 🎯Módulo Frontend 
 
## Dashboard web para visualizar:

- PSD en tiempo real

- Estado de los sensores

- Configuración actual

- Métricas CPU/RAM/Uptime

- Histórico de capturas

## Tecnologías

- Vite

- JavaScript Vanilla

- Fetch API

- Canvas 2D para la PSD

## 🔄 Ciclo de actualización

Cada 1 segundo:

setInterval(async () => {
    const psd = await getLatestPSD(selectedMAC);
    plotPSD(canvas, psd.freq, psd.Pxx);
}, 1000);

# 🎯 Flujo Completo del Sistema (Extremo a Extremo)

[Frontend]
      │ GET /sensors
      ▼
[Backend FASTAPI]
      │ GET /configuration/{mac}
      ▼
[Run Server]
      │ ZMQ topic: acquire
      ▼
[Orquestador C]
      │ Captura IQ + PSD Welch
      │ ZMQ topic: data
      ▼
[Run Server]
      │ POST /data
      ▼
[Backend]
      │ Persistencia + estado
      ▼
[Frontend]
      │ GET /data/latest/{mac}
      ▼
   Gráfica PSD (Canvas)

# 🌐 Despliegue

## Backend
cd backend
uvicorn main:app --reload

## Front End
cd frontend
npm install
npm run dev

## Sensor 

- C engine 
make
./orchestrator

- Run server
  python run_server.py

  

