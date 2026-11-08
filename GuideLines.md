# CLAUDE.md — Rutas por calle (heatmap personal con consulta por calle)

> Guía para Claude Code. Léela completa antes de escribir código. Las reglas marcadas como **MUST** no son negociables; si una tarea parece requerir romperlas, detente y pregunta.

---

## 1. Qué estamos construyendo

Una aplicación web personal que:

1. Importa todas mis actividades (principalmente ciclismo) desde **exportaciones de Garmin (.FIT)**, archivos **GPX** y, de forma secundaria, desde la **API de Strava**.
2. Las muestra en un **mapa tipo heatmap personal**.
3. Al hacer **click en cualquier calle**, muestra **todas las actividades que pasaron por ahí**: las resalta en el mapa y las lista con fecha, distancia, desnivel y enlace a la actividad original.

Usuario principal: un solo atleta (yo). Diseñar la arquitectura multiusuario (por `user_id`) desde el inicio, pero **sin** compartir datos entre usuarios.

### Principios del producto
- **El mapa es el producto.** La UI flota sobre el mapa y nunca compite con él.
- **La privacidad de ubicación es la prioridad de seguridad #1.** Un heatmap revela dónde vive y trabaja una persona.
- **La velocidad es parte del diseño.** El click debe sentirse instantáneo (objetivo p95 < 150 ms en el endpoint de consulta).
- **Garmin primero, Strava después.** Los datos propios no deben depender de las políticas de un tercero.

---

## 2. Stack

| Capa | Tecnología | Notas |
|---|---|---|
| Base de datos | PostgreSQL 16 + PostGIS 3 | Índices GIST, `ST_DWithin`, `ST_AsMVT`, Row-Level Security |
| Backend | ASP.NET Core 8+ (minimal APIs), C# | `Npgsql` + `NetTopologySuite`, `Dapper` o EF Core con NTS |
| Parsing FIT | Garmin FIT SDK oficial (C#) | No usar parsers de terceros sin aprobación |
| Parsing GPX | `System.Xml` con DTD deshabilitado | Ver §5.5 |
| Jobs en segundo plano | `BackgroundService` + `System.Threading.Channels` | Hangfire solo si se requieren reintentos persistentes |
| Vector tiles | Martin (servidor de tiles de PostGIS) | Siempre detrás de la API autenticada (proxy), nunca expuesto |
| Identidad | ASP.NET Core Identity (email + contraseña, TOTP 2FA) | Registro cerrado; el dueño se crea por CLI. Strava es una conexión de datos, NO un método de login (§5.3) |
| Frontend | Vite + TypeScript (strict) + Svelte 5, SPA (sin SvelteKit) | **Decidido.** No usar React ni mezclar frameworks |
| Mapa | MapLibre GL JS, build CSP (`maplibre-gl-csp.js` + worker propio) | No usar Leaflet. No usar tiles raster de OSM por defecto |
| Mapa base | Protomaps (PMTiles autoalojado) + estilo propio | Estilo diseñado a medida (§7.2) |
| Map matching (fase 7) | Valhalla en Docker con extracto de México (Geofabrik) | Opcional |
| Infraestructura | Docker Compose + Caddy (HTTPS automático) | Un VPS; sin servicios administrados innecesarios |
| Tests | xUnit + Testcontainers (Postgres/PostGIS reales), Vitest, Playwright | Nada de mocks de PostGIS para lógica espacial |

**Versiones:** fija versiones exactas (lockfiles commiteados: `packages.lock.json`, `pnpm-lock.yaml`). Usa `pnpm` en el frontend.

---

## 3. Estructura del repositorio

```
/
├── CLAUDE.md
├── docker-compose.yml
├── docker-compose.override.example.yml
├── .env.example                 # Solo nombres de variables, NUNCA valores reales
├── db/
│   ├── migrations/              # SQL versionado (DbUp o similar), idempotente
│   └── functions/               # Funciones PostGIS para Martin (tiles por usuario)
├── src/
│   ├── Api/                     # ASP.NET Core: endpoints, auth, proxy de tiles
│   ├── Core/                    # Dominio: Activity, PrivacyZone, servicios puros
│   ├── Infrastructure/          # Postgres, Strava client, FIT/GPX parsers, cifrado
│   └── Worker/                  # Importación, sincronización, webhooks
├── web/                         # Frontend Vite + Svelte + MapLibre
│   ├── src/lib/map/             # Capas, estilos, interacciones
│   ├── src/lib/design/          # Tokens, tipografía, componentes base
│   └── public/styles/           # Estilo del mapa base (JSON)
├── tests/
│   ├── Api.Tests/
│   ├── Core.Tests/
│   ├── Integration.Tests/       # Testcontainers + PostGIS
│   └── e2e/                     # Playwright
└── fixtures/                    # Actividades SINTÉTICAS para desarrollo y tests
```

---

## 4. Modelo de datos (mínimo)

```sql
users(id uuid pk, created_at, display_name)

oauth_connections(
  id uuid pk, user_id uuid fk, provider text,          -- 'strava'
  athlete_id bigint, scopes text[],
  access_token_enc bytea, refresh_token_enc bytea,     -- CIFRADOS (§5.2)
  expires_at timestamptz, created_at, updated_at
)

-- Actividad CANÓNICA: una rodada real = una fila, aunque llegue por varias fuentes
activities(
  id uuid pk, user_id uuid fk,
  primary_source_id uuid fk -> activity_sources,  -- la fuente cuya geometría se usa
  name text, sport_type text, started_at timestamptz,
  distance_m real, elevation_gain_m real, moving_time_s int,
  geom_raw geometry(LineString, 4326),     -- PRIVADA: nunca sale del servidor (§5.1)
  geom_public geometry(LineString, 4326),  -- ya recortada por privacidad; base de TODO lo que sale
  geom_render geometry(LineString, 4326),  -- geom_public simplificada para tiles
  geom_thumb geometry(LineString, 4326),   -- geom_public muy simplificada (≤ 100 vértices) para miniaturas
  elevation_profile real[],                -- 64 muestras equidistantes; null si la fuente no trae altitud
  privacy_version int not null,            -- versión de zonas con la que se calculó geom_public
  created_at, updated_at
)
create index on activities using gist(geom_public);
create index on activities using gist(geom_render);
create index on activities(user_id, started_at);

-- Cada fuente que aportó la actividad (Garmin, Strava, GPX). Idempotencia por fuente.
activity_sources(
  id uuid pk, activity_id uuid fk, user_id uuid fk,
  source text,                 -- 'garmin_fit' | 'gpx' | 'strava'
  source_id text,              -- id externo o hash SHA-256 del archivo
  resolution_rank int,         -- 1 FIT > 2 Strava streams > 3 GPX > 4 summary_polyline
  attribution text,            -- p. ej. 'garmin' si requiere atribución
  external_url text,           -- p. ej. URL de Strava para "View on Strava"
  imported_at timestamptz,
  unique(user_id, source, source_id)
)

privacy_settings(
  user_id uuid pk,
  setup_completed_at timestamptz,          -- null = onboarding de privacidad pendiente (§5.1)
  trim_start_end_m int not null default 400,
  version int not null default 1           -- se incrementa con cada cambio de zonas o trim
)

privacy_zones(id uuid pk, user_id uuid fk, center geography(Point), radius_m int, offset_seed int, created_at)

import_jobs(id uuid pk, user_id uuid fk, status text, total int, processed int, error text, created_at, updated_at)
```

- Habilita **RLS** en `activities`, `privacy_zones`, `oauth_connections`, `import_jobs` con la política `user_id = current_setting('app.user_id')::uuid`. La API fija `app.user_id` por transacción.
- RLS también en `activity_sources` y `privacy_settings`.
- **Dos mecanismos de deduplicación distintos (no confundirlos):**
  1. **Idempotencia dentro de una fuente:** la restricción `unique(user_id, source, source_id)` en `activity_sources`. Reimportar el mismo archivo o recibir el mismo webhook no crea nada nuevo.
  2. **Unificación entre fuentes** (misma rodada vía Garmin y vía Strava): lógica en el Worker. Candidatos = mismo usuario, `started_at` ±2 min y distancia ±3 %. Se confirma si `ST_HausdorffDistance` entre las geometrías (en metros, proyectando a UTM) es < 50 m. Si coincide, la nueva fuente se **enlaza** a la actividad existente como otra fila de `activity_sources` (no se descarta: así se conserva el enlace "View on Strava" aunque la geometría venga de Garmin). La geometría canónica es la de menor `resolution_rank`.
  - Un candidato que cumple tiempo y distancia pero falla Hausdorff se guarda como actividad separada y se registra en logs (sin coordenadas) para revisión.

---

## 5. Seguridad — reglas MUST

### 5.1 Privacidad de ubicación
- **Fail-closed:** mientras `privacy_settings.setup_completed_at` sea null, la API **MUST** responder `409` con `{"code":"privacy_setup_required"}` en tiles, click, exportaciones y miniaturas. El frontend lleva al onboarding de privacidad. Completar el onboarding sin zonas solo es posible confirmando explícitamente "no quiero zonas".
- **Recorte por defecto siempre activo:** se eliminan los primeros y últimos `trim_start_end_m` (por defecto 400 m) de **cada** actividad, aunque no haya zonas. Bajarlo a 0 requiere una acción explícita en la configuración.
- **Geometría materializada:** el recorte (trim + `ST_Difference` con los buffers de las zonas) se calcula en el Worker y se guarda en `geom_public`. **Todo** lo que sale del servidor (tiles, click, exportaciones, miniaturas) se deriva de `geom_public`, `geom_render` o `geom_thumb`; **MUST NOT** usar `geom_raw` en ninguna consulta expuesta. `geom_raw` solo se usa para deduplicar y recalcular.
- Al cambiar zonas o trim: se incrementa `privacy_settings.version` y un job recalcula las geometrías derivadas. Mientras haya actividades con `privacy_version` desactualizada, esas actividades **no se sirven** (fail-closed también aquí).
- **MUST** aplicar el recorte en el servidor, nunca en el cliente.
- Para evitar triangulación: radio mínimo de 300 m, y el centro del buffer se desplaza aleatoriamente (estable por zona) entre 50 y 150 m del punto real.
- **MUST** filtrar cada tile por el `user_id` de la **sesión**. Nunca aceptar `user_id` como parámetro del cliente. Las URLs `/{z}/{x}/{y}` son adivinables (riesgo IDOR).
- **MUST NOT** registrar coordenadas, polylines ni geometrías en logs, trazas o mensajes de error.
- Martin escucha solo en la red interna de Docker. La API hace de proxy, valida la sesión y pasa el `user_id` como argumento de la función SQL.

### 5.2 Secretos y tokens
- **MUST NOT** commitear secretos. `.env` está en `.gitignore`; solo existe `.env.example` con nombres de variables.
- Los tokens OAuth se cifran en reposo con **ASP.NET Core Data Protection** (llaves persistidas fuera del repo y protegidas) o con AES-256-GCM y una llave en un secret manager.
- **MUST NOT** enviar tokens al frontend, a localStorage ni a logs.
- Strava puede devolver un **refresh token nuevo** en cada renovación. **MUST** persistirlo siempre de forma atómica.
- Las API keys de proveedores de mapas se restringen por dominio (`Referer`).

### 5.3 Autenticación y sesión
- **Identidad local (fase 1):** ASP.NET Core Identity con email + contraseña.
  - Hash por defecto de Identity (PBKDF2), con iteraciones al valor recomendado actual o mayores.
  - Contraseña mínima de 12 caracteres; sin reglas de composición forzadas.
  - **TOTP 2FA** disponible desde la fase 1, obligatorio para el usuario dueño.
  - Lockout tras 5 intentos fallidos (15 min).
  - **Registro público deshabilitado.** El usuario dueño se crea con un comando de CLI (`dotnet run --project src/Api -- create-owner`) que pide los datos de forma interactiva y nunca los acepta como argumentos (para que no queden en el historial de la shell).
  - Mensajes de error de login genéricos: no revelar si el email existe.
- **Strava es una conexión de datos, no un método de login.** Se conecta desde la configuración de un usuario ya autenticado (fase 5).
- OAuth 2.0 con Strava: **authorization code flow** en el servidor, con el parámetro `state` (anti-CSRF) ligado a la sesión y de un solo uso.
- Scopes mínimos: `activity:read`; `activity:read_all` solo si el usuario lo activa explícitamente.
- Cookies de sesión con `HttpOnly`, `Secure`, `SameSite=Lax`, rotación al iniciar sesión y expiración por inactividad.
- Protección anti-CSRF en todos los endpoints que mutan estado (antiforgery de ASP.NET Core).

### 5.4 Webhooks de Strava
- Valida la suscripción con `hub.verify_token` (valor aleatorio guardado en una variable de entorno).
- Los eventos **no vienen firmados**: trátalos solo como aviso. **MUST** volver a pedir la actividad al API con el token del usuario; nunca persistir el payload directamente.
- Responde `200` en menos de 2 s y encola el trabajo.
- El evento de desautorización (`athlete` + `authorized: false`) **MUST** borrar los tokens y todos los datos con origen Strava de ese usuario.

### 5.5 Carga de archivos (FIT, GPX, ZIP)
- Límites: 25 MB por archivo y 2 GB por ZIP de exportación, configurables.
- **Zip bomb:** antes de extraer, valida el tamaño descomprimido total (máx. 10 GB), la razón de compresión y el número de entradas (máx. 50 000).
- **Zip slip:** rechaza entradas con `..`, rutas absolutas o symlinks; normaliza y verifica que la ruta final quede dentro del directorio temporal.
- **XXE en GPX:** `XmlReaderSettings { DtdProcessing = DtdProcessing.Prohibit, XmlResolver = null }`.
- Valida el tipo por contenido (magic bytes), no por extensión.
- Procesa en el Worker, nunca en el hilo del request, y borra los archivos temporales al terminar, incluso si hay error (`try/finally`).
- Valida las coordenadas parseadas: lat ∈ [-90, 90], lng ∈ [-180, 180]; descarta puntos con saltos físicamente imposibles (> 200 km/h entre puntos).

### 5.6 API
- **MUST** usar solo queries parametrizadas. Nada de SQL concatenado, tampoco en funciones dinámicas.
- Valida todos los inputs: rangos de lat/lng, tolerancia del click limitada a [5, 50] m, paginación con un tope.
- Rate limiting (`Microsoft.AspNetCore.RateLimiting`): click 30/min, import 5/hora, auth 10/min por IP.
- Los errores al cliente son genéricos (ProblemDetails sin stack traces); el detalle va a los logs del servidor.
- Logging estructurado (Serilog) con redacción de campos sensibles: `token`, `code`, `polyline`, `lat`, `lng`, `geom`.

### 5.7 Headers y transporte
- HTTPS obligatorio (Caddy), con HSTS `max-age=31536000; includeSubDomains`.
- **CSP estricta, sin `unsafe-inline` ni `unsafe-eval`:**
  ```
  default-src 'self';
  script-src 'self';
  style-src 'self';
  img-src 'self' data: blob:;
  worker-src 'self';
  connect-src 'self';
  font-src 'self';
  object-src 'none';
  base-uri 'none';
  form-action 'self';
  frame-ancestors 'none';
  ```
  - Usa el **build CSP de MapLibre** (`maplibre-gl-csp.js`) y sirve su worker desde el propio origen (`setWorkerUrl`). Así no hace falta `blob:` en `worker-src`.
  - Carga `maplibre-gl.css` como hoja estática. Los estilos que MapLibre y Svelte aplican vía propiedades del DOM (`element.style.x = …`) no están sujetos a `style-src`; las etiquetas `<style>` inline y los atributos `style` en el HTML sí lo están.
  - Vite debe extraer todo el CSS a archivos (sin inyección de `<style>` en producción).
  - Durante el desarrollo, activa primero `Content-Security-Policy-Report-Only` con un endpoint de reporte. Un test de Playwright **MUST** fallar si aparece cualquier violación de CSP en el flujo principal.
  - Si una dependencia exige estilos inline: primero busca una alternativa; si no la hay, usa un **hash** o **nonce** para ese caso concreto, documentado en §12. **Nunca** `unsafe-inline` global.
- `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin` y `Permissions-Policy` con geolocalización deshabilitada salvo que se use.
- CORS: solo el origen propio, sin comodines.
- Fuentes e iconos **autoalojados** (sin Google Fonts en runtime), por privacidad y por CSP.

### 5.8 Datos del usuario
- Botón **"Borrar todos mis datos"** que borra de verdad (actividades, tokens, zonas, jobs), con confirmación explícita.
- Exportación de los datos propios en GeoJSON, con las zonas de privacidad **ya aplicadas**.
- Respaldos de Postgres cifrados y con retención definida.

### 5.9 Cadena de suministro
- Dependencias mínimas; justifica en el PR cada paquete nuevo.
- `dotnet list package --vulnerable` y `pnpm audit` en CI; Dependabot/Renovate activado.
- Imágenes Docker con tag y digest fijos; contenedores sin root y con sistema de archivos de solo lectura donde aplique.

---

## 6. Cumplimiento con Strava (MUST)

- Los datos de Strava **solo se muestran al usuario que los aportó**. No construir funciones sociales ni públicas con datos de Strava.
- **MUST NOT** usar datos del API de Strava en modelos o aplicaciones de IA (prompts, embeddings, RAG), ni siquiera derivados o anonimizados. Consecuencia para este repo: **nunca pegar datos reales de Strava en prompts, issues o conversaciones con asistentes.** Usar siempre `fixtures/` sintéticos.
- El enlace a la actividad original lleva exactamente el texto **"View on Strava"**.
- Los logos "Powered by / Compatible with Strava" se usan sin modificar, alterar ni animar, y sin sugerir afiliación ni patrocinio de Strava.
- Si una actividad viene de un dispositivo Garmin vía Strava, se muestra la atribución a Garmin según sus lineamientos (campo `attribution`).
- El base URL del API es **configurable** (`STRAVA_API_BASE_URL`, valor por defecto `https://www.strava.com/api/v3`). Strava anunció un nuevo base URL, `https://api-v3.strava.com`, **disponible a partir del 4 de enero de 2027**, con una ventana de migración antes de una fecha límite. Al iniciar la fase 5, **MUST** verificar el changelog oficial (developers.strava.com) para confirmar la URL vigente y la fecha límite, y actualizar este punto.
- Los tokens van **siempre en el header `Authorization: Bearer`**, nunca en el query string (requisito explícito de los cambios de 2027).
- Respeta los rate limits leyendo los headers de uso, con backoff exponencial y una cola; para el historial, prefiere el listado de actividades con `summary_polyline` (200 por página).

---

## 7. Diseño — moderno, estético y NO genérico

### 7.1 Anti-patrones prohibidos
Estas son las señales típicas de una "app hecha por IA". **No las uses:**
- Cuadrícula de cards con sombra suave, borde redondeado de 12 px e ícono arriba a la izquierda.
- Gradientes morados, azul-violeta o "aurora" de fondo.
- Glassmorphism generalizado (blur en todo).
- Inter/system-ui como única tipografía sin decisión consciente.
- Emoji como íconos.
- Sidebar fija de dashboard con un menú de 8 secciones.
- Landing con hero centrado, título enorme y dos botones.
- Pin azul por defecto, popups default de la librería, tiles de OSM sin estilizar.
- El naranja de Strava como color principal (además de genérico, sugiere afiliación).
- Animaciones decorativas que no comunican nada; skeletons grises en todo.
- Copy genérico: "Welcome back!", "No data found", "Oops! Something went wrong".

### 7.2 Mapa base (la decisión de diseño más importante)
- Estilo propio en Maputnik sobre Protomaps, guardado en `web/public/styles/`. Dos variantes: `dark` (por defecto) y `light`.
- Fondo de bajo contraste y desaturado; las líneas del usuario son las protagonistas.
- POIs ocultos salvo los útiles para ciclismo (fuentes de agua, tiendas, talleres), y solo a zoom alto.
- Etiquetas de calles discretas, que aparecen de forma progresiva con el zoom.
- Agua y parques insinuados con 1–2 % de diferencia de luminosidad.
- **Elemento de identidad:** curvas de nivel finas o un hillshade sutil. La topografía es parte de cada rodada.

### 7.3 Color
- Define todos los colores como tokens en **OKLCH** (`web/src/lib/design/tokens.css`).
- Paleta: neutros ligeramente cálidos o fríos (decide uno, no gris puro), **un** color de acento para la selección y una rampa para el heatmap.
- Rampa del heatmap perceptualmente uniforme (luminosidad creciente y monótona) de oscuro a brillante, con **blending aditivo**: el "calor" debe salir de la superposición real de rutas.
- Valida la rampa y el acento con simuladores de deuteranopia, protanopia y tritanopia.
- Contraste WCAG AA mínimo en todo el texto, incluido el texto sobre el mapa (usa halos o placas sólidas).

### 7.4 Tipografía
- Dos familias como máximo: una sans con carácter para la interfaz y títulos (elegir conscientemente y documentar el porqué en `design/README.md`) y una monoespaciada para datos y coordenadas.
- Números siempre con `font-variant-numeric: tabular-nums`.
- Escala tipográfica modular (p. ej., razón 1.25) definida en tokens; nada de tamaños sueltos.
- Fuentes autoalojadas en WOFF2 con subset latino.

### 7.5 La interacción del click (momento estrella)
1. Click → aparece un **círculo pulsante del tamaño real de la tolerancia** (p. ej., 15 m en metros reales, escalado con el zoom). Comunica que la búsqueda es aproximada.
2. El resto del heatmap se atenúa al 10–15 % de opacidad.
3. Las rutas coincidentes se pintan con el color de acento y se **dibujan animadas** con `line-gradient` + `line-progress`, en orden cronológico y escalonadas.
4. El resaltado usa `feature-state` (sin recrear capas ni fuentes).
5. `Esc` o click en un área vacía limpia la selección con una transición inversa.

### 7.6 Panel de resultados (narrativa, no tabla)
- El encabezado cuenta la historia: *"Has pasado por aquí 47 veces · la primera en marzo de 2021 · la última hace 3 días"*.
- Lista agrupada por año → mes. Cada item muestra:
  - una miniatura de la silueta de la ruta: el endpoint del click devuelve `geom_thumb` como polyline codificada (≤ 100 vértices) y el **cliente** la dibuja como `<path>` SVG en un componente Svelte. No se guarda SVG en la base de datos;
  - una sparkline del perfil de elevación a partir de `elevation_profile` (64 muestras calculadas al importar). Si es null, no se muestra la sparkline (sin placeholder falso),
  - fecha, distancia y desnivel,
  - el enlace "View on Strava" si aplica.
- **Vinculación en ambos sentidos:** pasar el cursor o enfocar un item resalta su ruta en el mapa, y viceversa.
- Móvil: **bottom sheet** con anclajes al 25 / 60 / 100 % y gestos de arrastre.
- Escritorio: panel flotante desacoplado del borde, redimensionable; nunca una sidebar fija.
- **Filtro temporal:** histograma de frecuencia de actividades con brush para elegir un rango de fechas; filtra el mapa y los resultados.

### 7.7 Primeros usos y estados
- **La importación es un momento:** mientras se procesa, las rutas aparecen en el mapa en orden cronológico y el historial "se dibuja" solo. Un contador discreto muestra el progreso.
- Estados vacíos con voz propia y una acción clara (p. ej., "Aquí todavía no has rodado. ¿Importamos tu historial de Garmin?").
- Errores honestos y accionables, sin culpar al usuario.

### 7.8 Detalles que dan oficio
- Barra de escala, indicador de norte cuando el mapa está rotado y coordenadas del punto clickeado en monoespaciada.
- Formatos con `Intl` en `es-MX`: km con un decimal, metros de desnivel, 24 h, fechas como "8 nov 2026".
- Íconos de un solo set con trazo consistente (p. ej., Lucide o Phosphor, uno solo), o propios para los elementos clave (marcador del click, zonas de privacidad).
- Marcadores y popups **diseñados a medida**, nunca los default de MapLibre.

### 7.9 Movimiento
- Duraciones de 150–250 ms para la UI y de 400–1200 ms solo para el trazado de rutas.
- Curvas `cubic-bezier` definidas como tokens (sin `ease` por defecto).
- **MUST** respetar `prefers-reduced-motion`: sin trazado animado ni pulso; cambios instantáneos.

### 7.10 Accesibilidad
- Alternativa al click en el mapa: **buscador de calles por nombre** que produce el mismo resultado.
- Navegación completa por teclado con `focus-visible` diseñado (no el outline default ni ausente).
- `aria-live="polite"` para anunciar el número de resultados.
- Áreas táctiles de 44 × 44 px como mínimo.
- Probar con lector de pantalla (VoiceOver/NVDA) en el flujo principal.

### 7.11 Rendimiento percibido
- Heatmap a zoom bajo desde tiles agregadas; líneas individuales solo a zoom ≥ 13.
- Grosor de línea interpolado por zoom.
- Click: consulta en PostGIS con `ST_DWithin` sobre `geography` y un índice; objetivo p95 < 150 ms.
- Bundle inicial < 250 KB gzip sin contar MapLibre; MapLibre en un chunk separado.

---

## 8. Buenas prácticas de desarrollo

### Código
- C#: `<Nullable>enable</Nullable>`, `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`, analizadores (.NET analyzers + StyleCop o similar) y `.editorconfig` commiteado.
- TypeScript con `strict: true` y `noUncheckedIndexedAccess: true`; ESLint + Prettier.
- Dominio puro en `Core/`, sin dependencias de infraestructura. Las operaciones espaciales complejas viven en SQL/PostGIS, no en C#.
- Nada de lógica de negocio en los endpoints: endpoint → servicio → repositorio.
- Inyección de dependencias en todo; sin estáticos con estado.
- `CancellationToken` propagado en todo el I/O.
- Idempotencia: reimportar el mismo archivo o recibir el mismo webhook dos veces no duplica datos.

### Git
- Ramas por feature y commits con Conventional Commits (`feat:`, `fix:`, `sec:`, `chore:`…).
- Commits pequeños y enfocados. Nunca un commit que mezcle refactor con feature.
- Hook de pre-commit con detección de secretos (gitleaks).

### Tests (definición mínima)
- Unitarios para el parsing de FIT/GPX con fixtures sintéticos, la deduplicación y el recorte de zonas de privacidad.
- Integración con **Testcontainers + PostGIS real** para la consulta del click, la generación de tiles y RLS.
- **Tests de seguridad obligatorios:**
  - Un usuario A no puede obtener tiles ni resultados del usuario B (IDOR).
  - Las zonas de privacidad nunca aparecen en tiles, en resultados del click ni en exportaciones.
  - Zip slip, zip bomb y XXE rechazados.
  - Los tokens no aparecen en logs ni en respuestas.
- E2E con Playwright: importar, ver el heatmap, click en una calle, ver resultados y abrir "View on Strava".
- Pruebas visuales (capturas) del flujo del click en tema claro y oscuro, en móvil y escritorio.

### CI
Build → lint → tests → audit de dependencias → escaneo de secretos → construcción de imágenes. Un PR no se fusiona con CI en rojo.

---

## 9. Fases de desarrollo

1. **Fundaciones:** Docker Compose (Postgres+PostGIS, API, Caddy), migraciones con todas las tablas de §4, RLS, headers de seguridad y CSP (§5.7), identidad local con ASP.NET Core Identity + TOTP + comando `create-owner` (§5.3), scaffolding de `web/` en Svelte 5, `.env.example`.
2. **Privacidad e importación Garmin/GPX:** primero `privacy_settings`, zonas, trim y el onboarding de privacidad con fail-closed (§5.1); después la carga del ZIP del export con todas las protecciones (§5.5), parsing en el Worker, cálculo de `geom_public`/`geom_render`/`geom_thumb`/`elevation_profile` y los dos mecanismos de deduplicación (§4). **Ningún endpoint que exponga geometría puede existir antes de que el fail-closed esté implementado y probado.**
3. **Mapa y heatmap:** estilo de mapa base propio, Martin detrás del proxy autenticado, tokens de diseño, capa del heatmap.
4. **Click por calle:** endpoint con `ST_DWithin`, interacción completa (§7.5), panel de resultados (§7.6), buscador de calles.
5. **Strava:** OAuth, sincronización del historial, webhooks, desautorización, atribuciones y cumplimiento (§6).
6. **Pulido:** filtro temporal, importación animada, accesibilidad completa, rendimiento, pruebas visuales.
7. **(Opcional) Map matching con Valhalla:** guardar `way_id` recorridos y hacer que el click seleccione "la calle" exacta en lugar de un buffer.

Cada fase termina con sus tests en verde y una demo funcional.

---

## 10. Definición de terminado (por tarea)

- [ ] Cumple todas las reglas MUST aplicables de §5 y §6.
- [ ] Tests nuevos o actualizados, incluyendo casos de seguridad cuando aplica.
- [ ] Sin warnings; lint y formato pasan.
- [ ] Ningún secreto, token, coordenada real ni dato real de Strava en código, fixtures, logs o commits.
- [ ] La UI respeta los tokens de diseño, no introduce anti-patrones de §7.1 y funciona en móvil, teclado y `prefers-reduced-motion`.
- [ ] Textos de interfaz en español (es-MX), con voz propia.
- [ ] Documentación actualizada si cambió la arquitectura, un endpoint o una variable de entorno.

---

## 11. Cómo trabajar conmigo (instrucciones para Claude Code)

- Antes de implementar una fase, propón un plan breve y espera confirmación.
- Si una decisión afecta la seguridad o la privacidad, explícala en una línea en el PR o mensaje.
- Si una librería nueva parece necesaria, pregunta primero y justifica la elección.
- Nunca desactives una validación, test o regla de seguridad "temporalmente" para avanzar. Si algo bloquea, detente y pregunta.
- Para la interfaz, muestra capturas (Playwright) del resultado antes de dar por terminada una pantalla.
- Cuando se tome una decisión de arquitectura, librería o diseño, regístrala en §12 con fecha y motivo.

---

## 12. Registro de decisiones

| # | Fecha | Decisión | Motivo |
|---|---|---|---|
| D1 | 2026-11-08 | La identidad es local (ASP.NET Core Identity + TOTP, registro cerrado, dueño por CLI). Strava es solo conexión de datos. | La app no debe depender de Strava para existir; el import de Garmin funciona sin Strava. |
| D2 | 2026-11-08 | Frontend en Svelte 5 como SPA con Vite, sin SvelteKit. React descartado. | El backend es .NET; no se necesita SSR. Svelte genera bundles pequeños y CSS extraíble (CSP). |
| D3 | 2026-11-08 | Privacidad fail-closed: sin onboarding de privacidad no se sirve geometría; trim de 400 m por defecto; geometría pública materializada y versionada. | Evitar cualquier ventana en la que se filtre la ubicación real. |
| D4 | 2026-11-08 | CSP sin `unsafe-inline`: build CSP de MapLibre, worker en el mismo origen, CSS extraído; excepciones solo con hash/nonce documentado. | Cerrar la contradicción entre "CSP estricta" y `unsafe-inline`. |
| D5 | 2026-11-08 | Modelo de actividad canónica + `activity_sources`. Idempotencia por fuente (restricción única) separada de la unificación entre fuentes (tiempo + distancia + Hausdorff). | Son dos problemas distintos; enlazar fuentes conserva "View on Strava" y la mejor geometría. |
| D6 | 2026-11-08 | `STRAVA_API_BASE_URL` configurable con valor por defecto actual; verificar la migración a `api-v3.strava.com` al iniciar la fase 5. | El anuncio indica disponibilidad desde enero de 2027 con ventana de migración. |
| D7 | 2026-11-08 | Miniaturas renderizadas en el cliente desde `geom_thumb`; perfil de elevación de 64 muestras calculado al importar. Sin SVG en la base de datos. | Menos almacenamiento, y la miniatura respeta automáticamente la privacidad porque deriva de `geom_public`. |