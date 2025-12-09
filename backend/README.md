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
