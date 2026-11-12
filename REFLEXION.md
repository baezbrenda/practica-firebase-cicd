# Reflexión · CI/CD con Firebase Hosting

**Repo:** https://github.com/baezbrenda/practica-firebase-cicd
**Producción:** https://practica-cicd-brenda.web.app
**PR de pipeline (con comentario del bot y preview):** https://github.com/baezbrenda/practica-firebase-cicd/pull/1
**PR de headers:** https://github.com/baezbrenda/practica-firebase-cicd/pull/2

## Medición ANTES (B1, producción sin tocar nada)

```
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
```

## Medición DESPUÉS (B5, producción tras mergear el PR #2)

```
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
```

Verificación en producción (ventana privada): el filtro de `/cursos` responde (Inicial → "2 de 6 cursos"). El footer muestra el SHA y la rama `main`.

## Pipeline y rollback (Parte A)

- Preview: workflow en `pull_request`, con `expires: 3d` y `concurrency` que cancela builds anteriores del mismo PR.
- Producción: workflow en `push` a `main`, `channelId: live`, `concurrency` sin cancelación y `environment: production`.
- Trazabilidad: en PR se usa `github.event.pull_request.head.sha` (y no `github.sha`, que apunta a un commit de merge temporal) y `github.head_ref`.
- Rollback ejecutado desde la consola (Hosting → Release history): el sitio volvió a `Versión local` y luego a la versión del merge. Fue instantáneo y sin recompilar.

## Preguntas

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