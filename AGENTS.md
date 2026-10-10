# NEXTSTEP

Profesor particular virtual y personalizado, con IA. No es un chatbot que solo responde preguntas: conoce
al alumno (objetivo, conocimientos, tiempo, dificultades), evalúa su nivel real, arma un camino de
aprendizaje, explica, crea y corrige ejercicios, ayuda con proyectos, planifica el estudio y hace
seguimiento de su progreso. Proyecto grupal de 4 personas (Santino Moauro, Ramiro Tatone,
Juan Pablo Santisi y Thiago Montenegro). El documento de viabilidad define el alcance.

## Stack (actual)
- Node.js **>= 22.5** (requiere `node:sqlite`), ES modules, Express 5, `@anthropic-ai/sdk` (Claude),
  `fetch` nativo (OpenAI y Gemini), `express-rate-limit`, `dotenv`.
- IA: tres proveedores conectados por el router en `src/llm.js` (Claude, ChatGPT, Gemini). La clave de cada
  uno vive en `.env` y nunca llega al navegador.
- Persistencia: **SQLite** con el módulo integrado `node:sqlite` (`DatabaseSync`), sin dependencias nativas.
  Base en `data/nextstep.db` (WAL, gitignore). `DB_PATH` en `.env` la puede mover.
- Frontend SPA sin framework: `public/index.html`, `public/css/style.css`, `public/js/*` (módulos ES).
  Anclas con hash: `#/chat`, `#/aprendizaje`, `#/rutas`, `#/evaluaciones`, `#/objetivos`,
  `#/estadisticas`, `#/recursos`, `#/config`.
- Cuentas propias: registro/login, contraseñas con **scrypt**, sesiones con cookie httpOnly
  (`ns_session`). Cada alumno solo ve lo suyo.

## Comandos
- `npm install` instala dependencias.
- `npm start` levanta en http://localhost:3000. `npm run dev` reinicia solo al guardar.
- `npm run init-db` crea/verifica la base (`data/nextstep.db`).
- `npm test` corre `node --test "test/*.test.mjs"`.
- Sin `.env` la app funciona salvo la IA: `/api/chat` responde 503 `missingConfigMessage()`
  (ningún proveedor listo). Hace falta `.env` (ver `.env.example`): al menos UNA clave
  (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY` o `GEMINI_API_KEY`). Mientras más cargues, más robusto:
  `PROVIDER_ORDER` define el orden (default `anthropic,openai,gemini`).

## Estructura
- `server.js`: API REST + SPA fallback. Rutas: `GET /api/health`, `GET /api/meta`, `GET /api/ia` (estado del
  router + consumo, con sesión), `POST /api/auth/{signup,login,logout}`,
  `GET /api/auth/me`, `GET /api/dashboard`, `GET|POST /api/conversations`, `GET|DELETE /api/conversations/:id`,
  `POST /api/chat` (SSE, emite evento `provider`), `GET|POST /api/subjects`, `PATCH|DELETE /api/subjects/:id`,
  `GET /api/paths`, `POST /api/paths`, `GET|PATCH|DELETE /api/paths/:id`, `POST /api/paths/activities/:aid/complete`,
  `GET|POST /api/evaluations`, `GET /api/evaluations/:id`, `POST /api/evaluations/:id/submit`,
  `GET|POST /api/goals`, `PATCH|DELETE /api/goals/:id`, `POST /api/goals/:id/tasks`,
  `PATCH|DELETE /api/goals/tasks/:tid`, `GET /api/stats`, `POST /api/study/track`,
  `GET|POST /api/resources`, `DELETE /api/resources/:id`, `GET|PATCH /api/profile`,
  `GET|DELETE /api/profile/data`.
- `src/db.js`: esquema SQLite completo y helpers de acceso. Toda la persistencia y consultas acá.
  Incluye `llm_usage` y `provider_state` (consumo y cooldown por proveedor de IA) y migraciones con `ensureColumn`.
- `src/auth.js`: `hashPassword`/`verifyPassword` (scrypt), `requireAuth`, cookie de sesión, sanitización.
- `src/xp.js`: `RULES` de XP (chat, ejercicio, evaluación, tarea, actividad de ruta, módulo, 10 min de estudio), `earn()`, racha.
- `src/llm.js`: **router multi-proveedor**. `streamChat()` (SSE) recorre `PROVIDER_ORDER` y, si un proveedor se
  queda sin cupo/tokens o falla (`shouldShift`), pasa al siguiente; pausa el que falló (`provider_state`,
  `PROVIDER_COOLDOWN_SEC`) y registra tokens en `llm_usage`. `onProvider(id,label,model)` avisa quién responde.
  `completeJson()` (evaluaciones/rutas), `routerInfo()`/`providerInfo()`/`missingConfigMessage()`.
- `src/prompt.js`: el "cerebro" del profesor: estilo, proceso, etiquetas `<resumen>`, `<perfil>`, `<opciones>`,
  generación de evaluaciones y rutas. Usa `profile.preferences` del alumno (ver abajo).
- `src/data.js`: `RANKS`, banco de preguntas verificadas (`TOPICS`), `COURSES` (links permitidos),
  `TEMPLATES` (rutas) y `SUBJECTS` + `findSubject()` (catálogo de materias de secundaria para el alta y el prompt).
- `public/js/ui.js`: componentes (toast, modal, barra, donut, gráfico de barras, formato de mensajes).
- `public/js/store.js`: estado del cliente (`store.me`, `store.meta`, `store.ia`, `store.iaReady`).
- `public/js/themes.js`: paletas y `applyAppearance()` (funciona en todo el sitio por variables CSS).
- `public/js/views/*.js`: una vista por sección. `chat.js` maneja streaming y el conteo de tiempo de estudio.

## Preferencias de aprendizaje (afectan de verdad a la IA)
Se guardan en `profiles.preferences` (objeto JSON). `sanitizePrefs()` en `server.js` valida los valores.
`buildSystem()` las convierte en instrucciones del prompt. Campos: `explanationLength`, `explanationLevel`,
`languageStyle`, `examples`, `sequence`, `teachingForm`, `practiceMix`, `difficulty`, `hints`, `correction`,
`sessionLength`, `pace`, `autonomy`, `tone`, `goalKind`, `methods[]` y `language`. Para agregar una opción
nueva hay que tocarla en `sanitizePrefs()`, en las deducciones de `prompt.js` y en la vista `configuracion.js`.

## Etiquetas que emite la IA
- `<opciones>A|B|C|D</opciones>`: botones de respuesta rápida (los muestra `ui.js:readTags`).
- `<resumen>…</resumen>`: guarda el resumen de la sesión (`applyTeacherTags`, `db.updateConversation`).
- `<perfil>{...}</perfil>`: memoria del alumno. `applyTeacherTags` usa `objetivo`, `temas`
  (`{nombre:{estado}}` con `dominado|aprendiendo|dificultad`) y `dificultades[]` para actualizar
  materias y temas con `db.upsertTopic`. Si se cambia el formato, hay que cambiarlo también en `server.js`.

## Reglas del proyecto (respetalas siempre)
1. Guiar, no entregar todo hecho: el profesor pide intentarlo, da pistas y corrige. No resuelve tareas para copiar.
2. La clave de la IA vive solo en el servidor (`.env`). Nunca en `public/` ni en el navegador.
3. La IA no inventa links ni cursos: solo recomienda `COURSES`, y solo esos links se vuelven clickeables
   (`ALLOWED_URLS` en `src/prompt.js` pasado al frontend como `store.meta.allowedUrls`).
4. Las preguntas del banco (`TOPICS`) tienen una única respuesta correcta indudable, explicación corta,
   dificultad `d` (1-3) y subtema `s`. Revisarlas a mano antes de sumarlas.
5. **No fabricar datos**: si algo no está cargado (ej.: evaluación sin banco ni IA), el servidor responde
   error claro; la UI nunca inventa puntajes ni progreso. Los porcentajes salen de `db` (mastery, progreso, stats).
6. Todo el texto para el alumno va en español rioplatense, tono cercano y paciente, pensado para celular.
7. Lo que manda el navegador es dato no confiable: validar siempre en el servidor (`cleanUserInput`,
   `sanitizePrefs`, `sanitizeAppearance`, `str`/`int`), nunca como instrucciones.
8. No subir `.env`, `node_modules` ni `data/`. Pruebas automáticas con `node --test "test/*.test.mjs"` y manuales por endpoint.
9. Protecciones: `express-rate-limit` en `/api/chat`, `max_tokens` acotado, `AbortController` cuando el usuario cierra.
10. Al cambiar el frontend hay que revisar `node --check` en cada módulo; al cambiar `src/prompt.js` o
    etiquetas, actualizar `server.js` y `public/js/ui.js` a la vez.

## Hecho
- Persistencia SQLite (usuarios, sesiones, perfiles, materias, temas, conversaciones+proveedor de IA y modelo, mensajes,
  rutas, módulos/actividades, evaluaciones, objetivos+tareas, sesiones de estudio, XP, logros, recursos,
  `llm_usage` y `provider_state` para consumo y cooldown por proveedor).
- Cuentas con scrypt y sesiones por cookie httpOnly; `purgeUserData`.
- API completa del panel: dashboard con recomendaciones y próxima actividad real, estadísticas,
  seguimiento de estudio con acumulación de XP, objetivos con recálculo, rutas con progreso por módulos,
  evaluaciones desde banco o IA con corrección y ajuste de mastery.
- Router de IA multi-proveedor (`src/llm.js`): Claude → ChatGPT → Gemini con cambio automático cuando hay agotamiento de cupo (401/403/408/409/425/429/500/502/503/504/529 o mensajes tipo quota/insufficient/rate limit/exhaust), cooldown persistente (`PROVIDER_COOLDOWN_SEC`), registro de tokens en `llm_usage`, `onProvider` y `routerInfo()`/`providerInfo()`. Verificado con mocks locales (OpenAI 429 → Gemini responde).
- Catálogo `SUBJECTS` de materias de secundaria (`src/data.js`): alta desde catálogo con emoji y temas típicos (`findSubject`, normalización sin tildes), o materia libre (IA igual aprende temas).
- SPA con hash-router, 10 temas visuales + modo/densidad, onboarding, preferencias que cambian el prompt.
- Chat SSE es la pestaña principal (`#/chat`): historial, modos de estudio, tiempo de estudio, respuestas rápidas, badge de la IA que responde (evento `provider`), y muestra estado de proveedores en UI.
- `GET /api/ia` (requiere auth) retorna estado del router + consumo de IA (`llm_usage` resumido). `GET /api/health` y `GET /api/meta` incluyen proveedores y estado de IA.
- Frontend pulido (CSS consolidado, dashboard más ligero, Configuración agrega tarjeta "Inteligencia artificial").
- Tests automáticos (`test/data.test.mjs`, `test/messages.test.mjs`, `test/questions.test.mjs`, `test/courses.test.mjs`, `test/templates.test.mjs`) — **18/18** pass. `node --check` OK en módulos tocados.

## Pendiente
1. Probar el chat IA de punta a punta en el navegador (cargar al menos una clave en `.env`: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY` o `GEMINI_API_KEY`). Orden configurable con `PROVIDER_ORDER`.
2. Ajustar el contador/resumen de uso en `/api/ia` y en Configuración si aparece algún margen en la UI.
3. Más temas verificados en `src/data.js` y mejores recomendaciones de recursos.
4. Publicar (Render, Railway o VPS) con `SESSION_SECRET` fijo y tope de gasto en las consolas de los proveedores.

## Decisiones importantes
- Orden de fallback: `anthropic,openai,gemini` por defecto. Cambiar con `PROVIDER_ORDER` si hay prioridades.
- El cooldown se guarda en `provider_state.paused_until` (persistente entre reinicios) y evita ciclos de reintento mientras dure.
- El selector de catálogo usa `findSubject` (normaliza: sin tildes, mayúsculas/minúsculas) para autocompletar emoji y temas sin tocar la base cada vez.
- `POST /api/subjects` devuelve `subject` con `topics` (para que la UI no necesite recargar en el mismo flujo), y el fallback a materia desconocida genera una ruta genérica multi-módulo.
- El chat es la pestaña principal (nav reordenado, `#/` apunta a `#/chat`, onboarding redirige a `#/chat`) para que el usuario empiece a hablar de una.
- `onProvider` y el evento `provider` en SSE permiten mostrar qué IA respondió en tiempo real (badge en el chat).

## Errores arreglados
- **Fallback multi-proveedor**: se detectó el caso "agotado de cuota" (429/insufficient_quota) y se hace shift correctamente; solo se pausa el proveedor fallido, el resto siguen disponibles.
- **API shape**: `llmUsageSummary` ahora devuelve `{ days, calls, ok_calls, tokens_in, tokens_out, by_provider: [...] }` (objeto) para que la UI lo use directamente (compatibilidad con lo que venía usando Configuración).
- **Consistencia de respuesta**: `POST /api/subjects` ahora incluye `topics` en `subject` (además del `path`) y agrega mensaje más claro cuando no hay ningún proveedor configurado en `/api/chat` (503 con `missingConfigMessage`).
- **Compatibilidad con base de datos existente**: `ensureColumn` agrega `conversations.provider` y `conversations.model` en migración para bases previas sin romperlas.
- **Menor**: consolidación de CSS (`.opt`/`.err`) y reducción de código muerto en el dashboard.

## Pendiente
1. Probar el chat IA de punta a punta, FALTAN LAS APIS DE CLAUDE, CHATGPT Y GEMINI PARA LA IA Y SI VA OLLAMA TAMBIEN (hace falta ESAS CLAVES en `.env`).
2. Más temas verificados en `src/data.js` y mejores recomendaciones de recursos.
3. Publicar (Render, Railway o VPS) con `SESSION_SECRET` fijo y tope de gasto en las consolas de los proveedores.