---
flow: Session Compacting
dependencies:
  - self.ai.languageModel: Prompt API
  - self.ai.summarizer: Summarizer API
  - self.ai.languageDetector: Language Detector API
steps:
  - "Monitor contextUsage and trigger warning when reaching limit"
  - "Detect the language of each conversation turn"
  - "Summarize each chat turn (prose only) into a concise version"
  - "Recreate the prompt session using summaries as initialPrompts"
  - "Maintain a separate fullHistory for fallback recovery"
---

# Session Compacting Reference

El objetivo es comprimir el historial de una conversación larga antes de que ocurra una pérdida de contexto por desbordamiento (`contextoverflow`), recreando la sesión con los resúmenes en `initialPrompts`.

## Implementación de Compactación Completa

```javascript
const fullHistory = []; // Almacena el historial original sin alteraciones
let activeSession = null;

async function compactSession() {
  if (!activeSession) return;
  
  console.log("Iniciando compactación del historial...");
  const compactedPrompts = [];
  
  // 1. Procesar y resumir cada mensaje del historial
  for (const message of fullHistory) {
    try {
      // Detectar idioma
      const lang = await detectLanguage(message.content) || navigator.language;
      
      // Separar prosa de bloques de código (para no resumir el código)
      const parts = splitByCodeFences(message.content);
      const processedParts = [];
      
      for (const part of parts) {
        if (part.type === 'prose' && part.content.trim().length > 100) {
          const summarizer = await getCachedSummarizer('plain-text', lang);
          const summary = await summarizer.summarize(part.content, {
            context: `Esta es una intervención de rol '${message.role}' en un chat. Resúmela al máximo manteniendo el sentido.`
          });
          processedParts.push(summary);
        } else {
          processedParts.push(part.content); // Mantener código o textos cortos intactos
        }
      }
      
      compactedPrompts.push({
        role: message.role,
        content: processedParts.join('\n')
      });
    } catch (e) {
      console.warn("Fallo al resumir el mensaje, se mantiene versión original:", e);
      compactedPrompts.push({ role: message.role, content: message.content });
    }
  }

  // 2. Destruir sesión antigua y liberar memoria
  activeSession.destroy();
  
  // 3. Crear nueva sesión seeded con el historial compactado
  const languages = [...new Set(fullHistory.map(m => m.lang).filter(Boolean))];
  
  activeSession = await LanguageModel.create({
    expectedInputs: [{ type: 'text', languages: languages.length ? languages : [navigator.language] }],
    expectedOutputs: [{ type: 'text', languages: languages.length ? languages : [navigator.language] }],
    initialPrompts: compactedPrompts
  });
  
  // Re-registrar el listener de overflow
  activeSession.addEventListener('contextoverflow', handleOverflow);
  console.log("Nueva sesión inicializada. Tokens usados:", activeSession.contextUsage);
}
```

---

## Utilidades de Apoyo

### 1. Detección de Idioma
Evita errores de traducción y optimiza la inicialización del modelo de resumen.

```javascript
async function detectLanguage(text) {
  if (!('LanguageDetector' in self)) return null;
  try {
    const detector = await LanguageDetector.create();
    const results = await detector.detect(text);
    if (results.length > 0 && results[0].confidence > 0.7) {
      return results[0].detectedLanguage;
    }
  } catch (e) {
    console.error(e);
  }
  return null;
}
```

### 2. Separar Código de Prosa (RegExp Splitter)
Evita la destrucción o malformación de sintaxis de bloques de código por parte del Summarizer.

```javascript
function splitByCodeFences(text) {
  const parts = [];
  const regex = /^```[^\n]*\n[\s\S]*?^```[ \t]*$/gm;
  let lastIndex = 0;
  let match;
  
  while ((match = regex.exec(text)) !== null) {
    if (match.index > lastIndex) {
      parts.push({
        type: 'prose',
        content: text.slice(lastIndex, match.index)
      });
    }
    parts.push({ type: 'code', content: match[0] });
    lastIndex = match.index + match[0].length;
  }
  
  if (lastIndex < text.length) {
    parts.push({ type: 'prose', content: text.slice(lastIndex) });
  }
  return parts;
}
```

### 3. Caching de Instancias de Summarizer
```javascript
const summarizerCache = {};

async function getCachedSummarizer(format, lang) {
  const key = `${format}:${lang}`;
  if (summarizerCache[key]) return summarizerCache[key];
  
  const options = {
    type: 'tldr',
    format: format,
    length: 'short',
    expectedInputLanguages: [lang],
    outputLanguage: lang
  };
  
  let availability = await Summarizer.availability(options);
  if (availability === 'unavailable') {
    throw new Error(`Summarizer no disponible para idioma: ${lang}`);
  }
  
  summarizerCache[key] = await Summarizer.create(options);
  return summarizerCache[key];
}
```