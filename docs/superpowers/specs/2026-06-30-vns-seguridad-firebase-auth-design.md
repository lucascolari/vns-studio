# VNS Studio — Seguridad del panel con Firebase Auth

**Fecha:** 2026-06-30
**Proyecto:** vns-studio (repo público, GitHub Pages)
**Estado:** Diseño aprobado, pendiente de implementación

## Problema

El panel de administración (`vns_admin.html`) protege el acceso con credenciales
escritas directamente en el código:

```js
var CREDS = { 'Lauti': 'vnsstudio', 'Santi': 'vnsstudio' };
```

Como el repositorio es **público**, cualquiera puede leer esas contraseñas en
GitHub o haciendo "ver código fuente" en el sitio publicado. El "login" es
decorativo: no aporta seguridad real.

Más grave: la base de datos (Firebase Realtime Database) se usa solo con la
`databaseURL` y sin reglas de escritura verificadas. **Si las reglas están
abiertas (`".write": true`), cualquier persona en internet puede leer, modificar
o borrar toda la agenda de turnos directamente, sin siquiera abrir el panel.**
Existen bots que escanean internet buscando bases de Firebase abiertas.

> **Aclaración de alcance del riesgo:** no hay datos personales sensibles
> guardados. La base solo contiene qué horarios están ocupados (los clientes
> reservan por WhatsApp; sus datos no se guardan en Firebase). El riesgo real es
> de **integridad/disponibilidad**: que alguien borre o ensucie la agenda, no
> una fuga de datos.

## Solución

Mover la autenticación **fuera del código** usando **Firebase Authentication**, y
cerrar la base con **reglas de seguridad** que solo permitan escribir a usuarios
logueados.

### Componentes

1. **Firebase Authentication (Email/Password)**
   - Se activa en la consola de Firebase.
   - Se crea un usuario por barbero (ej. `lauti@vnsstudio...`, `santi@vnsstudio...`)
     con su contraseña. Las credenciales viven en Firebase, **nunca en el código**.

2. **Mapa de barberos (`/barberos/{uid}`)**
   - Un nodo que asocia cada cuenta de Firebase Auth con su nombre de barbero:
     `barberos/{uid-de-lauti} = "Lauti"`, `barberos/{uid-de-santi} = "Santi"`.
   - Se carga **a mano una sola vez** en la consola (se necesita el UID de cada
     usuario, que aparece en Authentication). No es secreto.
   - Es la **única fuente** de la identidad del barbero: la usan tanto el panel
     (para saber bajo qué nombre guardar) como las reglas (para autorizar).

3. **Reglas de la Realtime Database**
   ```json
   {
     "rules": {
       "barberos": {
         ".read": "auth != null",
         ".write": false
       },
       "ocupados": {
         ".read": true,
         "$clave": {
           ".write": "auth != null && $clave.beginsWith(root.child('barberos').child(auth.uid).val() + '_')"
         }
       }
     }
   }
   ```
   - `ocupados` lectura `true` → la web pública (`index.html`) sigue leyendo los
     turnos sin login, que es lo que necesita para mostrar disponibilidad.
   - La escritura solo se permite si el usuario está logueado **y** la clave que
     escribe empieza con su propio nombre + `_` (ej. el dueño de la cuenta
     mapeada a `"Lauti"` solo puede escribir claves `Lauti_...`). Así **cada
     barbero edita únicamente lo suyo**; tocar la agenda de otro es rechazado.
   - `barberos` es de solo lectura para logueados y nadie lo puede escribir desde
     la app (se administra a mano en la consola).

   > **Nota sobre `beginsWith`:** funciona porque los nombres se separan con `_`
   > y ningún nombre es prefijo de otro (`Lauti_` vs `Santi_`). Si en el futuro
   > se agregan nombres donde uno sea prefijo de otro, conviene pasar a una
   > estructura anidada `ocupados/{barbero}/{fecha}`.

4. **Panel (`vns_admin.html`)**
   - Se agrega el SDK de Firebase Auth (`firebase-auth.js`).
   - La `firebaseConfig` se completa con los datos públicos del proyecto
     (`apiKey`, `authDomain`, `projectId`, etc.) que requiere Auth. Estos valores
     son públicos por diseño y no representan un riesgo.
   - El login deja de usar `CREDS`. En su lugar:
     - El formulario pide **email + contraseña**.
     - `signInWithEmailAndPassword(email, pass)` valida contra Firebase.
     - Tras loguear, el panel **lee `/barberos/{uid}`** para saber qué barbero es
       (`userActivo`) y bajo qué nombre guardar los turnos
       (`ocupados/{barbero}_{fecha}`). Ya no hay mapa de nombres en el código: la
       identidad sale de Firebase, la misma fuente que usan las reglas.
   - `logout()` usa `firebase.auth().signOut()`.
   - Las escrituras (`db.ref('ocupados/...').set(...)`) no cambian: una vez
     logueado, el SDK adjunta la identidad automáticamente y las reglas la aceptan.

### Flujo de datos

- **Web pública:** lee `ocupados.json` por REST (sin login) → muestra
  disponibilidad. Sin cambios.
- **Cliente reserva:** por WhatsApp (sin cambios).
- **Barbero:** abre el panel → `signInWithEmailAndPassword` → marca horarios
  ocupados/libres → `set()` con auth válido → reglas permiten la escritura.

### Manejo de errores

- **Credenciales incorrectas:** se muestra el mensaje de error existente
  (`mostrarError`) usando los códigos de error de Firebase Auth
  (`auth/wrong-password`, `auth/user-not-found`, etc.).
- **Sin conexión:** mantener el indicador de estado de Firebase ya presente
  (`fbStatus`).

### Verificación

1. La web pública sigue mostrando la disponibilidad correctamente (lectura OK).
2. Con login válido, un barbero marca un horario **propio** y se guarda (OK).
3. Un barbero **no puede** escribir la clave de otro (probar y confirmar que
   Firebase lo rechaza).
4. Sin login, un intento de escritura directa contra la base es **rechazado**
   (probar con un comando REST y confirmar error de permisos).
5. Las contraseñas viejas (`vnsstudio`) ya no sirven para nada.

## Identidad y aislamiento de los barberos

Los turnos se guardan por nombre de barbero (`{barbero}_{fecha}`). Cada cuenta de
Firebase Auth está mapeada a su nombre en `/barberos/{uid}`. Las reglas usan ese
mapa para que **cada barbero pueda escribir únicamente sus propias claves**
(`Lauti_...` solo el dueño de la cuenta de Lauti). Un intento de editar la agenda
de otro barbero es rechazado por Firebase. La lectura de la agenda es pública (la
web la necesita), pero eso no permite modificarla.

Barberos iniciales: **Lauti** y **Santi** (dos cuentas). Agregar uno nuevo en el
futuro = crear su usuario en Authentication + agregar su entrada en `/barberos`.

## Fuera de alcance

- **Neaboats:** se resuelve por separado, en su propio ciclo, con el enfoque de
  **Google Apps Script** (el script valida usuario+contraseña del lado del
  servidor y guarda los secretos, sin incorporar Firebase). No se toca en este
  spec.
- **Limpieza de historial de Git:** las contraseñas viejas quedan en el historial
  pero son **inservibles** apenas se activa Firebase Auth, así que no se reescribe
  el historial.

## Pasos que dependen de la consola de Firebase (los hace Lucas)

1. Revisar las reglas actuales (diagnóstico inicial).
2. Activar Email/Password en Authentication.
3. Crear un usuario por barbero (Lauti y Santi).
4. Copiar el **UID** de cada usuario y cargar el mapa `/barberos/{uid} = "Lauti"`
   y `/barberos/{uid} = "Santi"` en la Realtime Database.
5. Obtener la `firebaseConfig` completa (Configuración del proyecto).
6. Publicar las reglas de la base.

## Pasos de código (los hace Claude)

1. Agregar SDK de Firebase Auth y completar `firebaseConfig`.
2. Reescribir el formulario de login (email + contraseña).
3. Reescribir `doLogin()` con `signInWithEmailAndPassword` + mapa email→nombre.
4. Reescribir `logout()` con `signOut()`.
5. Ajustar mensajes de error a los códigos de Firebase Auth.
