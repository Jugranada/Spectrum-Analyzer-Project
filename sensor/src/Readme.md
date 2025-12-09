# 📘 Módulo Sensor – Proyecto ANE–UNAL: Spectrum Monitoring Platform

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
