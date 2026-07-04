---
pattern: Scale Summarization
technique: Summary of Summaries / Recursive Summarization
concept: Splitting large texts into chunks, summarizing each individually, and summarizing the result recursively
parameters:
  chunkSize: 3000 (characters, approx. 750 tokens)
  chunkOverlap: 200 (characters, to preserve context between bounds)
methods_used:
  - measureInputUsage(): "Checks input tokens length"
  - inputQuota: "Retrieves total allowed tokens for input"
limitations:
  - "Recursion decreases final summary accuracy (farther from source)"
  - "Linear execution takes time proportional to number of chunks"
---

# Scale Summarization Reference

Cuando un documento excede el límite del context window (~750 tokens ≈ 3000 caracteres en Chrome), se divide el texto y se resume de manera escalonada o recursiva.

## Implementación de Resumen de Resúmenes

```javascript
import { RecursiveCharacterTextSplitter } from "langchain/text_splitter";

async function summarizeLargeDocument(documentText) {
  const summarizer = await Summarizer.create({
    type: 'tldr',
    format: 'plain-text',
    length: 'long'
  });

  // 1. Dividir el texto en fragmentos que quepan en la ventana de contexto
  const splitter = new RecursiveCharacterTextSplitter({
    chunkSize: 3000,
    chunkOverlap: 200,
  });
  
  const chunks = await splitter.splitText(documentText);
  const partialSummaries = [];

  // 2. Generar resúmenes individuales para cada fragmento
  for (const chunk of chunks) {
    const summary = await summarizer.summarize(chunk);
    partialSummaries.push(summary);
  }

  // 3. Concatenar y realizar el resumen final de resúmenes
  const combinedSummaries = partialSummaries.join("\n");
  const finalSummary = await summarizer.summarize(combinedSummaries);
  
  summarizer.destroy();
  return finalSummary;
}
```

---

## Resumen de Resúmenes Recursivo
Si el documento original es extremadamente largo, la suma de los primeros resúmenes parciales puede seguir superando la ventana de contexto. Se requiere un bucle recursivo:

```javascript
async function recursiveSummarize(chunks, maxTokenThreshold = 3000) {
  const summarizer = await Summarizer.create({
    type: 'tldr',
    format: 'plain-text',
    length: 'long'
  });

  let activeChunks = [...chunks];
  
  while (activeChunks.length > 1) {
    const summaries = [];
    
    for (const chunk of activeChunks) {
      const summary = await summarizer.summarize(chunk);
      summaries.push(summary);
    }
    
    // Si la unión de resúmenes aún supera el umbral, volvemos a dividir
    const concatenatedText = summaries.join("\n");
    
    if (concatenatedText.length > maxTokenThreshold) {
      const splitter = new RecursiveCharacterTextSplitter({
        chunkSize: maxTokenThreshold,
        chunkOverlap: 200
      });
      activeChunks = await splitter.splitText(concatenatedText);
    } else {
      activeChunks = [concatenatedText]; // Listo para el resumen final
    }
  }

  const finalSummary = await summarizer.summarize(activeChunks[0]);
  summarizer.destroy();
  
  return finalSummary;
}
```