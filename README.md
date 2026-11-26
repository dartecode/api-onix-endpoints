# OnixApi — colección Bruno

Abrir esta carpeta en Bruno como colección YAML/OpenCollection 1.0.0.
Backend: `../APIOnix`, NestJS y Prisma. El contrato HTTP vigente está en
Swagger: http://localhost:3000/api/v1/docs; JSON: http://localhost:3000/api/v1/openapi.json.
Reglas funcionales: [`APIOnix/docs/funcionalidad/`](../APIOnix/docs/funcionalidad/).

## Preparación

1. Preparar API y PostgreSQL locales en Docker según la
   [guía del backend](../APIOnix/docs/operacion/GUIA-DESARROLLADORES.md).
   `.env.docker` configura desarrollo local; el `.env` del backend conserva AWS/QA.
   Aplicar migraciones y semilla mediante el servicio `tools`, no mediante scripts de la base histórica.
2. Levantar el API y comprobar `/api/v1/health/ready`. Sin SMTP en desarrollo, las temporales y
   códigos de correo aparecen en los logs del contenedor `api`; con SMTP se envían al correo.
3. Seleccionar ONIX LOCAL. Conservar URL como origen, sin `/api/v1` ni barra final.
4. Configurar las variables locales de la tabla. No guardar secretos en commits.
5. Primer superadministrador: [PRIMER-ADMIN.md](../APIOnix/docs/operacion/PRIMER-ADMIN.md).
   La semilla crea la cuenta solo si no hay otro activo; no restablece una contraseña existente.

Comandos de actualización local, ejecutados desde `APIOnix/`:

```bash
docker compose --env-file .env.docker -f compose.yaml -f compose.local.yaml build api tools
docker compose --env-file .env.docker -f compose.yaml -f compose.local.yaml run --rm tools pnpm db:deploy
docker compose --env-file .env.docker -f compose.yaml -f compose.local.yaml run --rm tools pnpm db:seed
docker compose --env-file .env.docker -f compose.yaml -f compose.local.yaml up -d --wait api
```

El API usa código compilado, sin recarga automática. Después de cambiarlo, reconstruir las imágenes.

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
Reemplazar los IDs de ejemplo en las URLs: UUID para usuarios; números para clientes,
instalaciones, puestos y correos. El ID interno de un cliente no es su `idPerseo`.
Las peticiones nuevas usan `1` como ejemplo; copiar el `data.id` real antes de ejecutarlas.
Los ejemplos de Catálogos usan provincia `17`, cantón `1701` y parroquia `170151` solo como
referencia; obtener los IDs vigentes desde las consultas geográficas de cada ambiente.
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

- Autenticacion/MFA: login, me, edición propia, renovación, cierre, cambios de contraseña,
  configuración correo/TOTP, verificación, confirmación, estado y desactivación MFA.
- Usuarios: alta, listado general o filtrado por cliente, consulta, edición,
  estado, desbloqueo, restablecimiento y consulta/reinicio MFA.
- Clientes: listado y filtros, ficha, resumen, historial, sincronización manual desde Perseo,
  comprobación de codificación de MAURER y gestión de correos locales.
- Catálogos: provincias, cantones y parroquias sincronizados desde Perseo y sincronización manual.
- Perfiles: catálogo, permisos y reemplazo de matriz.
- Instalaciones: las 11 rutas de instalaciones y puestos, con selección geográfica por IDs,
  horario, estado, ubicación desde mapa/GPS, verificación e historial.
- Salud: live y ready.

Los permisos se leen de la base en cada petición. Los servicios y políticas del API aplican
las reglas y el alcance; Prisma realiza el acceso a datos. Los perfiles de cliente tienen
alcance limitado a su cliente, siempre que su matriz les conceda el permiso de la operación.

## Clientes y Perseo

Perseo administra los datos comerciales y el estado `activo`/`inactivo`. Se retiraron de la
colección `crear-cliente` y `editar-cliente`: el API ya no ofrece esas rutas.

1. Configurar `PERSEO_SERVER` y `PERSEO_API_KEY` en el backend local (`.env.docker`) y recrear
   el servicio `api`. La clave del proveedor no se envía desde Bruno.
2. Con una sesión interna con `cliente.sincronizar`, ejecutar `sincronizar-clientes` sin cuerpo.
   HTTP 200 puede contener incidencias en `data.errores`: revisar también ese arreglo y los contadores.
3. Consultar `listar-clientes`: admite `busqueda`, `estado`, `sector` y `enPerseo` además de paginación.
   Los inactivos siguen visibles y los ausentes del proveedor quedan `enPerseo=false`, sin borrarse.
4. Ejecutar `buscar-cliente-maurer`: se espera `MAURER BOLAÑOS JURGEN SAMMY` cuando el origen
   entregue el nombre correcto. No asumir un ID entre ambientes: en el servidor verificado el
   26/11/2026 tiene `idPerseo=72`, no 62, y ya llega deteriorado desde Perseo.
   El backend respeta el charset y recupera bytes UTF-8 inválidos
   como Windows-1252; no recodifica texto UTF-8 válido solo por contener `Ã`.
5. Repetir la sincronización: sin cambios comerciales cuenta en `sinCambios`; una corrección de
   nombre cuenta en `actualizados` y conserva el ID interno y las relaciones.

Los consumidores finales (tipo `F` o identificación genérica de 10/13 nueves) se excluyen antes
de importar. `total` incluye todos los recibidos y `excluidos` cuenta estas omisiones, sin errores.
Los registros 1 y 71 del servidor verificado corresponden a esos consumidores finales;
la regla no depende de sus IDs. Los demás datos inválidos se corrigen en Perseo.
Sin configuración: `503 PERSEO_NOT_CONFIGURED`; conflicto de sincronización: `409`.
No hay clientes de prueba en la semilla; para crear una instalación se necesita un cliente activo.

## Catálogo geográfico

Perseo también es la fuente de provincias, cantones y parroquias. La semilla no crea una geografía
fija: ejecutar la sincronización manual antes de usar los selectores.

1. Configurar los mismos `PERSEO_SERVER`, `PERSEO_API_KEY` y `PERSEO_TIMEOUT_MS` del backend
   descritos para clientes. La clave no se envía desde Bruno.
2. Con una sesión interna que tenga `catalogo.sincronizar`, ejecutar
   `Catálogos/sincronizar-catalogos` sin cuerpo. Revisar los contadores por nivel en `data`:
   `recibidos`, `creados`, `actualizados`, `sinCambios` y `ausentesEnPerseo`. Los registros
   ausentes se conservan. Si otra sincronización está en curso, responderá `409`.
3. Recorrer `listar-provincias` → `listar-cantones` → `listar-parroquias`, copiando el `id` de
   cada respuesta a la URL siguiente. Las tres devuelven `{ id, nombre }` sin paginar, ordenadas
   por nombre; no requieren permiso propio, pero sí sesión completa y perfil interno.
4. Usar esos IDs en instalaciones como `idCanton` e `idParroquia`. La provincia se deriva del
   cantón; no se envía como texto. Al cambiar de cantón, enviar una parroquia compatible o
   `idParroquia: null`.

No asumir las cifras clásicas de 24 provincias y 221 cantones. En la verificación local de
Perseo llegaron 25 provincias, 228 cantones y 1.436 parroquias; Quito (`1701`) tuvo 70
parroquias y una de sus parroquias es `170151` ALANGASÍ. Incluir zonas no delimitadas si el
proveedor las entrega. Las lecturas `404` son `PROVINCE_NOT_FOUND` o `CANTON_NOT_FOUND`;
la sincronización puede responder `503` si falla la configuración, el proveedor o la
validación de jerarquía. Un perfil de cliente recibe `403 FORBIDDEN`.

Contrato y casos en [CATALOGO-GEOGRAFICO.md](../APIOnix/docs/funcionalidad/CATALOGO-GEOGRAFICO.md)
y [INSTALACIONES.md](../APIOnix/docs/funcionalidad/INSTALACIONES.md).

## Instalaciones y puestos

Abrir la carpeta **Instalaciones y puestos** y usar una sesión completa con los permisos requeridos.
La petición `crear-instalacion` lleva un ejemplo nocturno; `editar-instalacion`, uno de 24 horas.
Los ejemplos incluyen `idCanton` e `idParroquia` tomados del catálogo; reemplazarlos por los IDs
vigentes del ambiente.

### Geografía

1. Obtener provincia, cantón y parroquia desde **Catálogos**. Enviar solo `idCanton` e
   `idParroquia`; la provincia se calcula y devuelve como objeto `{ id, nombre }`.
2. Crear una instalación con un cantón existente y, si se envía, una parroquia de ese cantón.
   Un cantón inexistente o una parroquia ajena responde `400 VALIDATION_ERROR`.
3. En `editar-instalacion`, omitir un ID conserva el valor y `null` lo elimina. Al cambiar el
   cantón, enviar una parroquia compatible o `idParroquia: null`.
4. Comprobar que ficha y listado devuelven `canton`, `provincia` y `parroquia` como objetos o `null`.
   Los filtros `idProvincia`, `idCanton` e `idParroquia` son combinables; `busqueda` incluye el
   nombre del cantón. Las fotos nuevas del historial incluyen los IDs y nombres geográficos.

### Horarios

Probar cada caso en creación (con nombre único y un cliente activo) y edición:

| Apertura / cierre | POST | PATCH |
| --- | --- | --- |
| `18:00` / `08:00` | `201` | `200` |
| `00:00` / `00:00` | `201` | `200` |
| `09:00` / `09:00` | `400 INVALID_SCHEDULE` | `400 INVALID_SCHEDULE` |
| Ambas `null` | `201` | `200` |
| Una hora informada y la otra `null` | `400 INVALID_SCHEDULE` | `400 INVALID_SCHEDULE` |

El cierre menor que la apertura termina al día siguiente. En PATCH se valida el estado combinado:
omitir una hora conserva la guardada; para quitar el horario existente enviar ambas como `null`.
Un formato como `25:00` responde `400 VALIDATION_ERROR`, no `INVALID_SCHEDULE`.

### Permisos y matrices existentes

`instalacion.ver` se asigna inicialmente a `superadmin`, `administrador`, `supervisor`, `tecnico`
y `auditorInterno`. No se asigna a `operador`, `gerencia`, `dpd`, `clienteAdministrador`,
`clienteResponsableInstalacion` ni `clienteConsulta`. La autorización depende de la matriz vigente.

Para una base ya creada, la semilla no retira el permiso concedido anteriormente:

1. Ejecutar `Perfiles/listar` con un actor con `perfil.ver` y `perfil.editar`.
2. En cada uno de los seis perfiles afectados, copiar la lista actual, retirar solo
   `instalacion.ver` y enviarla en `Perfiles/reemplazar-permisos`, cambiando el código en la URL.
   El PUT reemplaza **toda** la matriz: conservar los demás permisos.
3. Los cambios reales se auditan y revocan las sesiones del perfil. Volver a iniciar sesión.
4. Con Operador: listado, ficha e historial → `403 FORBIDDEN`. Con Auditor interno con permiso:
   listado → `200`. Volver a la cuenta autorizada para continuar los cambios.

### Puestos, historial y ubicación

1. Crear un puesto, editar su nombre y cambiar su estado; usar el ID del puesto en `/puestos/:id`.
2. Consultar `historial-instalacion` usando el ID de la instalación padre. Las entradas nuevas
   `PUESTO_CREADO`, `PUESTO_ACTUALIZADO` y `PUESTO_ESTADO` deben tener `motivo: null` y
   `nuevo.puesto_nombre`; las actualizaciones conservan el nombre anterior en `anterior`.
3. Cambiar el estado de la instalación con motivo: su entrada `INSTALACION_ESTADO` sí lo conserva.
   El estado de un puesto solo recibe `estado`, no `motivo`. Los históricos no se reescriben.
4. Registrar coordenadas con `fuente: "mapa"`: precisión `aproximada`. Con `fuente: "gps"` y
   `instalacion.verificarUbicacion`: `verificada`. Registrar requiere `instalacion.ubicar`.
5. Sin coordenadas, verificar devuelve `422 LOCATION_MISSING`. Con coordenadas y permiso,
   `verificar-ubicacion` devuelve `200` y registra actor/fecha de verificación.

Reglas y contrato: [INSTALACIONES.md](../APIOnix/docs/funcionalidad/INSTALACIONES.md) y OpenAPI.
Esta guía describe la aceptación manual; no acredita su ejecución contra un ambiente concreto.
Resultados del entorno Docker local del 26/11/2026, incluidas exclusiones y el problema de nombre
en el origen: [evidencia del backend](../APIOnix/docs/integracion/VERIFICACION-LOCAL-20261126.md).

## Respuestas y comprobación manual

Éxito: `{ success: true, statusCode, data, meta?, path, timestamp }`.
Error: `{ success: false, statusCode, code, message, details?, path, timestamp }`.
Leer datos dentro de data. 400 validación; 401 credenciales/sesión; 403 permisos o
sesión restringida; 404 inexistente/fuera de alcance; 409 duplicado/conflicto;
422 regla de negocio; 503 base/correo/proveedor no disponible.

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
Detalle: [AUTENTICACION.md](../APIOnix/docs/funcionalidad/AUTENTICACION.md) y
[USUARIOS.md](../APIOnix/docs/funcionalidad/USUARIOS.md).
