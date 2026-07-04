---
api_name: Summarizer API
namespace: self.ai.summarizer
status: Shipping in Chrome 138 (Web + Extensions)
methods:
  - availability(options): Checks model status ('no' | 'readily' | 'after-download')
  - create(options): Instantiates a summarizer (requires user activation gesture)
  - summarize(text, options): Returns final text summary (batch)
  - summarizeStreaming(text, options): Returns a ReadableStream of cumulative text summary chunks
options:
  type: "'key-points' (default) | 'tldr' | 'teaser' | 'headline'"
  format: "'markdown' (default) | 'plain-text'"
  length: "'short' (default) | 'medium' | 'long'"
  preference: "'auto' (default) | 'speed' (prioritizes latency) | 'capability' (prioritizes nuances)"
  expectedInputLanguages: "Array of ISO 639-1 language codes (e.g. ['en', 'es'])"
  outputLanguage: "ISO 639-1 language code (e.g. 'es')"
---

# Summarizer API Reference

## Detección y Creación del Resumidor

```javascript
// Validar soporte
if (!('Summarizer' in self)) {
  console.log("Summarizer API no disponible en este navegador.");
  return;
}

const availability = await Summarizer.availability();
if (availability === 'unavailable') {
  console.log("El hardware no soporta la ejecución local de resúmenes.");
  return;
}

// Crear instancia del resumidor (requiere interacción del usuario)
const summarizer = await Summarizer.create({
  type: 'key-points',  // Viñetas clave
  format: 'markdown',   // Salida formateada
  length: 'medium',     // Longitud moderada
  preference: 'speed',  // Priorizar baja latencia
  monitor(m) {
    m.addEventListener('downloadprogress', (e) => {
      console.log(`Descargando modelo: ${(e.loaded / e.total) * 100}%`);
    });
  }
});
```

---

## Ejecución del Resumen

### Resumen en Batch (`summarizer.summarize()`)
Envía el texto y espera la respuesta completa. Se aconseja limpiar el texto y usar `element.innerText` para evitar enviar etiquetas HTML.

```javascript
const textToSummarize = document.getElementById('article-content').innerText;
const summary = await summarizer.summarize(textToSummarize, {
  context: "Este es un artículo técnico enfocado en desarrollo de software."
});
console.log(summary);
```

### Resumen en Streaming (`summarizer.summarizeStreaming()`)
Retorna un `ReadableStream`. Cada chunk del stream es el texto **completo** acumulado hasta ese momento.

```javascript
const stream = summarizer.summarizeStreaming(textToSummarize);
for await (const chunk of stream) {
  console.clear();
  console.log(chunk); // Muestra el progreso acumulado del resumen
}
```

---

## Tipos y Longitudes Soportadas
La longitud máxima de la salida está condicionada por el tipo de resumen:

| Tipo (`type`) | Significado | Longitud (`length`) | Comportamiento en Chrome |
|---|---|---|---|
| `"tldr"` | Resumen corto y directo | short / medium / long | 1 / 3 / 5 frases |
| `"teaser"` | Enfoque intriga/gancho | short / medium / long | 1 / 3 / 5 frases |
| `"key-points"` | Lista con viñetas | short / medium / long | 3 / 5 / 7 viñetas |
| `"headline"` | Línea de título/encabezado | short / medium / long | 12 / 17 / 22 palabras |
