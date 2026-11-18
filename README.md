# OnixApi — colección Bruno

Abrir esta carpeta en Bruno como colección YAML/OpenCollection 1.0.0.
Backend: `../APIOnix`; contrato PostgreSQL: `../BDOnix/docs/CONTRATO_API.md`.
Swagger: http://localhost:3000/api/v1/docs; JSON: /api/v1/openapi.json.

## Preparación

1. Instalar/actualizar la base según `BDOnix/README.md`, usando `db/instalar.sql`.
   El script `scripts/20261117_login_bloqueo.sql` corresponde al contrato anterior:
   no aplicarlo sobre el esquema actual de `db/`.
2. Arrancar APIOnix con PostgreSQL real. Sin SMTP en desarrollo, las temporales y
   códigos de correo aparecen en la consola del API; con SMTP se envían al correo.
3. Seleccionar ONIX LOCAL. Conservar URL como origen, sin `/api/v1` ni barra final.
4. Configurar las variables locales de la tabla. No guardar secretos en commits.
5. Primer administrador: `APIOnix/docs/PRIMER-ADMIN.md` (`pnpm admin:hash`).

## Variables locales

| Variable         | Uso                                                                               |
| ---------------- | --------------------------------------------------------------------------------- |
| URL              | Origen del API, por ejemplo http://localhost:3000                                 |
| TOKEN            | JWT del usuario; capturado al iniciar sesión o verificar MFA                      |
| ID_DESAFIO       | UUID de desafío; capturado en login, o copiar de data.idDesafio al configurar MFA |
| TOKEN_RENOVACION | Último token de renovación móvil; en web usar cookie jar                          |

El script de login conserva las credenciales existentes del request, captura
data.tokenAcceso y limpia TOKEN cuando no hay tokens. Leer el siguiente paso en data.paso.
Escribir contraseñas y códigos directamente en el cuerpo de cada petición.
Reemplazar los IDs de ejemplo en las URLs: UUID para usuarios y número para clientes.
Las peticiones administrativas usan TOKEN: iniciar sesión con el actor autorizado.
Si se alternan cuentas, copiar el token del actor directamente en el Bearer de la
petición que deba conservarlo, antes de iniciar sesión con la otra cuenta.
No hay tests ni aserciones automáticas, solo scripts para capturar credenciales.

## Flujos de contraseña

| Petición                   | POST después de /api/v1                   | Acceso                              |
| -------------------------- | ----------------------------------------- | ----------------------------------- |
| cambiar-contrasena-inicial | /autenticacion/cambiar-contrasena-inicial | Sesión cambio_contrasena            |
| cambiar-contrasena         | /autenticacion/cambiar-contrasena         | Sesión completa + contraseña actual |
| restablecer-contrasena     | /usuarios/:id/restablecer-contrasena      | usuario.editar y alcance            |

Inicial y propio reciben nuevaContrasena y confirmacionContrasena; el propio agrega
contrasenaActual. No se recortan espacios; estos no cuentan como símbolo. Se exige
minúscula, mayúscula y número o símbolo. La nueva debe ser distinta de la anterior.
El cambio inicial continúa al siguiente paso y emite las credenciales necesarias sin
pedir usuario y contraseña otra vez. El cambio propio sí revoca la sesión y limpia la
cookie: iniciar sesión nuevamente.
El restablecimiento no recibe contraseña: genera una temporal, la envía al correo,
revoca sesiones y exige cambio inicial. Conserva bloqueo e intentos; desbloquear es
una acción separada mediante PATCH /usuarios/:id/estado con accion=desbloquear.
Cambiar/restablecer conserva la configuración MFA e invalida desafíos pendientes.

## Login y MFA

Enviar usuario y contrasena con X-Onix-Client web o mobile y leer data.paso:

- cambio_contrasena: ejecutar cambio inicial; su respuesta continúa a configurar_mfa,
  mfa o sesion y actualiza TOKEN/ID_DESAFIO.
- configurar_mfa: obtener configuración por correo/TOTP, copiar data.idDesafio al
  cuerpo de mfa-confirmar junto con el código recibido; la confirmación devuelve la
  sesión completa y actualiza TOKEN.
- mfa: el login captura ID_DESAFIO; ejecutar mfa-verificar y obtener tokens.
- sesion: TOKEN completo, usar me y operaciones autorizadas.

En web la renovación llega en cookie HttpOnly onix_refresh, administrada por Bruno.
En móvil llega data.tokenRenovacion: copiar el último valor a TOKEN_RENOVACION.
Renovar rota la credencial, por lo que debe usarse siempre la más reciente.
Una sola sesión activa por cuenta: un login que emite sesión sustituye la anterior.

## Cobertura

- Autenticacion: login, me, renovación, cierre, dos cambios de contraseña y seis
  operaciones de MFA.
- Usuarios: alta, listado, consulta, edición, estado, desbloqueo, restablecimiento
  y consulta/reinicio MFA.
- Clientes: alta, listado, consulta y edición.
- Salud: live y ready.

Los permisos administrativos se comprueban en cada petición y dentro de los SP.
Los perfiles de cliente tienen alcance limitado a su cliente.

## Respuestas y comprobación manual

Éxito: `{ success: true, statusCode, data, meta?, path, timestamp }`.
Error: `{ success: false, statusCode, code, message, details?, path, timestamp }`.
Leer datos dentro de data. 400 validación; 401 credenciales/sesión; 403 permisos o
sesión restringida; 404 inexistente/fuera de alcance; 409 duplicado/conflicto;
422 regla del SP; 501 SP ausente; 503 base/correo no disponible.

1. Completar login inicial y MFA; comprobar me con sesión completa.
2. Cambiar clave desde el perfil: clave actual incorrecta → 401; confirmación
   distinta o reutilización → 400; cambio correcto → 200.
3. Comprobar que JWT y renovación anteriores ya no sirven. Login con clave nueva.
4. Desde otra cuenta autorizada, restablecer al objetivo; comprobar correo,
   revocación, login con temporal y cambio inicial obligatorio.
5. Comprobar sin permiso → 403, fuera de alcance → 404 y bloqueo conservado.
6. Revisar auditoría sin contraseñas ni hashes.

Si correo devuelve MAIL_UNAVAILABLE, el cambio puede estar guardado: repetir el
restablecimiento genera otra temporal. No repetir el alta ciegamente.
Detalle: `APIOnix/docs/CONTRASENAS.md`. Esta colección sigue el backend actual;
no documenta rate limiting ni desbloqueo con justificación del contrato anterior.
