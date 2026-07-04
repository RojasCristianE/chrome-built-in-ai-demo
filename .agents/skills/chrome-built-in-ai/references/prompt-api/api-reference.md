---
api_name: Prompt API
namespace: self.ai.languageModel
status: Shipping in Chrome 148 (Web) / Chrome 138 (Extensions)
hardware_requirements:
  os: Windows 10/11, macOS 13+, Linux, ChromeOS (Chromebook Plus)
  free_storage: ">= 22 GB"
  gpu: "Strictly > 4 GB VRAM (required for audio input)"
  cpu: ">= 16 GB RAM, >= 4 cores"
  network: "Required for initial download only (~1.5-2 GB)"
methods:
  - availability(options): Checks model status ('no' | 'readily' | 'after-download')
  - create(options): Instantiates a session (requires user activation gesture)
  - params(): Returns parameter limits (Extensions or Origin Trial only)
  - prompt(text, options): Returns final text response (batch)
  - promptStreaming(text, options): Returns a ReadableStream of cumulative text chunks
  - clone(options): Forks the session preserving context
  - destroy(): Frees session memory resources
properties:
  - contextWindow: Total token capacity
  - contextUsage: Active token count
events:
  - contextoverflow: Fired when old history starts getting evicted
---

# Prompt API Reference

## Detección y Creación de Sesión

### `LanguageModel.availability(options)`
Retorna una promesa con `'no'` (no soportado), `'readily'` (listo para usar) o `'after-download'` (soportado pero requiere descargar el modelo).

```javascript
const availability = await LanguageModel.availability({
  expectedInputs: [{ type: 'text', languages: ['es'] }],
  expectedOutputs: [{ type: 'text', languages: ['es'] }]
});
```

### `LanguageModel.create(options)`
Crea e inicializa la sesión. Requiere **interacción del usuario** (User Activation) para ejecutarse.

```javascript
const session = await LanguageModel.create({
  initialPrompts: [
    { role: 'system', content: 'Eres un tutor de programación.' }
  ],
  monitor(m) {
    m.addEventListener('downloadprogress', (e) => {
      console.log(`Progreso de descarga: ${(e.loaded / e.total) * 100}%`);
    });
  }
});
```

---

## Ejecución de Prompts

### `session.prompt(text | messages, options)`
Inferencia síncrona (batch). El primer argumento puede ser un string simple o un array de objetos de rol (`messages`).

```javascript
const response = await session.prompt("¿Qué es un closure en JavaScript?");
```

### `session.promptStreaming(text | messages, options)`
Retorna un `ReadableStream`. **Importante:** Cada fragmento retornado en el bucle es acumulativo, no una diferencia (delta).

```javascript
const stream = session.promptStreaming("Escribe un poema corto.");
for await (const chunk of stream) {
  console.clear();
  console.log(chunk); // Muestra el texto completo generado hasta el momento
}
```

### Abortar Inferencia
Se puede pasar un `AbortSignal` a través del objeto de opciones tanto en `prompt` como en `promptStreaming`:

```javascript
const controller = new AbortController();
const response = await session.prompt("Escribe una historia larga...", {
  signal: controller.signal
});
// Para cancelar:
controller.abort();
```

---

## Multimodalidad (Entrada de Imagen y Audio)
La Prompt API de Gemini Nano soporta entradas de múltiples modalidades:

```javascript
const session = await LanguageModel.create({
  expectedInputs: [
    { type: 'text', languages: ['en'] },
    { type: 'image' },
    { type: 'audio' }
  ],
  expectedOutputs: [{ type: 'text', languages: ['en'] }]
});
```

### Orígenes de Imagen Soportados
`HTMLImageElement`, `HTMLCanvasElement`, `HTMLVideoElement`, `ImageBitmap`, `Blob`, `ImageData`, `OffscreenCanvas`, `VideoFrame`.

### Orígenes de Audio Soportados (Requiere GPU)
`AudioBuffer`, `ArrayBufferView`, `ArrayBuffer`, `Blob`.

#### Ejemplo Multimodal:
```javascript
const imageBlob = await (await fetch('diagrama.jpg')).blob();
const response = await session.prompt([
  {
    role: 'user',
    content: [
      { type: 'text', value: 'Describe esta imagen en detalle:' },
      { type: 'image', value: imageBlob }
    ]
  }
]);
```

---

## Control de Formato por Prefijo (Response Prefixing)
Prefijar el mensaje del asistente guía la estructura inicial de la respuesta:

```javascript
const response = await session.prompt([
  { role: 'user', content: 'Dame el JSON de un usuario de pruebas' },
  { role: 'assistant', content: '{\n  "id": 1,', prefix: true }
]);
// La salida completará el JSON empezando desde la estructura predefinida
```
