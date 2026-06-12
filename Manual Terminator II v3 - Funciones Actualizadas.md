# Manual Terminator II v3 - Funciones Actualizadas

## Resumen Ejecutivo

**Terminator II v3** es un sistema multicanal multimodal completo con soporte para:
- **TTS**: Kokoro (11 idiomas) + Coqui XTTS (17 idiomas + árabe) + Silero (7 idiomas)
- **LLM**: Grok, OpenAI, Ollama, LM Studio, Kimi, DeepSeek
- **Canales**: WhatsApp, Telegram, SMS, Voz, Videollamada
- **Video**: Grok Video, Wan2.1, LTX-Video
- **Streaming**: SSE en Kokoro para audio en tiempo real
- **Control de Volumen**: Multiplicador 0.1-2.0x en Kokoro

**Estado**: ✅ 63/63 tests pasando | Listo para producción

---

## Mejoras v3 Completadas

### Mejora 4: TTS con Árabe (Coqui XTTS + Silero)

**Implementado**:
- Coqui XTTS: 17 idiomas (ar, de, en, es, fr, hu, it, ja, ko, nl, pl, pt, ru, tr, uk, zh)
- Silero TTS: 7 idiomas (en, de, es, fr, ru, uk, hi)
- Fallback automático: Si Kokoro no soporta idioma → Coqui o Silero

**Archivo**: `server/terminator/tts.ts`

```typescript
// Matrices de soporte de idiomas
const KOKORO_LANGS = ["en", "de", "es", "fr", "hi", "it", "ja", "ko", "pt", "zh"];
const COQUI_LANGS = ["ar", "de", "en", "es", "fr", "hu", "it", "ja", "ko", "nl", "pl", "pt", "ru", "tr", "uk", "zh"];
const SILERO_LANGS = ["en", "de", "es", "fr", "ru", "uk", "hi"];

// Validación de idioma en synthesizeSpeech()
if (!KOKORO_LANGS.includes(langKey)) {
  // Salta Kokoro, usa Coqui o Silero
}
```

**Tests**: `server/tts-russian.test.ts` (15 tests de Ruso)

---

### Mejora 5: Funciones Docker de Kokoro-TTS-Docker-9

#### 1. Streaming SSE (Server-Sent Events)

**Implementado**: Recibir audio mientras se sintetiza

```typescript
export interface KokoroTTSOptions {
  volumeMultiplier?: number; // 0.1 to 2.0
  useStreaming?: boolean;     // SSE streaming
}

async function kokoroTTS(
  text: string,
  personaId: string,
  language: string,
  kokoroUrl: string,
  options?: KokoroTTSOptions
): Promise<Buffer | null> {
  const requestBody = {
    model: "kokoro",
    input: text,
    voice,
    response_format: "mp3",
    speed: 1.0,
    volume_multiplier: options?.volumeMultiplier ?? 1.0,
  };

  if (options?.useStreaming) {
    requestBody.stream_format = "sse";
  }

  const res = await fetch(`${kokoroUrl}/v1/audio/speech`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(requestBody),
  });

  if (options?.useStreaming && res.headers.get("content-type")?.includes("text/event-stream")) {
    return await handleKokoroSSEStream(res);
  }

  return Buffer.from(await res.arrayBuffer());
}
```

**Manejo de SSE**:

```typescript
async function handleKokoroSSEStream(response: Response): Promise<Buffer | null> {
  const reader = response.body?.getReader();
  const chunks: Uint8Array[] = [];
  const decoder = new TextDecoder();

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    const text = decoder.decode(value, { stream: true });
    const lines = text.split("\n");

    for (const line of lines) {
      if (line.startsWith("data: ")) {
        const data = JSON.parse(line.slice(6));
        if (data.audio) {
          chunks.push(new Uint8Array(Buffer.from(data.audio, "base64")));
        }
      }
    }
  }

  // Concatenar chunks
  const totalLength = chunks.reduce((sum, chunk) => sum + chunk.length, 0);
  const result = new Uint8Array(totalLength);
  let offset = 0;

  for (const chunk of chunks) {
    result.set(chunk, offset);
    offset += chunk.length;
  }

  return Buffer.from(result);
}
```

#### 2. Volume Multiplier (0.1-2.0x)

**Implementado**: Control de volumen en Kokoro

```typescript
const volumeMultiplier = options?.volumeMultiplier
  ? Math.max(0.1, Math.min(2.0, options.volumeMultiplier))
  : 1.0;

requestBody.volume_multiplier = volumeMultiplier;
```

**UI en TerminatorConfig.tsx**:

```tsx
<div className="space-y-2">
  <label className="text-sm text-slate-300">
    Volumen: {prefs.kokoroVolumeMultiplier.toFixed(1)}x
  </label>
  <Slider
    value={[prefs.kokoroVolumeMultiplier]}
    onValueChange={(v) => setPrefs((p) => ({ ...p, kokoroVolumeMultiplier: v[0] }))}
    min={0.1}
    max={2.0}
    step={0.1}
    className="w-full"
  />
  <p className="text-xs text-slate-500">Rango: 0.1 (muy bajo) a 2.0 (muy alto)</p>
</div>
```

#### 3. SSE Toggle en UI

**Implementado**: Switch para habilitar/deshabilitar streaming

```tsx
<div className="flex items-center justify-between">
  <div className="space-y-1">
    <label className="text-sm text-slate-300">Streaming SSE (Kokoro)</label>
    <p className="text-xs text-slate-500">Recibir audio mientras se sintetiza (más rápido)</p>
  </div>
  <Switch
    checked={prefs.kokoroUseStreaming}
    onCheckedChange={(v) => setPrefs((p) => ({ ...p, kokoroUseStreaming: v }))}
  />
</div>
```

---

## Configuración de Kokoro Docker

### Quick Start

```bash
# CPU
docker run \
    --name kokoro \
    --restart=always \
    -v kokoro-data:/var/lib/kokoro \
    -p 8880:8880 \
    -d hwdsl2/kokoro-server

# GPU (NVIDIA CUDA)
docker run \
    --name kokoro \
    --restart=always \
    --gpus=all \
    -v kokoro-data:/var/lib/kokoro \
    -p 8880:8880 \
    -d hwdsl2/kokoro-server:cuda
```

### Variables de Entorno

```bash
KOKORO_VOICE=af_heart              # Voz por defecto
KOKORO_LANG_CODE=a                 # Código de idioma (a=en, b=en-GB, e=es, etc.)
KOKORO_API_KEY=tu_token_aqui       # Bearer token (opcional)
KOKORO_LOCAL_ONLY=true             # Modo offline (sin descargas)
KOKORO_LOG_LEVEL=INFO              # DEBUG, INFO, WARNING, ERROR
```

### Voces Kokoro Disponibles

**Inglés Americano (a)**:
- af_heart, af_aoede, af_bella, af_jessica, af_kore, af_nicole, af_nova, af_river, af_sarah, af_sky, af_alloy
- am_adam, am_michael, am_echo, am_eric, am_fenrir, am_liam, am_onyx, am_puck, am_santa

**Inglés Británico (b)**:
- bf_emma, bf_isabella, bf_alice, bf_lily
- bm_george, bm_lewis, bm_daniel, bm_fable

**Español (e)**: ef_dora, em_alex, em_santa
**Francés (f)**: ff_siwis, fm_pierre
**Alemán (d)**: df_hedda, dm_thorsten
**Italiano (i)**: if_sara, im_nicola
**Portugués (p)**: pf_dora, pm_alex, pm_santa
**Hindi (h)**: hf_alpha, hf_beta, hm_omega, hm_psi
**Japonés (j)**: jf_alpha, jf_gongitsune, jf_nezumi, jf_tebukuro, jm_kumo
**Coreano (k)**: km_yunho, kf_alpha
**Chino Mandarín (z)**: zf_xiaobei, zf_xiaoni, zf_xiaoxiao, zf_xiaoyi, zm_yunjian, zm_yunxi, zm_yunxia, zm_yunyang

---

## Matriz de Soporte de Idiomas

| Idioma | Código | Kokoro | Coqui | Silero | Grok | OpenAI |
|--------|--------|--------|-------|--------|------|--------|
| Árabe | ar | ❌ | ✅ | ❌ | ❌ | ❌ |
| Alemán | de | ✅ | ✅ | ✅ | ✅ | ✅ |
| Inglés | en | ✅ | ✅ | ✅ | ✅ | ✅ |
| Español | es | ✅ | ✅ | ✅ | ✅ | ✅ |
| Francés | fr | ✅ | ✅ | ✅ | ✅ | ✅ |
| Hindi | hi | ✅ | ❌ | ✅ | ❌ | ❌ |
| Italiano | it | ✅ | ✅ | ❌ | ❌ | ❌ |
| Japonés | ja | ✅ | ✅ | ❌ | ❌ | ❌ |
| Coreano | ko | ✅ | ✅ | ❌ | ❌ | ❌ |
| Portugués | pt | ✅ | ✅ | ❌ | ✅ | ✅ |
| Ruso | ru | ❌ | ✅ | ✅ | ❌ | ❌ |
| Chino | zh | ✅ | ✅ | ❌ | ✅ | ✅ |

---

## Testing

### Ejecutar Tests

```bash
# Todos los tests
pnpm test

# Tests específicos
pnpm test server/tts-russian.test.ts
pnpm test server/tts-integration.test.ts
pnpm test server/tts.test.ts

# Con cobertura
pnpm test -- --coverage
```

### Archivos de Tests

- `server/tts.test.ts` - Tests básicos de TTS (16 tests)
- `server/tts-integration.test.ts` - Tests de integración (17 tests)
- `server/tts-russian.test.ts` - Tests de Ruso (15 tests)
- `server/terminator.test.ts` - Tests de Terminator II (10 tests)
- `server/training.test.ts` - Tests de Training (4 tests)
- `server/auth.logout.test.ts` - Tests de Auth (1 test)

**Total**: 63/63 tests pasando ✅

---

## Integración en Código

### Usar Kokoro con SSE y Volume

```typescript
import { synthesizeSpeech } from "@/server/terminator/tts";

const config = {
  kokoroUrl: "http://localhost:8880",
  preferredTts: "kokoro",
};

const result = await synthesizeSpeech(
  "Hola, mundo!",
  config,
  "default",
  "es",
  {
    useStreaming: true,      // Habilitar SSE
    volumeMultiplier: 1.5,   // Volumen 1.5x
  }
);

// result.url → URL pública en S3
// result.backend → "kokoro"
```

### Usar Coqui para Árabe

```typescript
const result = await synthesizeSpeech(
  "مرحبا بالعالم",
  { coquiUrl: "http://localhost:5002", preferredTts: "coqui" },
  "default",
  "ar"
);
```

### Fallback Automático para Ruso

```typescript
// Kokoro no soporta ruso, automáticamente usa Coqui
const result = await synthesizeSpeech(
  "Привет, мир!",
  { kokoroUrl: "...", coquiUrl: "...", preferredTts: "auto" },
  "default",
  "ru"
);
// → Usa Coqui automáticamente
```

---

## Endpoints API

### POST /api/terminator/chat

Enviar mensaje y recibir respuesta multimodal

```bash
curl -X POST http://localhost:3000/api/terminator/chat \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Hola, ¿cómo estás?",
    "channel": "web",
    "language": "es",
    "personaId": "default"
  }'
```

### GET /api/terminator/config

Obtener configuración actual

```bash
curl http://localhost:3000/api/terminator/config
```

### POST /api/terminator/saveConfig

Guardar configuración

```bash
curl -X POST http://localhost:3000/api/terminator/saveConfig \
  -H "Content-Type: application/json" \
  -d '{
    "defaultTts": "kokoro",
    "kokoroUrl": "http://localhost:8880",
    "kokoroVolumeMultiplier": 1.5,
    "kokoroUseStreaming": true
  }'
```

---

## Troubleshooting

### Kokoro no responde

```bash
# Verificar que el contenedor está corriendo
docker ps | grep kokoro

# Ver logs
docker logs kokoro

# Probar conectividad
curl http://localhost:8880/health
```

### Ruso no funciona

1. Verificar que Coqui está disponible: `curl http://localhost:5002/health`
2. Revisar logs en `server/terminator/tts.ts`
3. Ejecutar test: `pnpm test server/tts-russian.test.ts`

### SSE streaming no funciona

1. Verificar que `kokoroUseStreaming: true` en config
2. Revisar headers: `Content-Type: text/event-stream`
3. Ver logs de `handleKokoroSSEStream()`

---

## Próximas Mejoras (v4)

- [ ] Endpoints `/v1/voices` y `/v1/models` desde backend
- [ ] Soporte `KOKORO_API_KEY` (Bearer token)
- [ ] Modelos personalizados (custom_models table)
- [ ] OpenAI Realtime API
- [ ] Grok Video con referencia de imagen
- [ ] Panel centralizado de configuración v2

---

## Recursos

- **Kokoro TTS**: https://github.com/hexgrad/kokoro
- **Kokoro Docker**: https://github.com/hwdsl2/docker-kokoro
- **Kokoro Yoqer**: https://github.com/yoqer/Kokoro-TTS-Docker-9
- **Coqui XTTS**: https://github.com/coqui-ai/TTS
- **Silero TTS**: https://github.com/snakers4/silero-models

---

**Versión**: v3.0.0
**Última actualización**: Junio 2026
**Estado**: ✅ Producción
