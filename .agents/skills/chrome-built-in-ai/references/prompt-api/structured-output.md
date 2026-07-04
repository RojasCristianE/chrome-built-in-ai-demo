---
feature: Structured Output
introduced_in: Chrome 137
parameter: responseConstraint
format: JSON Schema (Draft-04 to latest)
use_cases:
  - Classification (Sentiment, Category, Tags)
  - Data Extraction (Contact details, Dates, Events)
  - Parameterized responses (Grades, Ratings, Metrics)
options:
  - omitResponseConstraintInput: "boolean (avoids duplicating schema instructions inside the prompt context window)"
---

# Structured Output Reference

## Uso de `responseConstraint`
Permite restringir la respuesta del modelo Gemini Nano a una estructura JSON específica utilizando un esquema JSON Schema.

```javascript
const session = await LanguageModel.create();

const schema = {
  type: "object",
  properties: {
    categoria: { type: "string", enum: ["soporte", "ventas", "facturacion", "tecnico"] },
    urgencia: { type: "string", enum: ["baja", "media", "alta"] }
  },
  required: ["categoria", "urgencia"],
  additionalProperties: false
};

const result = await session.prompt(
  "Clasifica el siguiente ticket: 'No puedo acceder a mi panel de facturas.'",
  {
    responseConstraint: schema
  }
);

const parsed = JSON.parse(result);
console.log(parsed.categoria); // "facturacion"
console.log(parsed.urgencia);  // "media"
```

---

## Restricciones y Estructuras Comunes

### Respuesta Booleana Simple
```javascript
const schema = { type: "boolean" };
const result = await session.prompt("¿Es este texto spam? '¡Felicidades, ganaste $1000!'", {
  responseConstraint: schema
});
const isSpam = JSON.parse(result); // true o false
```

### Arreglos con Patrón RegExp
```javascript
const schema = {
  type: "object",
  properties: {
    hashtags: {
      type: "array",
      maxItems: 3,
      items: {
        type: "string",
        pattern: "^#[a-zA-Z0-9]+$"
      }
    }
  },
  required: ["hashtags"],
  additionalProperties: false
};
```

---

## Optimización de Tokens (`omitResponseConstraintInput`)
Por defecto, el navegador adjunta las directivas del esquema JSON Schema dentro de la ventana de contexto del prompt. Para evitar este consumo de tokens redundantes, se puede especificar `omitResponseConstraintInput: true` y proporcionar una guía textual simple en el prompt:

```javascript
const schema = {
  type: "object",
  properties: {
    calificacion: { type: "number", minimum: 0, maximum: 5 }
  },
  required: ["calificacion"],
  additionalProperties: false
};

const feedback = "El servicio fue excelente, pero la comida tardó demasiado.";

const result = await session.prompt(
  `Resume el feedback en un objeto JSON con la propiedad 'calificacion' (número de 0 a 5):\n\n${feedback}`,
  {
    responseConstraint: schema,
    omitResponseConstraintInput: true // Optimiza espacio de tokens en el contexto
  }
);
```
