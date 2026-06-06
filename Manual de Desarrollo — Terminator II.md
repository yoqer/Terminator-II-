# Manual de Desarrollo — Terminator II

**Versión:** 2.0.0  
**Fecha:** Junio 2026  
**Proyecto:** TenMiNaTor Web — Módulo Terminator II  
**Autor:** Manus AI / Equipo yoqer  

---

## Índice

1. [Visión General](#1-visión-general)
2. [Arquitectura del Sistema](#2-arquitectura-del-sistema)
3. [Requisitos e Instalación](#3-requisitos-e-instalación)
4. [Configuración de Entorno](#4-configuración-de-entorno)
5. [Módulos del Servidor](#5-módulos-del-servidor)
6. [Canales de Comunicación](#6-canales-de-comunicación)
7. [Pipeline Multimodal](#7-pipeline-multimodal)
8. [Interfaz Web](#8-interfaz-web)
9. [Sistema de Claves API](#9-sistema-de-claves-api)
10. [Despliegue en Producción](#10-despliegue-en-producción)
11. [Despliegue Local](#11-despliegue-local)
12. [Tests y Calidad](#12-tests-y-calidad)
13. [Referencia de API](#13-referencia-de-api)
14. [Solución de Problemas](#14-solución-de-problemas)

---

## 1. Visión General

**Terminator II** es un sistema de inteligencia artificial conversacional multimodal y multicanal, construido sobre el framework **TenMiNaTor Web** (React 19 + Express 4 + tRPC 11 + Drizzle ORM). Permite interactuar con modelos de lenguaje locales y en la nube, sintetizar voz, generar vídeo y recibir mensajes desde múltiples canales (web, WhatsApp, Telegram, SMS, llamadas de voz y videollamadas).

El sistema está diseñado con una arquitectura de **fallback en cascada**: si un backend no está disponible, el siguiente de la cadena toma el relevo automáticamente, garantizando respuesta en todo momento.

### Capacidades principales

| Categoría | Backends soportados |
|---|---|
| **LLM** | Ollama (local), LM Studio (local), Grok xAI, OpenAI, Kimi (Moonshot), DeepSeek |
| **TTS** | Kokoro (local, 11 idiomas), Grok Aurora (5 voces), OpenAI TTS, Piper (local, ONNX) |
| **STT** | Grok STT (25 idiomas), OpenAI Whisper |
| **Vídeo** | Grok Video API (texto→vídeo, imagen→vídeo, extensión), Wan2.1 (local, stub) |
| **Canales** | Web (chat + videollamada), WhatsApp Business, Telegram Bot, SMS/MMS Twilio, Voz Twilio ConversationRelay |

---

## 2. Arquitectura del Sistema

```
┌─────────────────────────────────────────────────────────────────┐
│                        TERMINATOR II                            │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │  Web Chat    │  │  Video Call  │  │  External Channels   │  │
│  │  (React SPA) │  │  (WebRTC)    │  │  WA / TG / SMS / Voz │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘  │
│         │                 │                      │              │
│         └─────────────────┴──────────────────────┘             │
│                           │                                     │
│                    ┌──────▼──────┐                              │
│                    │  tRPC API   │  ← /api/trpc                 │
│                    │  + Webhooks │  ← /webhook/*                │
│                    └──────┬──────┘                              │
│                           │                                     │
│                    ┌──────▼──────────────────────────────────┐  │
│                    │         ORCHESTRATOR                    │  │
│                    │  1. Load history (DB)                   │  │
│                    │  2. Call LLM → text + videoP + audioP   │  │
│                    │  3. Parallel: TTS + Video generation    │  │
│                    │  4. Save messages (DB)                  │  │
│                    │  5. Return TerminatorResponse           │  │
│                    └──────┬──────────────────────────────────┘  │
│                           │                                     │
│         ┌─────────────────┼─────────────────┐                  │
│         │                 │                 │                   │
│  ┌──────▼──────┐  ┌───────▼──────┐  ┌──────▼──────┐           │
│  │  LLM Router │  │  TTS Router  │  │ Video Router│           │
│  │  llm.ts     │  │  tts.ts      │  │  video.ts   │           │
│  └─────────────┘  └──────────────┘  └─────────────┘           │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Storage: MySQL/TiDB (Drizzle) + S3 (archivos de audio)  │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Estructura de archivos del módulo Terminator II

```
server/terminator/
├── orchestrator.ts    ← Coordinador principal del pipeline
├── llm.ts             ← Router LLM con fallback en cascada
├── tts.ts             ← Router TTS/STT (Kokoro → Grok → OpenAI → Piper)
├── video.ts           ← Router de generación de vídeo
├── channels.ts        ← Parsers y senders de canales externos
├── webhooks.ts        ← Rutas Express para webhooks y upload de audio
├── router.ts          ← Procedimientos tRPC del Terminator II
└── db.ts              ← Helpers de base de datos

client/src/
├── pages/
│   ├── TerminatorChat.tsx      ← Chat web principal
│   ├── TerminatorConfig.tsx    ← Configuración de claves y preferencias
│   └── TerminatorVideoCall.tsx ← Videollamada WebRTC
├── components/
│   └── ApiKeyManager.tsx       ← Gestor de claves API reutilizable
└── hooks/
    └── useApiKeys.ts           ← Hook de gestión de claves (localStorage + BD)
```

---

## 3. Requisitos e Instalación

### Requisitos del sistema

| Componente | Mínimo | Recomendado |
|---|---|---|
| Node.js | 18.x | 22.x LTS |
| pnpm | 8.x | 10.x |
| MySQL / TiDB | 8.0 | TiDB Serverless |
| RAM | 512 MB | 2 GB |
| Almacenamiento | 1 GB | 10 GB |

### Instalación

```bash
# 1. Clonar el repositorio
git clone https://github.com/yoqer/tenminator-web.git
cd tenminator-web

# 2. Instalar dependencias
pnpm install

# 3. Configurar variables de entorno (ver sección 4)
cp .env.example .env
# Editar .env con tus credenciales

# 4. Migrar la base de datos
pnpm db:push

# 5. Iniciar en desarrollo
pnpm dev

# 6. Compilar para producción
pnpm build
pnpm start
```

### Servicios locales opcionales

Para usar backends locales de TTS y LLM, instala los siguientes servicios:

**Kokoro TTS (Docker):**
```bash
docker run -d -p 8880:8880 ghcr.io/remsky/kokoro-fastapi-cpu:latest
# Con GPU:
docker run -d -p 8880:8880 --gpus all ghcr.io/remsky/kokoro-fastapi-gpu:latest
```

**Ollama:**
```bash
curl -fsSL https://ollama.ai/install.sh | sh
ollama pull salamandra:7b   # Español nativo (BSC)
ollama pull qwen3:8b        # Multilingüe
ollama pull llama3.2:3b     # Ligero
```

**LM Studio:**  
Descargar desde [lmstudio.ai](https://lmstudio.ai) e iniciar el servidor local en el puerto 1234.

---

## 4. Configuración de Entorno

Las variables de entorno se gestionan de dos formas:

1. **Variables de sistema** (`.env`): para credenciales de infraestructura y fallback global.
2. **Claves por usuario** (BD + localStorage): para que cada usuario configure sus propias APIs desde la interfaz web en `/terminator/config`.

### Variables de entorno del servidor

```env
# Base de datos (obligatorio)
DATABASE_URL=mysql://user:pass@host:4000/dbname

# JWT y OAuth (obligatorio)
JWT_SECRET=tu_secreto_jwt
VITE_APP_ID=tu_app_id_manus

# Fallback global de APIs (opcional — los usuarios pueden configurar las suyas)
XAI_API_KEY=xai-...           # Grok (LLM + TTS + Vídeo)
OPENAI_API_KEY=sk-...         # OpenAI (LLM + TTS + Whisper)
MOONSHOT_API_KEY=sk-...       # Kimi (LLM)
DEEPSEEK_API_KEY=sk-...       # DeepSeek (LLM)

# Twilio (para SMS, voz y videollamada)
TWILIO_ACCOUNT_SID=AC...
TWILIO_AUTH_TOKEN=...
TWILIO_PHONE_NUMBER=+1...
TWILIO_API_KEY_SID=SK...      # Para Twilio Video JWT
TWILIO_API_KEY_SECRET=...

# WhatsApp Business API
WHATSAPP_TOKEN=...
WHATSAPP_PHONE_NUMBER_ID=...
WHATSAPP_VERIFY_TOKEN=terminator_verify_2026

# Telegram Bot
TELEGRAM_BOT_TOKEN=...

# Servicios locales (opcional)
OLLAMA_URL=http://localhost:11434
LMSTUDIO_URL=http://localhost:1234
KOKORO_URL=http://localhost:8880
PIPER_URL=http://localhost:5000
```

---

## 5. Módulos del Servidor

### 5.1 Orquestador (`orchestrator.ts`)

El orquestador es el núcleo del sistema. Recibe un `TerminatorMessage` y ejecuta el pipeline completo:

```typescript
interface TerminatorMessage {
  channelId: string;    // ID único de la conversación
  channel: ChannelType; // "web" | "whatsapp" | "telegram" | "sms" | "voice"
  text?: string;
  audioUrl?: string;
  imageUrl?: string;
  language?: string;
}

interface TerminatorResponse {
  text: string;
  audioUrl: string | null;
  videoUrl: string | null;
  videoPrompt: string | null;
  audioPrompt: string | null;
  backend: { llm: string; tts: string | null; video: string | null };
  conversationId: number | null;
}
```

La configuración se construye con **triple fallback**: config del usuario en BD → variable de entorno → valor por defecto.

### 5.2 Router LLM (`llm.ts`)

Cadena de fallback: **Ollama → LM Studio → Grok → OpenAI → Kimi → DeepSeek**

El LLM recibe instrucciones de sistema para generar respuestas multimodales estructuradas:

```
"Para Generar Video quiero: [descripción visual]"
"Para Generar Audio quiero: [descripción de audio/voz]"
```

El parser extrae estas secciones con expresiones regulares y las pasa al generador de vídeo y TTS respectivamente.

**Modelos preferidos por hardware (Ollama):**

| VRAM/RAM | Modelo recomendado | Idioma |
|---|---|---|
| < 4 GB | `phi4-mini:3.8b` | Multilingüe |
| 4–8 GB | `qwen3:8b` | Multilingüe |
| 8–16 GB | `salamandra:7b` | Español nativo |
| > 16 GB | `alia:40b` | Español nativo (BSC) |

### 5.3 Router TTS (`tts.ts`)

Cadena de fallback: **Kokoro → Grok Aurora → OpenAI TTS → Piper**

**Mapa de idiomas Kokoro:**

| Código | Idioma | Prefijo Kokoro |
|---|---|---|
| `es` | Español | `e` |
| `en` | Inglés (US) | `a` |
| `en-gb` | Inglés (UK) | `b` |
| `de` | Alemán | `d` |
| `fr` | Francés | `f` |
| `hi` | Hindi | `h` |
| `it` | Italiano | `i` |
| `ja` | Japonés | `j` |
| `ko` | Coreano | `k` |
| `pt` | Portugués (BR) | `p` |
| `zh` | Chino mandarín | `z` |

**Mapa de voces por personaje:**

| Personaje | Voz Kokoro | Voz Grok |
|---|---|---|
| `default` | `af_heart` | `eve` |
| `ani` | `af_sky` | `ara` |
| `valentin` | `am_adam` | `rex` |

### 5.4 Router de Vídeo (`video.ts`)

Actualmente soporta **Grok Video API** (producción) y detección de backends locales (Wan2.1, LTX-Video) como stubs preparados para integración futura.

El flujo de Grok Video:
1. Enviar prompt a `POST https://api.x.ai/v1/video/generations`
2. Polling del estado hasta `succeeded`
3. Descargar el vídeo generado
4. Subir a S3 y devolver URL pública

---

## 6. Canales de Comunicación

### 6.1 WhatsApp Business API

**Configuración del webhook:**
```
URL: https://tu-dominio.com/webhook/whatsapp
Método: POST (mensajes) + GET (verificación)
Token de verificación: valor de WHATSAPP_VERIFY_TOKEN
```

El sistema soporta mensajes de texto, audio (OGG/Opus → transcripción STT) e imágenes.

### 6.2 Telegram Bot

**Configuración del webhook:**
```bash
curl -X POST "https://api.telegram.org/bot{TOKEN}/setWebhook" \
  -d "url=https://tu-dominio.com/webhook/telegram"
```

### 6.3 SMS/MMS Twilio

**Configuración en Twilio Console:**
```
Webhook URL: https://tu-dominio.com/webhook/sms
Método: POST
```

Los MMS (imágenes) se pasan como `imageUrl` al orquestador.

### 6.4 Voz Twilio ConversationRelay

El sistema implementa el protocolo ConversationRelay de Twilio para llamadas de voz con IA en tiempo real:

1. Twilio llama a `POST /twiml` → el servidor responde con TwiML que conecta al WebSocket
2. El WebSocket en `/ws/voice` recibe transcripciones en tiempo real
3. El servidor llama al LLM y devuelve texto que Twilio sintetiza con su TTS

**Configuración en Twilio Console:**
```
Voice webhook: https://tu-dominio.com/twiml
```

### 6.5 Videollamada WebRTC (Twilio Video)

La página `/terminator/video-call` permite:
- Iniciar una sala Twilio Video con cámara y micrófono locales
- Grabar audio en chunks de 5 segundos
- Subir cada chunk a S3 via `/api/terminator/upload-audio`
- Transcribir con STT y enviar al orquestador
- Reproducir la respuesta de audio y vídeo generado

**Requisitos:** `TWILIO_ACCOUNT_SID`, `TWILIO_API_KEY_SID`, `TWILIO_API_KEY_SECRET`

---

## 7. Pipeline Multimodal

El pipeline multimodal es la característica central del Terminator II. Cuando el usuario activa la generación de vídeo, el LLM recibe instrucciones para estructurar su respuesta en tres partes:

```
[Respuesta de texto normal al usuario]

"Para Generar Video quiero: [descripción visual detallada del escenario, personaje, acción]"
"Para Generar Audio quiero: [descripción del audio, voz, música, efectos]"
```

El orquestador parsea estas secciones y lanza **en paralelo**:
- `synthesizeSpeech(audioPrompt, ttsConfig, language, personaId)` → URL de audio en S3
- `generateVideo(videoPrompt, videoConfig)` → URL de vídeo en S3

Ambas operaciones son independientes: si una falla, la otra continúa y el sistema responde con lo que tenga disponible.

---

## 8. Interfaz Web

### Rutas disponibles

| Ruta | Componente | Descripción |
|---|---|---|
| `/` | `Home.tsx` | Página principal con acceso al Terminator II |
| `/terminator` | `TerminatorChat.tsx` | Chat multimodal principal |
| `/terminator/config` | `TerminatorConfig.tsx` | Configuración de claves y preferencias |
| `/terminator/video-call` | `TerminatorVideoCall.tsx` | Videollamada WebRTC |

### TerminatorChat

La página de chat incluye:
- **Sidebar** de selección de modelos (LLM, TTS, vídeo, personaje, idioma)
- **Burbujas de mensaje** con soporte para texto, audio inline (`<audio>`) y vídeo inline (`<video>`)
- **Grabación de voz** desde el navegador con `MediaRecorder`
- **Toggle de generación de vídeo** (desactivado por defecto para mayor velocidad)

### TerminatorConfig

Organizado en tres pestañas:
1. **Claves API** — Gestión de todas las claves con `ApiKeyManager` (grupos colapsables, show/hide, badge de estado servidor)
2. **Preferencias** — Backend por defecto, personaje, idioma, TTS, vídeo
3. **Webhooks** — Guía de configuración de WhatsApp, Telegram, SMS y Twilio

---

## 9. Sistema de Claves API

Las claves API se almacenan en **tres capas**:

```
Prioridad 1: BD del usuario (persistente entre dispositivos)
Prioridad 2: localStorage del navegador (caché local inmediata)
Prioridad 3: Variables de entorno del servidor (fallback global)
```

El hook `useApiKeys` gestiona la sincronización entre capas:

```typescript
const {
  keys,           // Claves actuales (localStorage)
  serverPresence, // Qué claves están guardadas en BD (sin revelar valores)
  updateKey,      // Actualizar una clave en localStorage
  saveToServer,   // Sincronizar localStorage → BD
  clearAll,       // Borrar localStorage y BD
  hasAnyKey,      // Boolean: ¿hay alguna clave configurada?
} = useApiKeys();
```

Las claves **nunca se devuelven en texto plano** desde el servidor: `getConfig` devuelve únicamente booleanos de presencia (`xaiApiKey: true/false`).

---

## 10. Despliegue en Producción (Hosting Web)

### Opción A: Manus Hosting (recomendado)

1. Crear checkpoint en el panel de Manus
2. Hacer clic en **Publish** en la cabecera del panel de gestión
3. Configurar dominio personalizado en **Settings → Domains**

### Opción B: Railway / Render / Fly.io

```bash
# Compilar
pnpm build

# Variables de entorno requeridas en la plataforma:
DATABASE_URL, JWT_SECRET, VITE_APP_ID, OAUTH_SERVER_URL, VITE_OAUTH_PORTAL_URL
# + claves opcionales: XAI_API_KEY, OPENAI_API_KEY, TWILIO_*, WHATSAPP_*, TELEGRAM_*

# Comando de inicio
node dist/server/_core/index.js
```

### Opción C: Docker

```dockerfile
FROM node:22-alpine
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN npm install -g pnpm && pnpm install --frozen-lockfile
COPY . .
RUN pnpm build
EXPOSE 3000
CMD ["node", "dist/server/_core/index.js"]
```

```bash
docker build -t terminator-ii .
docker run -p 3000:3000 --env-file .env terminator-ii
```

---

## 11. Despliegue Local

```bash
# Desarrollo
pnpm dev
# → http://localhost:3000

# Producción local
pnpm build && pnpm start
# → http://localhost:3000

# Con ngrok para webhooks externos (WhatsApp, Telegram)
ngrok http 3000
# Usar la URL de ngrok como base para los webhooks
```

### Servicios locales recomendados para uso offline total

```bash
# 1. Kokoro TTS (voz local)
docker run -d -p 8880:8880 ghcr.io/remsky/kokoro-fastapi-cpu:latest

# 2. Ollama (LLM local)
ollama serve
ollama pull salamandra:7b

# 3. (Opcional) Piper TTS para idiomas adicionales
# Descargar modelos desde https://github.com/rhasspy/piper/releases
```

Con estos tres servicios, el Terminator II funciona **completamente offline** sin necesidad de ninguna clave API externa.

---

## 12. Tests y Calidad

```bash
# Ejecutar todos los tests
pnpm test

# Tests con cobertura
pnpm test --coverage

# Verificación TypeScript
npx tsc --noEmit

# Lint
pnpm lint
```

### Suites de tests actuales

| Suite | Tests | Cobertura |
|---|---|---|
| `auth.logout.test.ts` | 1 | Autenticación OAuth |
| `training.test.ts` | 4 | Sesiones de entrenamiento y uploads |
| `terminator.test.ts` | 10 | Parser multimodal, orquestador, canales, backends |

---

## 13. Referencia de API

### tRPC Procedures

| Procedimiento | Tipo | Descripción |
|---|---|---|
| `terminator.chat` | `publicMutation` | Enviar mensaje y recibir respuesta multimodal |
| `terminator.getHistory` | `publicQuery` | Obtener historial de conversación |
| `terminator.getConversations` | `publicQuery` | Listar todas las conversaciones |
| `terminator.getConfig` | `protectedQuery` | Obtener configuración del usuario (claves enmascaradas) |
| `terminator.saveConfig` | `protectedMutation` | Guardar configuración y claves API |
| `terminator.checkBackends` | `publicQuery` | Verificar disponibilidad de backends |
| `terminator.getVideoToken` | `protectedMutation` | Obtener token JWT para sala Twilio Video |

### REST Endpoints

| Endpoint | Método | Descripción |
|---|---|---|
| `/webhook/whatsapp` | GET | Verificación del webhook de WhatsApp |
| `/webhook/whatsapp` | POST | Recibir mensajes de WhatsApp |
| `/webhook/telegram` | POST | Recibir mensajes de Telegram |
| `/webhook/sms` | POST | Recibir SMS/MMS de Twilio |
| `/twiml` | POST | TwiML para llamadas de voz Twilio |
| `/ws/voice` | WebSocket | ConversationRelay en tiempo real |
| `/api/terminator/upload-audio` | POST | Subir audio para STT (multipart/form-data) |

---

## 14. Solución de Problemas

### El chat no responde

1. Verificar que al menos un backend LLM está disponible: ir a `/terminator/config` → pestaña **Claves API** → botón **Verificar backends**
2. Si se usa Ollama local, comprobar que el servicio está activo: `curl http://localhost:11434/v1/models`
3. Si se usan APIs en la nube, verificar que las claves están configuradas correctamente

### El TTS no genera audio

1. Kokoro local: verificar que el contenedor Docker está corriendo en el puerto 8880
2. Grok TTS: verificar que `XAI_API_KEY` está configurada
3. El sistema siempre devuelve texto aunque el TTS falle

### Los webhooks de WhatsApp no funcionan

1. La URL del servidor debe ser HTTPS accesible públicamente
2. El token de verificación debe coincidir con `WHATSAPP_VERIFY_TOKEN`
3. En desarrollo local, usar ngrok: `ngrok http 3000`

### La videollamada no conecta

1. Verificar que `TWILIO_API_KEY_SID` y `TWILIO_API_KEY_SECRET` están configurados (son diferentes de `TWILIO_AUTH_TOKEN`)
2. Crear las API Keys en: Twilio Console → Account → API Keys & Tokens

---

*Manual generado por Manus AI — Junio 2026*
