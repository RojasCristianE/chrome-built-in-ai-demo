---
name: chrome-built-in-ai
description: Build web applications and Chrome Extensions that use Chrome's built-in AI APIs (Prompt API and Summarizer API) powered by Gemini Nano running locally on-device. Use this skill whenever the user mentions 'Chrome AI', 'built-in AI', 'Gemini Nano', 'on-device AI', 'client-side AI', 'LanguageModel', 'Summarizer', 'ai.languageModel', 'local LLM in browser', 'prompt API', 'summarizer API', 'browser AI', 'edge AI in Chrome', 'offline AI', or wants to build features like local text summarization, on-device chat, client-side content classification, structured JSON extraction from text, or any AI feature that runs entirely in the user's browser without server calls. Also trigger when the user wants to add a cloud fallback for local AI, implement hybrid inference (local + remote), or handle the model download flow for end users.
---

# Chrome Built-in AI: Prompt API & Summarizer API

This skill provides everything needed to build web applications and Chrome Extensions that leverage Chrome's built-in AI APIs. These APIs run Gemini Nano **locally on the user's device**, meaning zero server costs, full offline capability, and complete data privacy.

## Covered APIs

| API | Global Object | Availability | Docs Reference |
|---|---|---|---|
| **Prompt API** | `LanguageModel` | Chrome 138 (Extensions), Chrome 148 (Web) | `references/prompt-api/` |
| **Summarizer API** | `Summarizer` | Chrome 138 (Web + Extensions) | `references/summarizer-api/` |

> Other built-in AI APIs exist (Translator, Language Detector, Writer, Rewriter, Proofreader) but are **out of scope** for this skill. If the user asks about those, mention they exist but don't attempt to implement them with these instructions.

## Hardware & Environment Requirements

Before writing any code, confirm the user understands these constraints — they affect end users, not just developers:

- **OS**: Windows 10/11, macOS 13+, Linux, ChromeOS (Chromebook Plus only). No Android/iOS support.
- **Storage**: ≥22 GB free disk space on the Chrome profile volume (model is ~1.5–2 GB but Chrome reserves headroom).
- **GPU**: >4 GB VRAM, **or** CPU: ≥16 GB RAM + ≥4 cores. Audio input in the Prompt API requires GPU.
- **Network**: Unmetered connection required for initial model download only. After that, fully offline.
- **Model removal**: If free space drops below 10 GB after download, Chrome removes the model automatically.

Check model status at `chrome://on-device-internals` in the user's browser.

## Feature Detection Pattern (Use This Always)

Every project using these APIs **must** include feature detection before attempting to use them. Never assume the API exists.

### Prompt API

```javascript
// Feature detection
if (!('LanguageModel' in self)) {
  // API not available — show fallback UI or use cloud API
  return;
}

// Check availability (pass the SAME options you'll use in prompt/promptStreaming)
const availability = await LanguageModel.availability({
  expectedInputs: [{ type: 'text', languages: ['en'] }],
  expectedOutputs: [{ type: 'text', languages: ['en'] }],
});

if (availability === 'unavailable') {
  // Model can't run on this device
  return;
}

// Create session (requires user activation — a click, tap, or keypress)
const session = await LanguageModel.create({
  // Optional: system prompt via initialPrompts
  initialPrompts: [
    { role: 'system', content: 'You are a helpful assistant.' }
  ],
  // Optional: monitor download progress
  monitor(m) {
    m.addEventListener('downloadprogress', (e) => {
      console.log(`Downloaded ${e.loaded * 100}%`);
    });
  },
});
```

### Summarizer API

```javascript
if (!('Summarizer' in self)) {
  return;
}

const availability = await Summarizer.availability();

if (availability === 'unavailable') {
  return;
}

// Create summarizer (requires user activation)
const summarizer = await Summarizer.create({
  type: 'key-points',      // 'key-points' | 'tldr' | 'teaser' | 'headline'
  format: 'markdown',       // 'markdown' | 'plain-text'
  length: 'medium',         // 'short' | 'medium' | 'long'
  monitor(m) {
    m.addEventListener('downloadprogress', (e) => {
      console.log(`Downloaded ${e.loaded * 100}%`);
    });
  },
});
```

## Prompt API — Core Patterns

### Basic prompt (batch)

```javascript
const result = await session.prompt('What is the capital of France?');
console.log(result); // "The capital of France is Paris."
```

### Streaming prompt

```javascript
const stream = session.promptStreaming('Write me a poem!');
for await (const chunk of stream) {
  console.log(chunk); // Each chunk is the FULL response so far
}
```

### Structured output with JSON Schema

Force the model to respond with valid JSON by passing `responseConstraint`:

```javascript
const schema = {
  type: 'object',
  properties: {
    sentiment: { type: 'string', enum: ['positive', 'negative', 'neutral'] },
    confidence: { type: 'number', minimum: 0, maximum: 1 }
  },
  required: ['sentiment', 'confidence'],
  additionalProperties: false
};

const result = await session.prompt(
  `Classify the sentiment: "This product is amazing!"`,
  { responseConstraint: schema }
);
const parsed = JSON.parse(result);
// { sentiment: "positive", confidence: 0.95 }
```

### Multimodal input (image + audio)

The Prompt API supports images and audio as input (text-only output):

```javascript
const session = await LanguageModel.create({
  expectedInputs: [
    { type: 'text', languages: ['en'] },
    { type: 'image' },
  ],
  expectedOutputs: [{ type: 'text', languages: ['en'] }],
});

const imageBlob = await (await fetch('photo.jpg')).blob();
const response = await session.prompt([{
  role: 'user',
  content: [
    { type: 'text', value: 'Describe this image:' },
    { type: 'image', value: imageBlob },
  ],
}]);
```

Supported image sources: `HTMLImageElement`, `HTMLCanvasElement`, `HTMLVideoElement`, `ImageBitmap`, `Blob`, `ImageData`, `OffscreenCanvas`, `VideoFrame`.
Supported audio sources: `AudioBuffer`, `ArrayBufferView`, `ArrayBuffer`, `Blob`. Audio input **requires GPU**.

### Session management

```javascript
// Check context usage
console.log(`${session.contextUsage}/${session.contextWindow}`);

// Listen for context overflow (oldest messages get evicted)
session.addEventListener('contextoverflow', () => {
  console.warn('Context window overflow — oldest messages are being dropped.');
});

// Clone a session (preserves context + initialPrompts)
const clone = await session.clone();

// Abort a running prompt
const controller = new AbortController();
const result = await session.prompt('Write an essay', {
  signal: controller.signal,
});
controller.abort(); // cancels

// Destroy when done (frees memory)
session.destroy();
```

### Constrain response with prefix

Guide the model to use a specific format by prefilling the assistant response:

```javascript
const result = await session.prompt([
  { role: 'user', content: 'Create a TOML character sheet for a barbarian' },
  { role: 'assistant', content: '```toml\n', prefix: true },
]);
```

## Summarizer API — Core Patterns

### Summary types and lengths

| Type | Short | Medium | Long |
|---|---|---|---|
| `tldr` | 1 sentence | 3 sentences | 5 sentences |
| `teaser` | 1 sentence | 3 sentences | 5 sentences |
| `key-points` | 3 bullets | 5 bullets | 7 bullets |
| `headline` | 12 words | 17 words | 22 words |

### Batch summarization

```javascript
const summary = await summarizer.summarize(longText, {
  context: 'This article is about renewable energy.',
});
```

> **Tip**: Use `element.innerText` instead of `innerHTML` to strip markup before summarizing.

### Streaming summarization

```javascript
const stream = summarizer.summarizeStreaming(longText);
for await (const chunk of stream) {
  console.log(chunk);
}
```

### Summary of summaries (for long documents)

When text exceeds the context window (~750 tokens ≈ 3000 chars), split the text and summarize each chunk, then summarize the concatenated summaries:

```javascript
// Use RecursiveCharacterTextSplitter from LangChain.js or manual splitting
// chunkSize: ~3000 chars, chunkOverlap: ~200 chars
const chunks = splitText(longDocument);
const partialSummaries = [];

for (const chunk of chunks) {
  const summary = await summarizer.summarize(chunk);
  partialSummaries.push(summary);
}

const finalSummary = await summarizer.summarize(partialSummaries.join('\n'));
```

For recursive summarization of very long documents, see `references/summarizer-api/scale-summarization.md`.

### Language support

Both APIs support: `en`, `ja`, `es`, `de`, `fr`. Set languages explicitly:

```javascript
const summarizer = await Summarizer.create({
  type: 'key-points',
  expectedInputLanguages: ['en', 'es'],
  outputLanguage: 'es',
  expectedContextLanguages: ['en'],
  sharedContext: 'Summarize articles from a multilingual newspaper in Spanish.',
});
```

## Session Compacting (Advanced)

For long-lived conversations that approach the context window limit, use the Summarizer API to compact session history:

1. Monitor `session.contextUsage` relative to `session.contextWindow`.
2. Listen for `contextoverflow` events.
3. Summarize each message in history with the Summarizer API.
4. Destroy old session, create new one with compacted summaries as `initialPrompts`.

For full implementation details with code, read `references/prompt-api/session-compacting.md`.

## Hybrid Inference Pattern (Local + Cloud Fallback)

A resilient production pattern: try local first, fall back to a cloud provider if unavailable.

```javascript
async function hybridInference(prompt, cloudConfig) {
  // 1. Try local (Prompt API)
  if ('LanguageModel' in self) {
    const availability = await LanguageModel.availability();
    if (availability !== 'unavailable') {
      try {
        const session = await LanguageModel.create();
        const result = await session.prompt(prompt);
        session.destroy();
        return { source: 'local', result };
      } catch (err) {
        console.warn('Local inference failed, falling back to cloud:', err);
      }
    }
  }

  // 2. Fallback to cloud API (e.g., Cloudflare Workers AI)
  const { accountId, apiToken, model } = cloudConfig;
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${accountId}/ai/run/${model}`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${apiToken}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ prompt }),
    }
  );
  const data = await response.json();
  return { source: 'cloud', result: data.result.response };
}
```

## Permission Policy & iframes

Both APIs are available to top-level windows and same-origin iframes by default. For cross-origin iframes, grant access via the `allow` attribute:

```html
<!-- Prompt API -->
<iframe src="https://example.com/" allow="language-model"></iframe>

<!-- Summarizer API -->
<iframe src="https://example.com/" allow="summarizer"></iframe>
```

Neither API is available in Web Workers currently.

## Common Pitfalls

1. **User activation required**: `create()` requires a user gesture (click, tap, keypress). Don't call it on page load — call it in response to user interaction.
2. **Don't cache sessions forever**: Sessions consume memory. Destroy them when not in use. The model unloads after a period with no active sessions.
3. **Pass same options to `availability()` and `prompt()`**: The availability check must reflect the exact options you'll use, including languages and modalities.
4. **`promptStreaming()` chunks are cumulative**: Each chunk contains the full response so far, not just the new delta.
5. **`innerText` not `innerHTML`**: Strip HTML before sending content to the Summarizer API.
6. **JSON Schema eats tokens**: The `responseConstraint` schema is included in the context window. Use `omitResponseConstraintInput: true` if you guide format via the prompt itself.
7. **22 GB storage**: Users need significant free space. Detect `availability === 'after-download'` and show a clear download prompt with size information.

## TypeScript Support

Install official type definitions:

```bash
npm install --save-dev @types/dom-chromium-ai
```

This provides full typings for `LanguageModel`, `Summarizer`, and all related interfaces.

## Decision Flow: Which API to Use

```
User needs AI in the browser?
├── Summarize/condense text? → Summarizer API
│   ├── Short text (< 3000 chars)? → Direct summarize()
│   └── Long text? → Summary of summaries pattern
├── General prompting/chat? → Prompt API
│   ├── Need structured JSON output? → Use responseConstraint
│   ├── Need image/audio input? → Use multimodal with expectedInputs
│   └── Long conversation? → Monitor contextUsage, use session compacting
└── Needs to work without Chrome/offline fallback? → Hybrid inference pattern
```

## Reference Documentation

For detailed API documentation beyond what's covered in this skill, read the reference files:

### Prompt API
- `references/prompt-api/api-reference.md` — Full API reference with all options, examples, and edge cases
- `references/prompt-api/structured-output.md` — JSON Schema constraint details
- `references/prompt-api/session-management.md` — Session cloning, restoration, and quota management
- `references/prompt-api/session-compacting.md` — Summarizer-based session compaction for long conversations

### Summarizer API
- `references/summarizer-api/api-reference.md` — Full API reference with types, formats, and lengths
- `references/summarizer-api/scale-summarization.md` — Summary of summaries technique for long documents
