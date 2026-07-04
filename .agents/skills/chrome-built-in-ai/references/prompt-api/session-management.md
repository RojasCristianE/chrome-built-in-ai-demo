---
feature: Session Management
namespace: LanguageModelInstance
properties:
  - contextWindow: "Total token capacity limit of the model context"
  - contextUsage: "Active token count currently consumed by the session"
methods:
  - clone(options): "Creates an independent copy of the session preserving history"
  - destroy(): "Terminates the session and frees browser memory resources"
events:
  - contextoverflow: "Dispatched when the context window limit is exceeded and history eviction begins"
best_practices:
  - "Clone a pre-seeded master session instead of re-instantiating multiple initial prompts"
  - "Always call destroy() to free up VRAM/RAM immediately after usage"
  - "Cache a single empty session to keep the on-device model warmed up in memory"
---

# Session Management Reference

## Clonación de Sesiones (`session.clone()`)
Hereda el historial de interacciones y los prompts de sistema del objeto padre. Permite ejecutar múltiples hilos de conversación independientes partiendo de una misma base común de forma inmediata.

```javascript
// Inicializar sesión maestra con prompt de sistema pesado
const masterSession = await LanguageModel.create({
  initialPrompts: [
    { role: 'system', content: 'Eres un asistente experto en la legislación laboral de Nicaragua.' }
  ]
});

// Clonar de forma independiente para atender a múltiples usuarios
const sessionUsuario1 = await masterSession.clone();
const sessionUsuario2 = await masterSession.clone();

// Se pueden usar en paralelo sin interferir entre sí
const res1 = await sessionUsuario1.prompt("¿Cuánto dura el periodo de prueba?");
const res2 = await sessionUsuario2.prompt("¿Cómo se calculan las horas extras?");
```

---

## Control de Cuota de Contexto
Cada prompt y respuesta se acumula en la ventana de contexto de la sesión.

```javascript
// Consultar estado de la ventana de contexto (tokens)
const { contextWindow, contextUsage } = session;
const tokensDisponibles = contextWindow - contextUsage;
console.log(`Disponibles: ${tokensDisponibles} de ${contextWindow}`);

// Capturar desbordamiento
session.addEventListener('contextoverflow', () => {
  console.warn("Se ha alcanzado el límite de tokens. Las primeras interacciones serán descartadas.");
});
```

---

## Restauración de Sesiones Históricas (n-shot restore)
Dado que no existe una API nativa de serialización a disco, se puede guardar el historial en `localStorage` y restaurarlo al inicializar una nueva sesión pasando el historial en la propiedad `initialPrompts`.

```javascript
const sessionUuid = "user-session-12345";

// Guardar interacción
function saveInteraction(uuid, role, content) {
  const history = JSON.parse(localStorage.getItem(uuid) || '[]');
  history.push({ role, content });
  localStorage.setItem(uuid, JSON.stringify(history));
}

// Restaurar sesión
async function restoreSession(uuid) {
  const history = JSON.parse(localStorage.getItem(uuid) || '[]');
  
  // Se inicializa pasándole el historial completo recreado
  return await LanguageModel.create({
    initialPrompts: history
  });
}
```

---

## Liberación de Recursos
El modelo on-device consume memoria del sistema. Es una buena práctica destruir las sesiones inactivas:

```javascript
// Destruir y liberar recursos en memoria
session.destroy();
```