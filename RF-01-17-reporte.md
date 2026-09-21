# Auditoría Técnica Integral RF-01 a RF-17 — Reporte de estado

**Parte A (Fases 1 y 2): análisis estático y corrección.** Ejecutada el 21 de septiembre de 2026 en la máquina de desarrollo.
**Parte B (Fases 3 y 4): túnel y pruebas funcionales end-to-end.** Pendiente, en la máquina de la Casa del Libro Total.

Rama de trabajo en los tres repositorios: `audit/rf-01-17`, creada desde `main` con el árbol limpio.
**Los cambios están en el árbol de trabajo, sin commitear**, a la espera de la revisión manual.

---

## 1. Resumen ejecutivo por capa

| Capa | Estado | Lectura |
|---|---|---|
| `evento-pwa-api` | ⚠️ **Con hallazgos — todos los de build corregidos** | Compila, tipa y pasa lint sin avisos; 363/363 pruebas en verde. Se corrigieron 2 fallos de seguridad de severidad Alta y 1 carrera de concurrencia en el ingreso. Quedan 12 vulnerabilidades de dependencias que solo se cierran subiendo NestJS 11→12. |
| `evento-pwa-worker` | ⚠️ **Con hallazgos — corregidos** | Compila y tipa. Tenía un fallo Alto de disponibilidad (RNF-04) que podía matar el proceso en el primer microcorte de Redis. No tenía ESLint; ahora sí. `@supabase/supabase-js` ya estaba eliminado por completo. |
| `evento-pwa-web` | ✅ **Estable** | Compila, tipa y pasa lint sin un solo aviso. No se modificó ni una línea. Un hallazgo Alto **de despliegue, no de código**: el build necesita salida a internet. |

**Ninguna regresión.** La suite de la API pasó de 362 a 363 pruebas (la que se añadió cubre la carrera corregida en el canje del código).

---

## 2. Resultado de Fase 1 — Análisis estático

### 2.1 `evento-pwa-api`

| Comprobación | Antes | Después |
|---|---|---|
| `npm run build` (`nest build`) | ✅ | ✅ |
| `tsc --noEmit` | 🔴 **3 errores** | ✅ 0 |
| `eslint` (`--max-warnings=0`, sin `--fix`) | 🔴 **1 error** | ✅ 0 |
| `npx prisma validate` | ✅ válido | ✅ válido |
| `npx prisma format` | ⚠️ 12 líneas desalineadas, sin salto final | ✅ aplicado |
| `npm audit` | 🔴 **18 (17 altas, 1 moderada)** | ⚠️ **12 altas** |
| `npx jest` | ✅ 362/362 | ✅ **363/363** |

Sobre **`prisma validate`**: pasa, y eso ya responde al punto de *"ambos lados de cada relación declarados"* — Prisma rechaza el esquema si una relación es unilateral. El `format` solo movió espacios en `EventMemory` y añadió el salto final: **cero impacto sobre la base de datos**, no se generó ni se aplicó ninguna migración.

Sobre **endpoints, DTOs y guards**, revisados uno a uno: la validación global es `whitelist + forbidNonWhitelisted + transform`, todas las rutas de mutación llevan `JwtAuthGuard` y `ProfileCompleteGuard`, el `id` de publicación entra por `ParseUUIDPipe` y el tipo de reacción por `ParseEnumPipe`. La subida identifica el archivo **por su firma binaria**, no por el `mimetype` que declara el cliente, y los cuatro topes (95 MB medio, 4 MB miniatura, 8 MB avatar, 60 s video) se hacen cumplir en el servidor. Swagger solo se publica fuera de producción. **No se encontró ninguna ruta sin protección ni ningún DTO sin validar.**

### 2.2 `evento-pwa-worker`

| Comprobación | Antes | Después |
|---|---|---|
| `npm run build` (`tsc`) | ✅ | ✅ |
| `tsc --noEmit` | ✅ | ✅ |
| `eslint` | 🔴 **no existía configuración** | ✅ 0 errores |
| `npm audit` | ✅ 0 | ✅ 0 |

**`@supabase/supabase-js`: eliminado y verificado.** No está en `package.json`, no aparece en `package-lock.json`, no está en `node_modules` y ninguna línea de `src/` lo importa. Lo único que quedaba eran **dos comentarios** que mandaban a buscar el video en *"Supabase Storage (bucket 'media')"* — corregidos, porque el archivo vive en el disco local que comparte con la API. El criterio de aceptación queda **resuelto: eliminado**.

**Configuración BullMQ/ioredis revisada:** nombres de cola y prefijo coinciden con los de la API y ambos caen al mismo valor por omisión (`evento`); `maxRetriesPerRequest: null` está puesto en los dos lados, como exige BullMQ para un consumidor; `lazyConnect` deja el primer intento en manos del arranque; el `PING` posterior al `connect()` distingue "hay un Redis vivo" de "el puerto acepta conexiones". Los reintentos y el `backoff` exponencial los fija **la API al encolar** (`attempts`, `backoff`, `removeOnFail: false`), que es donde corresponde en BullMQ. El `delay` de los trabajos programados lo usa `MemoryScheduleService` (RF-13) y también vive en la API.

### 2.3 `evento-pwa-web`

| Comprobación | Resultado |
|---|---|
| `npm run build` (Next 16, App Router) | ✅ **con proxy**; 🔴 sin salida a internet (ver RNF-02-014) |
| `tsc --noEmit` | ✅ 0 |
| `eslint .` (`--max-warnings=0`) | ✅ 0 |

**Manifest y service worker:** `app/manifest.ts` genera `/manifest.webmanifest`, y `next.config.ts` le pone `Cache-Control: max-age=0, must-revalidate` a él y `no-store` a `/sw.js` — que es justo lo que evita que una PWA instalada se quede clavada en una versión vieja. El service worker propio se compila desde `app/sw/index.ts` y el build confirmó que lo encontró (`Found a custom worker implementation`).

**Estrategia de caché (RF-08/RF-15):** las 12 reglas están ordenadas de forma correcta —lo que nunca debe servirse de copia (`/auth/`, `/admin`, `/memory/status|generate|export|schedule`, video) va **antes** que las de lectura, que es lo que importa porque Workbox se queda con la primera que encaje. `/memory/status` está deliberadamente por delante de `/memory`. Las fotos van `CacheFirst` con `statuses: [0, 200]` (sin el 0 no se guardaría ni una, por ser respuestas opacas de otro origen) y el cupo está en 40. El video es `NetworkOnly` a propósito, para no llenar el teléfono con 2,5 GB.

**SSR en rutas autenticadas: verificado y correcto.** La tabla de rutas del build marca **las 14 rutas como `○ (Static)`**, incluidas `/feed`, `/admin`, `/profile`, `/memoria`, `/buscar` y `/capture`. Ninguna se renderiza en el servidor por petición, así que el JWT en `localStorage` —decisión intencional— no entra en conflicto con nada. No se tocó.

**Fallback `window.online`: presente y correcto.** `useOutbox` (`app/lib/use-outbox.ts`) dispara el reenvío por **dos** vías: el efecto que depende de `useIsOnline` —que escucha `window.addEventListener("online"/"offline")`— y un oyente de `visibilitychange`, necesario porque en iOS una pestaña en segundo plano se congela y el evento `online` puede no llegar nunca. Se verificó además lo que el propio código advierte que es crítico: **`OutboxIndicator` está montado en `app/(app)/layout.tsx`**, no en una pantalla, de modo que no se desmonta al navegar y el reintento sobrevive a la reconexión.

---

## 3. Resultado de Fase 4 — Pruebas funcionales

**No ejecutada. Corresponde a la Parte B**, en la máquina de la Casa del Libro Total y contra el dominio del túnel. Los cuatro flujos (feed en vivo por Socket.io, push con suscripción caducada 410, memoria digital por Puppeteer y acceso completo desde el dominio público) quedan **pendientes documentados**, no como "no aplica".

Lo que sí se pudo confirmar por lectura de código, y que allá solo habrá que comprobar: el manejo del **410/404** está implementado y probado (`push-sender.service.ts` borra la fila y devuelve `caducado`), y `PUSH_PROXY_URL` permite forzar o saltar el proxy solo para los envíos push.

---

## 4. Backlog priorizado

Severidad: Crítica / Alta / Media / Baja · Estado: Corregido / Pendiente de revisión / No aplica
Columna **Commit**: vacía a propósito — los cambios están sin commitear, a la espera de la revisión manual.

### 4.1 Corregidos

| ID | Capa | Sev. | Descripción | Causa raíz | Estado | Commit |
|---|---|---|---|---|---|---|
| RF-17-004 | api | **Alta** | La API leía `REDIS_URL` sin `trim()` mientras el worker sí lo hace, y el `.env` tenía 4 espacios al final del valor. Un `REDIS_URL` con solo espacios habría dejado a la API encolando trabajos que el worker, negándose a arrancar, no consumiría jamás. | Tres puntos de lectura duplicados (`hayRedis`, `app.module`, `main`) sin un ayudante compartido. | Corregido | — |
| RF-02-005 | api | **Alta** | El limitador de peticiones creía `CF-Connecting-IP` viniera de donde viniera. Cualquiera en el wifi del recinto podía cambiar esa cabecera en cada petición y estrenar contador, anulando el tope de 10/min que es el único freno para adivinar códigos de acceso. | La precondición documentada (`BIND_HOST=127.0.0.1`) no estaba puesta en el `.env` ni forzada por el código; el valor por omisión es `0.0.0.0`. | Corregido | — |
| RNF-04-009 | worker | **Alta** | El `Worker` de BullMQ no escuchaba `'error'`. Al ser un `EventEmitter`, un `'error'` sin oyente mata el proceso de Node: el primer tropiezo de reconexión con Redis dejaba el worker apagado en silencio para el resto del evento. | Solo se registraron `completed` y `failed`. Verificado en el código instalado de BullMQ: emite `'error'` en reconexiones y lecturas de trabajo. | Corregido | — |
| RNF-03-008 | api | **Alta** | 18 vulnerabilidades (17 altas, 1 moderada). | Dependencias desactualizadas. Reducidas a 12 con `npm audit fix` sin `--force`; el resto exige salto mayor (ver RNF-03-013). | Corregido (parcial) | — |
| RF-02-006 | api | Media | El canje del código OTP no era atómico: dos peticiones simultáneas con el mismo código pasaban ambas la comprobación y **ambas abrían sesión**. El código "de un solo uso" servía dos veces. | `update` por id después de una lectura previa; la condición `consumedAt: null` no viajaba dentro de la sentencia. | Corregido | — |
| RF-01-007 | api | Media | El origen CORS por omisión era `http://localhost:3001` —el puerto de la propia API— en lugar del de la web (3002), en `main.ts` y en el gateway de Socket.io. Sin `CORS_ORIGIN` en el `.env`, el navegador bloqueaba todo y la app aparecía vacía sin un solo error en el servidor. | Valor por omisión copiado del puerto equivocado. | Corregido | — |
| RNF-05-011 | worker | Media | El repositorio no tenía **ninguna** configuración de ESLint, así que nada vigilaba promesas sin `await` ni variables muertas en un proceso de fondo donde esos fallos no dan síntoma. | Nunca se añadió. Al estrenarla detectó de inmediato un `async` sin `await`, documentado en su sitio. | Corregido | — |
| RF-16-001 | api | Media | `tsc --noEmit` fallaba: al indexar `mock.calls[0][0]` TypeScript veía una tupla vacía. | `jest.fn()` sin firma tipada en el doble de `addBulk`, que dejaba `mock.calls` como `[]`. | Corregido | — |
| RF-14-002 | api | Media | `tsc --noEmit` fallaba en las fixtures de usuario de dos specs. | No se actualizaron al añadirse la columna `accessCodeHash` del ingreso por código personal. | Corregido | — |
| RF-16-010 | worker | Baja | Dos comentarios situaban el video original en *"Supabase Storage (bucket 'media')"* y uno de ellos, contradictoriamente, "en Postgres". Mandaban a buscar el archivo donde no está. | Resto del borrador anterior a la retirada de Supabase. | Corregido | — |
| RF-04-003 | api | Baja | ESLint: comentario de bloque mal cerrado en `src/posts/upload.ts`. | Quedó así al bajar el tope de 100 MB a 95 MB por el túnel. | Corregido | — |
| RNF-05-012 | api | Baja | `prisma format` dejaba 12 líneas desalineadas en `EventMemory` y faltaba el salto de línea final. | `schema.prisma` editado a mano. | Corregido | — |

### 4.2 Pendientes de revisión

| ID | Capa | Sev. | Descripción | Causa raíz | Estado | Commit |
|---|---|---|---|---|---|---|
| RNF-03-013 | api | **Alta** | 12 vulnerabilidades altas restantes. Solo hay **dos raíces reales**: `multer` (DoS por nombres de campo multipart y por fuga de descriptores en subidas abortadas) y `deepmerge-ts` (dentro de `@prisma/config`, solo en tiempo de CLI). Las otras diez son arrastre transitivo de los paquetes `@nestjs/*`. | Las dos exigen **salto mayor**: NestJS 11→12 y un cambio en la cadena de Prisma. | **Pendiente de revisión** — no se aplicó a propósito: subir el framework entero a días de la operación es exactamente el riesgo de regresión que los criterios de aceptación declaran bloqueante. Mitigación vigente: la subida exige JWT y el borde de Cloudflare corta en 100 MB. | — |
| RNF-02-014 | web | **Alta** | `npm run build` **falla sin salida a internet**: `next/font/google` descarga Inter, Poppins y Bebas Neue de `fonts.googleapis.com` en tiempo de compilación. La Parte B **obligará a recompilar** (las `NEXT_PUBLIC_*` se incrustan en el bundle y deben apuntar al dominio del túnel), justo en la máquina donde el propio manual advierte que el ancho de banda de salida fluctúa. | Fuentes no autoalojadas. | **Pendiente de revisión.** Workaround **verificado en esta máquina**: `HTTPS_PROXY=http://wsaprd.syc.loc:3128` + `NODE_OPTIONS=--use-env-proxy` antes de `npm run build` — Node no toma el proxy por su cuenta. Solución de fondo: pasar a `next/font/local` con los `.woff2` en el repo. | — |
| RF-07-015 | api | Media | `EVENT_START_DATE` está **vacía** en el `.env`. Sin ella, `diaDelEvento()` devuelve `null` para todo y la Memoria del Evento se queda sin su agrupación por día 1/2/3 (RF-07, RF-13). | Variable declarada pero sin valor. El código lo maneja con elegancia —prefiere el hueco honesto a inventar un día— así que **no es un fallo, es configuración que falta**. | **Pendiente (Parte B):** fijar la fecha del primer día en formato `YYYY-MM-DD`. | — |
| RNF-03-016 | api | Media | `BIND_HOST` no está en el `.env`, así que la API escucha en `0.0.0.0` y es alcanzable desde toda la red del recinto, no solo desde el túnel. | Falta en la configuración del despliegue. | **Pendiente (Parte B):** poner `BIND_HOST=127.0.0.1` en la máquina del túnel. El endurecimiento del limitador (RF-02-005) ya cubre el hueco aunque esto se olvide. | — |
| RF-01-017 | web | Baja | `NEXT_PUBLIC_SITE_URL` no está en `.env.local`; el build lo avisa. El `robots.txt` publicado en el dominio del túnel anunciaría `Host: http://localhost:3002`. | Falta en la configuración. No se rellenó a propósito: el valor correcto es el del túnel, y ponerle `localhost` ahora sería fijar el valor equivocado para la Parte B. | **Pendiente (Parte B):** `NEXT_PUBLIC_SITE_URL=https://cumbre.cltapp.site` y recompilar. | — |
| RF-02-018 | api | Baja | `ACCESS_CODE_SECRET` no está en el `.env` y cae a `JWT_SECRET`. Funciona, pero ata las dos vidas: rotar el JWT invalidaría **todos los códigos ya impresos en las acreditaciones**. | Variable opcional no configurada. | **Pendiente (Parte B):** fijarle un valor propio **antes** de acuñar los códigos definitivos. | — |
| RF-04-019 | worker | Baja | La cola `medios` no tiene productor: ninguna pieza de la API encola `miniatura-video`, así que el worker arranca, se conecta y se queda esperando. El propio `procesar` registra "miniatura pendiente de implementar". | Estado conocido y documentado en `queues.ts`. La miniatura de foto sí se genera, pero en la API con `sharp`. | **Pendiente de revisión:** decidir si el worker entra en operación en este evento o queda inerte a la espera de la fase de ffmpeg. | — |
| RF-04-020 | api | Baja | Si `feed.obtenerDeAutor()` fallara tras crearse la publicación, se borran los archivos del disco pero la fila queda en la base apuntando a archivos que ya no existen. | El `catch` que limpia el disco abarca también la lectura posterior a la escritura. | **Pendiente de revisión.** Ventana muy estrecha y sin impacto de seguridad; se deja anotado antes que tocar la ruta más caliente del evento por un caso improbable. | — |

### 4.3 No aplica

| ID | Capa | Sev. | Descripción | Razón técnica | Estado |
|---|---|---|---|---|---|
| RF-16-021 | worker | — | `@supabase/supabase-js` en el worker. | **Ya estaba eliminado** en `main` (commits `eb359d0` y `2f981b2`). Verificado en `package.json`, `package-lock.json`, `node_modules` y `src/`. Solo quedaban comentarios obsoletos, corregidos en RF-16-010. | **No aplica** |
| RF-10-022 | api | Baja | El gateway de Socket.io solo verifica la **firma** del token al conectar: no relee `isActive` ni revalida la caducidad mientras el socket siga abierto. | **Decisión de diseño correcta y documentada.** Por el socket no entra ninguna acción —no hay un solo `@SubscribeMessage`— y lo único que sale son publicaciones ya aprobadas y visibles en el feed. Toda autorización real vive en HTTP, donde `JwtStrategy` sí consulta la base en cada petición y aplica la baja lógica del RF-14. | **No aplica** |
| RF-08-023 | web | — | JWT en `localStorage` y ausencia de SSR en rutas autenticadas. | Fuera de alcance por indicación expresa: decisión intencional. Verificado como coherente —las 14 rutas son estáticas— y **no se tocó**. | **No aplica** |

---

## 5. Criterios de aceptación

| Criterio | Estado |
|---|---|
| Los 3 repos compilan y pasan lint/type-check sin errores | ✅ **Cumplido.** API: build ✅ · tsc 0 · eslint 0 · 363/363. Worker: build ✅ · tsc 0 · eslint 0 (configuración creada). Web: build ✅ · tsc 0 · eslint 0. |
| Los 4 flujos críticos de Fase 4 funcionan por el dominio del túnel | ⏳ **Parte B.** Fuera de alcance en esta ejecución, documentado como pendiente. |
| Todo hallazgo Crítico o Alto corregido, documentado o justificado | ✅ **Cumplido.** Cero hallazgos Críticos. Cuatro Altos corregidos (RF-17-004, RF-02-005, RNF-04-009, RNF-03-008 parcial). Dos Altos pendientes con justificación técnica escrita (RNF-03-013, RNF-02-014). |
| El estado de `@supabase/supabase-js` en el worker queda resuelto | ✅ **Cumplido: eliminado**, verificado en las cuatro superficies, y los comentarios que lo sobrevivían, corregidos. |
| Ningún fix rompe RF-01 a RF-17 (regresión = bloqueante) | ✅ **Cumplido.** 363/363 pruebas en verde (362 antes, +1 nueva). Durante la auditoría se introdujo y se resolvió un desajuste de versiones de NestJS provocado por `npm audit fix`, detectado precisamente por la suite. |

---

## 6. Nota sobre las dependencias, para la revisión

`npm audit fix` (sin `--force`) subió `@nestjs/core` de 11.1.28 a 11.2.5 **dejando `@nestjs/common` en 11.1.28**. La combinación compila sin una sola queja pero revienta en tiempo de ejecución (`Cannot find module '.../sse-signal.decorator'`, un decorador que solo existe desde 11.2). Lo delataron 11 suites de pruebas, no el compilador.

Se alineó toda la familia `@nestjs/*` en **11.2.5** y la pareja `prisma` / `@prisma/client` en **6.19.3** — que `audit fix` también había desparejado, dejando el CLI en 6.19.3 contra un cliente 6.14.0. Ambos casos quedan como aviso para la revisión: **en este proyecto, `npm audit fix` desempareja familias de paquetes**, y conviene ejecutar la suite completa después de tocar dependencias.

Cambios en `package.json` de la API: `@nestjs/common` `^11.0.1`→`^11.2.5`, `prisma` y `@prisma/client` `^6.14.0`→`^6.19.3`. En el worker: se añadieron `eslint`, `typescript-eslint`, `@eslint/js` y `globals` como dependencias de desarrollo, y el script `lint`.

---

## 7. Cómo reproducir esta verificación

```bash
# API
cd evento-pwa-api
npx tsc --noEmit -p tsconfig.json
npx eslint "{src,apps,libs,test}/**/*.ts" --max-warnings=0
npm run build && npx jest
npx prisma validate

# Worker
cd ../evento-pwa-worker
npx tsc --noEmit && npm run lint && npm run build

# Web — el proxy es obligatorio: next/font descarga de Google al compilar
cd ../evento-pwa-web
export HTTPS_PROXY=http://wsaprd.syc.loc:3128
export NODE_OPTIONS=--use-env-proxy
npx tsc --noEmit && npx eslint . --max-warnings=0 && npm run build
```
