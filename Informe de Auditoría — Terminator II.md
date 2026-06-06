# Informe de Auditoría — Terminator II

**Versión auditada:** 2.0.0 (checkpoint `5ebc4d15`)  
**Fecha:** Junio 2026  
**Auditor:** Manus AI  
**Proyecto:** tenminator-web — Módulo Terminator II  

---

## Resumen Ejecutivo

La auditoría del módulo Terminator II identificó **5 bugs críticos** y **3 áreas de mejora** en la versión 1.0 del sistema. Todos los bugs críticos fueron corregidos en la versión 2.0.0. El sistema supera la compilación TypeScript sin errores y los 15 tests de la suite pasan correctamente.

| Categoría | Encontrados | Corregidos | Pendientes |
|---|---|---|---|
| Bugs críticos | 5 | 5 | 0 |
| Mejoras de funcionalidad | 3 | 3 | 0 |
| Deuda técnica | 4 | 1 | 3 |
| **Total** | **12** | **9** | **3** |

---

## 1. Bugs Críticos Corregidos

### BUG-001: AccessToken Twilio Video con constructor incorrecto

**Severidad:** Crítica  
**Archivo:** `server/terminator/router.ts` (procedimiento `getVideoToken`)  
**Descripción:** El constructor de `AccessToken` de Twilio se llamaba con `authToken` como tercer parámetro, cuando la API de Twilio Video requiere `apiKeySid` y `apiKeySecret` (credenciales de API Key, distintas del Auth Token de la cuenta).

```typescript
// ❌ Antes (incorrecto — el JWT no era válido para Twilio Video)
new AccessToken(accountSid, authToken, { identity, ttl })

// ✅ Después (correcto)
new AccessToken(accountSid, apiKeySid, apiKeySecret, { identity, ttl })
```

**Impacto:** Las videollamadas WebRTC fallaban al conectar con la sala Twilio Video porque el token JWT generado era inválido.  
**Corrección:** Añadir `TWILIO_API_KEY_SID` y `TWILIO_API_KEY_SECRET` como campos separados en el schema de BD y en el constructor del AccessToken.

---

### BUG-002: `data:` URL de audio rechazada por validación del router

**Severidad:** Crítica  
**Archivo:** `client/src/pages/TerminatorVideoCall.tsx` + `server/terminator/router.ts`  
**Descripción:** La página de videollamada convertía el audio grabado a un `data:audio/webm;base64,...` URL y lo enviaba al procedimiento `terminator.chat`. Sin embargo, el router valida `audioUrl` con `z.string().url()`, que rechaza los `data:` URLs por no ser URLs HTTP válidas.

```typescript
// ❌ Antes (data: URL rechazada por zod)
const dataUrl = `data:audio/webm;base64,${base64}`;
await chatMutation.mutateAsync({ audioUrl: dataUrl });

// ✅ Después (upload a S3, URL real)
const uploadRes = await fetch("/api/terminator/upload-audio", { method: "POST", body: formData });
const { url: audioUrl } = await uploadRes.json();
await chatMutation.mutateAsync({ audioUrl });
```

**Impacto:** La transcripción de voz en videollamadas fallaba silenciosamente; el sistema respondía sin procesar el audio.  
**Corrección:** Crear el endpoint `POST /api/terminator/upload-audio` (con multer, límite 16 MB) que sube el audio a S3 y devuelve una URL real. La videollamada ahora sube primero y luego envía la URL.

---

### BUG-003: `kokoroUrl` ausente en el schema de BD y el orquestador

**Severidad:** Alta  
**Archivos:** `drizzle/schema.ts`, `server/terminator/orchestrator.ts`, `server/terminator/tts.ts`  
**Descripción:** El módulo TTS (`tts.ts`) fue diseñado con `kokoroUrl` en la interfaz `TTSConfig`, pero el campo no existía en la tabla `terminator_config` de la BD ni en la función `buildTTSConfig` del orquestador. Kokoro siempre usaba `http://localhost:8880` como URL fija, ignorando cualquier configuración del usuario.

**Impacto:** Los usuarios no podían configurar una URL personalizada para Kokoro (por ejemplo, un servidor Kokoro remoto o en un puerto diferente).  
**Corrección:** Añadir `kokoroUrl` y `piperUrl` al schema de Drizzle, migrar la BD, y actualizar `buildTTSConfig` para leer el valor con triple fallback (BD → env `KOKORO_URL` → `http://localhost:8880`).

---

### BUG-004: STT en videollamada requería sesión autenticada

**Severidad:** Alta  
**Archivo:** `server/terminator/router.ts` (línea 68)  
**Descripción:** La transcripción de audio (STT) en el procedimiento `chat` solo se ejecutaba si existía una configuración de usuario en BD (`if (!text && input.audioUrl && cfg)`). Los usuarios no autenticados o sin configuración guardada no podían usar STT aunque tuvieran claves en localStorage.

**Impacto:** La funcionalidad de voz en la videollamada (que usa `publicProcedure`) no funcionaba para usuarios sin sesión.  
**Corrección:** El endpoint de upload de audio es independiente del sistema de autenticación. El audio se sube a S3 y se transcribe usando las claves de entorno como fallback cuando no hay configuración de usuario.

---

### BUG-005: Imports de `multer` no disponibles en runtime

**Severidad:** Alta  
**Archivo:** `server/terminator/webhooks.ts`  
**Descripción:** Al añadir el endpoint de upload de audio, `multer` se importaba pero el paquete no estaba instalado en las dependencias del proyecto, causando un error `ERR_MODULE_NOT_FOUND` en runtime.

**Impacto:** El servidor crasheaba al iniciar después de añadir el endpoint de upload.  
**Corrección:** Instalar `multer` y `@types/multer` con `pnpm add multer @types/multer` y reiniciar el servidor.

---

## 2. Mejoras de Funcionalidad Implementadas

### MEJORA-001: Kokoro TTS local con 11 idiomas

**Descripción:** Se integró Kokoro como el primer backend TTS en la cadena de fallback, con soporte completo para 11 idiomas y un mapa de voces por personaje del Terminator.

**Implementación:**
- Función `kokoroTTS()` en `tts.ts` que llama al servidor FastAPI de Kokoro en el puerto 8880
- Mapa `KOKORO_LANG_PREFIX` con los 11 idiomas soportados
- Mapa `KOKORO_PERSONA_VOICE` que asigna voz por personaje y género
- Función `checkKokoro()` expuesta en el router para verificar disponibilidad
- Campo `kokoroUrl` añadido al schema de BD y al componente `ApiKeyManager`

---

### MEJORA-002: Página de videollamada WebRTC

**Descripción:** Se creó la página `TerminatorVideoCall.tsx` que implementa una sala de videollamada completa usando Twilio Video SDK.

**Funcionalidades:**
- Captura de cámara y micrófono locales con `createLocalTracks`
- Conexión a sala Twilio Video con token JWT del servidor
- Grabación de audio en chunks de 5 segundos con `MediaRecorder`
- Upload de audio a S3 para STT
- Reproducción de respuesta de audio y vídeo generado
- Historial de mensajes de la sesión de videollamada
- Botón de acceso desde `TerminatorChat.tsx`

---

### MEJORA-003: Sistema de claves API por usuario

**Descripción:** Se implementó un sistema completo de gestión de claves API que permite a cada usuario configurar sus propias credenciales sin depender de las variables de entorno del servidor.

**Componentes:**
- **Hook `useApiKeys`**: gestiona tres capas (localStorage → BD → env). Expone `serverPresence` (booleanos de presencia sin revelar valores), `updateKey`, `saveToServer` y `clearAll`.
- **Componente `ApiKeyManager`**: UI con grupos colapsables (LLMs en la nube, backends locales, Twilio, WhatsApp, Telegram), inputs con show/hide, badges de estado servidor, y hints de configuración.
- **`TerminatorConfig.tsx` reescrito**: tres pestañas (Claves API / Preferencias / Webhooks), guía de configuración de webhooks con URLs copiables.

---

## 3. Deuda Técnica Identificada (Pendiente)

### DT-001: Backends de vídeo local son stubs

**Severidad:** Media  
**Archivo:** `server/terminator/video.ts`  
**Descripción:** Los backends de vídeo local (Wan2.1, LTX-Video) solo detectan si el servicio está disponible en los puertos 7860/8000/8080, pero no implementan la generación real. El código registra "local backend detected but not yet integrated".

**Recomendación:** Implementar la llamada a la API de Wan2.1 (`POST /generate`) y LTX-Video (`POST /api/generate`) cuando estos servicios estén disponibles localmente.

---

### DT-002: Árabe no soportado en Kokoro

**Severidad:** Baja  
**Archivo:** `server/terminator/tts.ts`  
**Descripción:** El plan de implementación mencionaba soporte para árabe (`ar`) en Kokoro, pero el modelo Kokoro v1.0 no incluye árabe en su conjunto de idiomas. El mapa `KOKORO_LANG_PREFIX` no tiene entrada para `ar`.

**Recomendación:** Cuando Kokoro añada soporte para árabe en versiones futuras, añadir la entrada `ar: "ar"` al mapa.

---

### DT-003: Vídeo generado en videollamada no es track Twilio real

**Severidad:** Baja  
**Archivo:** `client/src/pages/TerminatorVideoCall.tsx`  
**Descripción:** El vídeo generado por IA se reproduce en el elemento `<video>` del panel remoto directamente (`videoEl.src = result.videoUrl`), en lugar de publicarse como un track de vídeo real en la sala Twilio. Esto significa que otros participantes de la sala no verían el vídeo generado.

**Recomendación:** Para una implementación completa, el servidor debería publicar el vídeo generado como un `RemoteVideoTrack` usando Twilio Media Extensions o un participante bot en la sala.

---

## 4. Estado de Compilación y Tests

### TypeScript

```
npx tsc --noEmit → 0 errores
```

### Tests (Vitest)

```
Test Files  3 passed (3)
     Tests  15 passed (15)
  Duration  4.11s
```

| Suite | Tests | Estado |
|---|---|---|
| `auth.logout.test.ts` | 1 | ✅ Pasando |
| `training.test.ts` | 4 | ✅ Pasando |
| `terminator.test.ts` | 10 | ✅ Pasando |

### Dependencias

| Paquete | Versión | Propósito |
|---|---|---|
| `twilio` | ^5.x | SMS, voz, videollamada |
| `twilio-video` | ^2.x | SDK cliente WebRTC |
| `multer` | ^2.1.1 | Upload de archivos multipart |
| `@types/multer` | ^2.1.0 | Tipos TypeScript para multer |
| `ws` | ^8.x | WebSocket para ConversationRelay |
| `nanoid` | ^5.x | IDs únicos para conversaciones |

---

## 5. Checklist de Seguridad

| Ítem | Estado | Notas |
|---|---|---|
| Claves API nunca devueltas en texto plano | ✅ | `getConfig` devuelve solo booleanos |
| Validación de entrada con Zod en todos los endpoints | ✅ | Todos los inputs tRPC tienen schema Zod |
| Autenticación JWT para operaciones sensibles | ✅ | `protectedProcedure` en `saveConfig` y `getVideoToken` |
| Límite de tamaño en upload de audio | ✅ | 16 MB máximo en multer |
| Verificación de token en webhook de WhatsApp | ✅ | Compara `hub.verify_token` con `WHATSAPP_VERIFY_TOKEN` |
| HTTPS obligatorio en producción | ⚠️ | Responsabilidad del operador/plataforma de hosting |
| Rate limiting en endpoints públicos | ❌ | No implementado — recomendado para producción |

---

*Informe generado por Manus AI — Junio 2026*
