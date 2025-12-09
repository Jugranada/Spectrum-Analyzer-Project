🛰️ Módulo Sensor – Documentación del Sistema de Adquisición y Comunicación

Este módulo implementa el subsistema de adquisición, procesamiento y transmisión de datos espectrales para las unidades remotas del proyecto ANE–UNAL Spectrum Monitoring.

El módulo consiste en dos componentes principales:

Orquestador (C Engine)

Control directo del hardware SDR (HackRF One).

Captura IQ.

Cálculo de PSD (Welch).

Publicación de resultados por ZMQ.

Registro de métricas del sistema.

Run Server (Python)

Recibe las PSD del orquestador vía ZMQ.

Expone una API interna al Backend FASTAPI.

Envía comandos de adquisición al motor C.

Integra el sensor con la plataforma central.

🧩 Arquitectura General del Módulo Sensor
                ┌───────────────────────────┐
                │     Backend FASTAPI       │
                │ (Orquestación central ANE)│
                │      http://.../api/*     │
                └───────────────┬───────────┘
                                │ REST (JSON)
                                ▼
                     ┌───────────────────────┐
                     │      Run Server        │
                     │      (Python)          │
                     │  - ZMQ publisher/sub   │
                     │  - Cliente REST local  │
                     └─────────────┬─────────┘
                                   │ ZMQ (JSON)
             ▲─────────────────────┘
             │ publish(data)          subscribe(acquire)
             │
┌─────────────────────────────────────────────────────────┐
│                    Orquestador (C Engine)               │
│  - Control HackRF                                        │
│  - Captura IQ                                            │
│  - PSD Welch                                             │
│  - Métricas CPU / RAM / disco                            │
└──────────────────────────────────────────────────────────┘

⚙️ 1. Flujo de Adquisición

El flujo interno para generar una PSD consiste en:

1.1 Recepción de comando PSD

El Orquestador C se mantiene escuchando un mensaje ZMQ:

zsub_init("acquire", handle_psd_message);


El Run Server envía un JSON de configuración:

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


El callback handle_psd_message:

Parsea el JSON.

Construye desired_config.

Calcula parámetros para:

HackRF (hack_cfg)

PSD Welch (psd_cfg)

Ring buffer (rb_cfg)

Activa config_received = true.

1.2 Configuración del HackRF

Cuando config_received == true, el motor:

hackrf_apply_cfg(device, &hack_cfg);


Esto ajusta:

Frecuencia central.

Sample rate.

LNA / VGA.

AMP.

1.3 Captura IQ

Se inicializa un ring buffer.

Se inicia la captura con:

hackrf_start_rx(device, rx_callback, NULL);


El callback rx_callback llena el ring buffer hasta acumular suficientes bytes (rb_cfg.total_bytes).

1.4 Cálculo de PSD (Welch)

Una vez capturado el IQ:

execute_welch_psd(sig, &psd_cfg, freq, psd);
scale_psd(psd, nperseg, desired.scale);


La salida es:

freq[] (vector de frecuencias relativas)

psd[] (vectores PSD en dBm/dBFS/dBmHz)

1.5 Publicación del resultado

El Orquestador envía la PSD al Run Server por ZMQ:

publish_results(freq, psd, length);


Formato del JSON enviado:

{
  "start_freq_hz": 88000000,
  "end_freq_hz": 108000000,
  "bin_count": 4096,
  "Pxx": [ -120.1, -119.5, -95.2, ... ]
}

1.6 Registro de métricas del sistema

Cada ciclo escribe datos en:

CSV_metrics_psdSDRService/<timestamp>_<MAC>.csv


Incluye:

Tiempos de adquisición y DSP.

CPU %.

RAM.

SWAP.

Disco %.

Parámetros PSD.

Tamaño de PSD.

🔌 2. Comunicación Orquestador ↔ Run Server (ZMQ)

La comunicación entre ambos componentes es:

Dirección	Topic ZMQ	Formato	Descripción
Run Server → Orquestador	"acquire"	JSON	Comando con parámetros de PSD
Orquestador → Run Server	"data"	JSON	Resultado del espectro (PSD)
2.1 Mensaje de entrada al Orquestador (acquire)

Ejemplo:

{
  "center_freq": 915000000,
  "span": 5000000,
  "rbw": 5000,
  "sample_rate": 2000000,
  "overlap": 0.5,
  "window_type": 1,
  "scale": "dBm",
  "lna_gain": 16,
  "vga_gain": 28,
  "amp_enabled": false
}

2.2 Mensaje publicado por el Orquestador (data)
{
  "start_freq_hz": 912500000,
  "end_freq_hz": 917500000,
  "bin_count": 4096,
  "Pxx": [ -120.1, -119.3, ... ]
}


Este JSON es recibido por el Run Server, que lo empaqueta y lo envía al Backend FASTAPI.

🌐 3. Comunicación Run Server ↔ Backend FASTAPI

El Run Server actúa como puente entre el motor C y el backend central.

El Backend espera:

Comandos de inicio de adquisición.

Datos PSD para graficar.

Estado del sensor.

3.1 Endpoints REST que usa el Run Server
POST /data

El Run Server envía al Backend la PSD completa:

{
  "start_freq_hz": 88000000,
  "end_freq_hz": 108000000,
  "center_freq_hz": 98000000,
  "timestamp": "2025-01-21T12:30:12.120",
  "Pxx": [-120.5, -119.0, ...]
}


El backend responde:

{ "status": "ok" }

GET /configuration/{mac}

El Backend envía comandos al sensor:

Respuesta típica:

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


El Run Server toma este JSON y lo reenvía al Orquestador por ZMQ topic "acquire".

📡 4. Flujo Completo: Backend → Sensor
1. FASTAPI recibe comando del operador ANE
2. FASTAPI → (GET /configuration/{mac}) → Run Server
3. Run Server → manda JSON por ZMQ topic "acquire"
4. Orquestador C:
      - Configura HackRF
      - Captura IQ
      - Calcula PSD
      - Publica resultado por ZMQ topic "data"
5. Run Server:
      - Recibe PSD
      - Envía PSD al backend vía POST /data
6. FASTAPI almacena / visualiza la PSD
