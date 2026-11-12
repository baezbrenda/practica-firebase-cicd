# Reflexión · CI/CD con Firebase Hosting y SQL Connect

**Repo:** https://github.com/baezbrenda/practica-firebase-cicd  
**Producción:** https://practica-cicd-brenda.web.app  
**PR de pipeline (con comentario del bot y preview):** https://github.com/baezbrenda/practica-firebase-cicd/pull/1  
**PR de headers (Clase 1):** https://github.com/baezbrenda/practica-firebase-cicd/pull/2  

---

# Parte 1 · CI/CD con Firebase Hosting y Headers de Seguridad

## Medición ANTES (B1, producción sin tocar nada)

▶ HTML: https://practica-cicd-brenda.web.app/
status 200
✘ content-security-policy (falta)
✔ strict-transport-security: max-age=31556926; includeSubDomains; preload
✘ x-content-type-options (falta)
✘ x-frame-options (falta)
✘ referrer-policy (falta)
✘ permissions-policy (falta)
· cache-control: max-age=3600

▶ Asset estático: /_next/static/chunks/1x6obpmlu1_c2.js
✘ cache-control: max-age=3600

❌ 6 control(es) pendiente(s).

## Medición DESPUÉS (B5, producción tras mergear el PR #2)

▶ HTML: https://practica-cicd-brenda.web.app/
status 200
✔ content-security-policy: default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self'; img-src 'self' https://picsum.photos …
✔ strict-transport-security: max-age=31556926; includeSubDomains; preload
✔ x-content-type-options: nosniff
✔ x-frame-options: DENY
✔ referrer-policy: strict-origin-when-cross-origin
✔ permissions-policy: camera=(), microphone=(), geolocation=()
· cache-control: no-cache

▶ Asset estático: /_next/static/chunks/1x6obpmlu1_c2.js
✔ cache-control: public, max-age=31536000, immutable

✅ Todos los controles pasaron.

Verificación en producción (ventana privada): el filtro de `/cursos` responde (Inicial → "2 de 6 cursos"). El footer muestra el SHA y la rama `main`.

## Pipeline y rollback (Parte A)

- Preview: workflow en `pull_request`, con `expires: 3d` y `concurrency` que cancela builds anteriores del mismo PR.
- Producción: workflow en `push` a `main`, `channelId: live`, `concurrency` sin cancelación y `environment: production`.
- Trazabilidad: en PR se usa `github.event.pull_request.head.sha` (y no `github.sha`, que apunta a un commit de merge temporal) y `github.head_ref`.
- Rollback ejecutado desde la consola (Hosting → Release history): el sitio volvió a `Versión local` y luego a la versión del merge. Fue instantáneo y sin recompilar.

## Preguntas · Clase 1

**1. ¿Por qué el preview tiene vencimiento?**
Cada PR crea una copia pública del sitio accesible por URL. Si no venciera, se acumularían versiones viejas, posiblemente con código sin revisar o vulnerable, que habría que mantener y que cualquiera con el link podría ver. Con `expires: 3d` se borran solas.

**2. Desplegar vs. liberar.**
Desplegar es subir una versión a Firebase; liberar es hacerla visible a los usuarios en el canal `live`. El preview despliega sin liberar; el merge a `main` despliega y libera; el rollback solo libera una versión que ya estaba subida, por eso fue instantáneo.

**3. ¿Por qué la clave de la service account va en un secreto?**
El repo es público y la clave da permiso de desplegar: quien la obtenga podría publicar contenido arbitrario en el sitio. Se guarda como secreto de GitHub (`FIREBASE_SERVICE_ACCOUNT_PRACTICA_CICD_BRENDA`). Para limitar el daño: permisos mínimos (solo Hosting), rotar la clave si se filtra, usar un environment con aprobación manual o autenticación OIDC en lugar de claves de larga duración.

**4. ¿Por qué la CSP rompió la app y qué opciones evalué?**
- `script-src 'self'` bloqueó los 3 scripts inline que Next.js inyecta para hidratar la app: el filtro de `/cursos` dejó de responder (la consola mostró "Executing inline script violates...").
- `img-src 'self'` bloqueó la imagen de portada, que viene de `picsum.photos` y redirige (302) a `fastly.picsum.photos`; hubo que permitir los dos orígenes exactos, sin comodines.
- Opciones para los scripts: (a) `'unsafe-inline'`: simple, pero permite cualquier script inline y debilita la defensa contra XSS; es la que usé. (b) nonce: no sirve, no hay servidor que genere uno por request. (c) hashes `sha256`: la más segura, pero hay que calcularlos después de `next build` y antes del deploy, porque cambian con el contenido.
- Costo de la imagen: se confía en dos terceros. Durante las pruebas picsum respondió 522 (timeout de Cloudflare) y la imagen no cargó en el preview; en local sí se vio con la misma política. Es un ejemplo del riesgo de depender de un origen externo.

**5. ¿Qué ataque mitiga cada header?**
- `Strict-Transport-Security`: fuerza HTTPS en visitas futuras; evita downgrade y SSL stripping. Nota: en producción Firebase devolvió `max-age=31556926` en lugar del `63072000` que declaré; el emulador sí respetó el mío.
- `X-Content-Type-Options: nosniff`: evita que el navegador "adivine" el tipo de un archivo (MIME sniffing) y ejecute como script algo que no lo es.
- `X-Frame-Options: DENY`: nadie puede embeber el sitio en un iframe; evita clickjacking.
- `Referrer-Policy: strict-origin-when-cross-origin`: al navegar a otros sitios solo se envía el origen, no la URL completa (evita filtrar rutas o parámetros).
- `Permissions-Policy`: deshabilita cámara, micrófono y ubicación para la página, reduciendo el daño si se inyecta código malicioso.

**6. ¿Por qué `/_next/static/` se cachea un año y el HTML no?**
Los assets llevan un hash del contenido en el nombre: si cambian, cambia la URL, así que pueden ser `immutable`. El HTML siempre tiene la misma URL y es el que apunta a los assets nuevos; si se cacheara, un deploy o un rollback tardaría en verse. Por eso `no-cache` (el navegador revalida siempre). Antes ambos tenían el valor por defecto `max-age=3600`.

**7. Si la app necesitara SSR.**
Firebase Hosting en Spark solo sirve archivos estáticos. Habría que usar App Hosting sobre Cloud Run, que exige plan Blaze (cuenta de facturación) y tiene costo variable. En la CSP se podría usar un nonce por request, porque ya habría servidor, evitando `'unsafe-inline'`. La arquitectura pasaría de CDN estático a un servicio con servidor y escalado.

**Pregunta de A6: ¿qué pasa con el próximo merge tras un rollback?**
`main` sigue teniendo el código nuevo, así que el siguiente push a `main` volvería a desplegarlo. En un equipo real se hace `git revert` del commit problemático (para que `main` coincida con producción) o se corrige en un nuevo PR. El rollback es el parche rápido; el revert es la solución definitiva.

---

## SQL Connect · Modelo relacional y autorización (Clase 2)

### Auditoría de seguridad (`npm run atacar`)

**ANTES de corregir `ListAlumnos`:**
La operación heredada `ListAlumnos` estaba expuesta con `@auth(level: PUBLIC)`, lo que permitía que cualquier cliente no autenticado leyera la lista completa de alumnos registrados (ID, nombre y email).

**DESPUÉS de corregir `ListAlumnos` a `@auth(level: NO_ACCESS)` y configurar `@auth(level: USER)`:**

Auditoría de SQL Connect · proyecto practica-cicd-brenda · emuladores locales

✔ ListCursos responde sin login (7 cursos) — público por diseño
✔ ListAlumnos rechazada para un cliente
unauthorized: you are not authorized to perform this operation
✔ Inscribirme rechazada sin login
unauthenticated: this operation requires a signed-in user
✔ Inscribirme es idempotente (repetir no falla)
✔ MisInscripciones solo devuelve lo del usuario que llama
✔ B no puede dar de baja a A

✅ Todos los controles pasaron

### Preguntas · Clase 2

**1. ¿Por qué ninguna operación de este conector recibe el id del alumno como variable? ¿Qué es IDOR y cómo lo evita `auth.uid`?**
Todo parámetro enviado desde el cliente puede ser interceptado y modificado por un usuario malicioso. **IDOR (Insecure Direct Object Reference)** ocurre cuando una API confía en un identificador enviado por el cliente (como `$alumnoId`) para acceder o modificar un recurso sin verificar la propiedad real, permitiendo operar en nombre de otro usuario. Al utilizar expresiones del servidor como `auth.uid` y `auth.token.email`, SQL Connect extrae la identidad directamente del token JWT validado por Firebase Auth en el servidor, eliminando la posibilidad de suplantación[cite: 3, 6].

**2. `ListCursos` es `PUBLIC` y `ListAlumnos` terminó en `NO_ACCESS`. ¿Qué criterio usaste? ¿Cómo haría el "panel de admin" para listar alumnos?**
Se aplicó el **principio de mínimo privilegio**: el catálogo de cursos debe ser accesible para cualquier visitante antes de registrarse (`PUBLIC`), mientras que la nómina de alumnos expone datos personales (PII) que ningún cliente web necesita consultar (`NO_ACCESS`)[cite: 7, 8]. Para un panel de administración, la operación se mantiene en `NO_ACCESS` para los clientes públicos y se ejecuta desde un entorno backend de confianza (como Cloud Functions o un servidor administrativo) utilizando el **Firebase Admin SDK** con credenciales de cuenta de servicio, o verificando un *Custom Claim* de administrador en el token (`auth.token.admin == true`).

**3. ¿Qué ventaja tiene una clave primaria compuesta en `Matricula` frente a validar duplicados en el código del cliente?**
Definir `@table(key: ["alumno", "curso"])` impone una restricción de unicidad (`PRIMARY KEY`) directamente en el motor de PostgreSQL[cite: 6, 10]. Validar duplicados únicamente en el cliente es inseguro porque puede evadirse llamando directamente a la API con scripts (`curl`) y además es vulnerable a condiciones de carrera (*race conditions*) si se envían dos peticiones simultáneas. La base de datos garantiza integridad referencial y atomicidad sin importar cómo se invoque la operación.

**4. La configuración de Firebase (`apiKey`, etc.) es pública. ¿Por qué eso no compromete la base de datos? ¿Qué capa faltaría para frenar a alguien que usa esa config desde un script?**
La `apiKey` de Firebase únicamente identifica al proyecto frente a los servicios de Google; no otorga permisos de lectura ni escritura. La seguridad real recae en la validación de tokens JWT de Firebase Auth y en las reglas `@auth` declaradas en cada operación del servidor[cite: 3]. Para evitar que alguien automatice llamadas contra la API usando esa configuración pública desde scripts externos, falta habilitar **Firebase App Check**, que exige una prueba criptográfica de integridad (como reCAPTCHA Enterprise) demostrando que la petición proviene de la aplicación web legítima[cite: 8].

**5. Comparado con una API REST propia (Express + `pg`), ¿qué te da SQL Connect y qué perdés?**
- **Ventajas (qué te da):** Eliminación de inyección SQL por diseño (el cliente nunca envía SQL, solo el nombre de la operación precompilada), generación automática de tipos y SDKs para el frontend, autorización declarativa junto a cada consulta y cero mantenimiento de infraestructura de servidores[cite: 4].
- **Desventajas (qué perdés):** Menor flexibilidad para ejecutar lógica de negocio compleja antes o después de tocar la base de datos (por ejemplo, validaciones transaccionales de cupos en tiempo real, envío de correos o integración con pasarelas de pago) y mayor acoplamiento al ecosistema de Firebase/Google Cloud.

**6. ¿Por qué no alcanzó con la CSP de la clase 1 y por qué conviene listar dominios exactos en `connect-src`?**
En la clase 1 solo teníamos `default-src 'self'`, lo cual bloquea por defecto las conexiones `fetch` o XHR hacia dominios externos, generando errores `Refused to connect` cuando la app intenta comunicarse con Firebase Auth y SQL Connect en producción[cite: 8]. Conviene especificar los orígenes exactos (`https://identitytoolkit.googleapis.com`, `https://securetoken.googleapis.com` y `https://us-east4-firebasedataconnect.googleapis.com`) en lugar de comodines amplios como `https://*.googleapis.com` para minimizar la superficie de exfiltración: ante una eventual vulnerabilidad XSS, un script inyectado no podría enviar datos robados hacia otros servicios arbitrarios alojados en Google Cloud[cite: 8].