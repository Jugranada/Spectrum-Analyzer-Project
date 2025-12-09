# 📘 Módulo Frontend – Proyecto: Spectrum Monitoring Platform

## 🛰️ Descripción General

El **Módulo Frontend** es la interfaz gráfica principal de la plataforma, encargada de:

- Mostrar PSDs en tiempo real provenientes de los sensores.  
- Visualizar el estado, salud y actividad de cada sensor (CPU, RAM, uptime, errores).  
- Gestionar configuraciones de adquisición para los sensores.  
- Consultar histórico de capturas almacenadas en el Backend.  
- Proveer un dashboard completo para la operación de la red ANE.  

Este módulo está construido con:

- **Vite**
- **JavaScript Vanilla (sin frameworks)**
- **Fetch API para comunicación con Backend**
- **HTML + CSS + JS modular**

---

## 🧩 Arquitectura General

```text
 ┌───────────────────────────────┐
 │          Usuario ANE           │
 │      (Operador / Analista)     │
 └──────────────┬────────────────┘
                │ HTTP (HTTPS)
                ▼
      ┌──────────────────────────────┐
      │         Frontend Vite        │
      │ - JS Vanilla                 │
      │ - Render dinámico            │
      │ - Fetch API                  │
      └──────────────┬──────────────┘
                     │ REST (JSON)
                     ▼
           ┌───────────────────────────────┐
           │          Backend FASTAPI        │
           │ /configuration/{mac}            │
           │ /data                           │
           │ /sensors                        │
           │ /status                         │
           └─────────────────────────────────┘

```
## ⚙️ Flujo Completo del Sensor

El usuario ingresa al Dashboard.

El Frontend consulta al Backend:

- /sensors → lista de sensores

- /status/{mac} → estado

- /data/latest/{mac} → última PSD

Renderiza:

- Tabla de sensores

- Gráfica PSD

- Estado del sensor

- Configuración actual

Si el usuario ajusta configuración:

- Se envía al Backend vía POST /configuration/{mac}

Si el usuario selecciona un sensor:

- Se actualizan en tiempo real los componentes del Dashboard.

---

# 1.Comunicación con el Backend
Toda la comunicación utiliza fetch() con JSON

## 1.1 Obtener lista de sensores

const response = await fetch(`${API_URL}/sensors`);
const sensors = await response.json();

## 1.2 Obtener lista de sensores
const res = await fetch(`${API_URL}/status/${mac}`);
const status = await res.json();

Respuesta tipica:
{
  "mac": "AA:BB:CC:11:22:33",
  "cpu": 18.5,
  "ram": 42.1,
  "uptime": 102233,
  "last_psd": "2025-01-21T12:30:12.120"
}

## 1.3 Obtener última PSD
const res = await fetch(`${API_URL}/data/latest/${mac}`);
const psd = await res.json();

Formato recibido:
{
  "start_freq_hz": 88000000,
  "end_freq_hz": 108000000,
  "bin_count": 4096,
  "Pxx": [-120.5, -118.3, ...]
}

## 1.4 Enviar nueva configuración
await fetch(`${API_URL}/configuration/${mac}`, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify(config)
});

# 2.Componentes Principales del Frontend

## 2.1 Gráfica PSD (canvas)
Renderizado mediante CanvasRenderingContext2D.
export function plotPSD(canvas, freq, pxx) {
    const ctx = canvas.getContext("2d");

    ctx.clearRect(0, 0, canvas.width, canvas.height);
    ctx.beginPath();

    const dx = canvas.width / pxx.length;

    pxx.forEach((value, i) => {
        const x = i * dx;
        const y = canvas.height - ((value + 140) * 2);
        ctx.lineTo(x, y);
    });

    ctx.stroke();
}

## 2.2 Tabla de Sensores

export function renderSensorsTable(container, sensors) {
    container.innerHTML = sensors.map(s => `
        <tr>
            <td>${s.mac}</td>
            <td>${s.location}</td>
            <td>${s.last_seen}</td>
        </tr>
    `).join("");
}

## 2.3 Formulario de Configuración
export function renderConfigForm(config) {
    document.querySelector("#center_freq").value = config.center_freq;
    document.querySelector("#span").value = config.span;
    // ...
}

# 3.Flujo de Actualización Automática
El frontend actualiza en ciclo continuo:
setInterval(async () => {
    const psd = await getLatestPSD(selectedMAC);
    plotPSD(canvas, psd.freq, psd.Pxx);
}, 1000);
Esto permite un pseudo-tiempo-real con un refresco de 1 segundo.

# 4 .Compilación y Ejecución

- Instalar dependencias
  npm install

- Ejecutar en desarrollo
  npm run dev

- Build de producción
  npm run build









