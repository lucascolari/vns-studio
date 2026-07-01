# VNS Studio — Seguridad con Firebase Auth · Plan de implementación

> **Para quien ejecuta:** este plan se hace **en colaboración**. Los pasos
> marcados **[Lucas]** se hacen en la consola de Firebase o en Git; los marcados
> **[Claude]** son cambios de código. No hay tests automáticos (sitio estático):
> la verificación es **manual en el navegador**. Pasos con checkbox (`- [ ]`).

**Goal:** Reemplazar el login falso del panel de VNS por Firebase Authentication
real, con reglas que permitan a cada barbero editar solo su propia agenda.

**Architecture:** El panel (`vns_admin.html`) autentica con Firebase Auth
(email+contraseña). La identidad del barbero sale del nodo `/barberos/{uid}`. Las
reglas de la Realtime Database permiten lectura pública de `ocupados` y escritura
solo de las claves propias de cada barbero logueado. La web pública no se toca.

**Tech Stack:** HTML/JS vanilla, Firebase JS SDK v8.10.1 (app + database + auth),
Firebase Realtime Database, GitHub Pages.

## Global Constraints

- SDK de Firebase: versión **8.10.1** (la que ya usa el proyecto). No mezclar con v9+.
- La web pública (`index.html`) **no se modifica**: sigue leyendo `ocupados.json`
  por REST sin login.
- Barberos: **Lauti** y **Santi** (dos cuentas).
- Los valores de `firebaseConfig` son públicos por diseño; no son secretos.
- **Orden seguro de despliegue:** primero se publica el código nuevo (funciona con
  las reglas viejas), y recién **después** se ajustan las reglas. Así no queda una
  ventana donde el panel esté roto.

---

### Task 0: Diagnóstico — revisar las reglas actuales [Lucas]

**Objetivo:** saber si la base está hoy abierta (define la urgencia). No cambia nada.

- [ ] **Step 1:** Entrar a [console.firebase.google.com](https://console.firebase.google.com) → proyecto **vns-studio** → **Realtime Database** → pestaña **Reglas**.
- [ ] **Step 2:** Anotar/sacar captura de lo que dice hoy. Si ves `".write": true` o `".read": true` a nivel raíz, la base está abierta (confirmá la urgencia). Si ya tiene restricciones, mejor todavía.
- [ ] **Step 3:** Avisarle a Claude qué decían las reglas (lo registramos y seguimos).

---

### Task 1: Activar Authentication y crear los usuarios [Lucas]

**Objetivo:** crear las cuentas reales de los barberos en Firebase.

- [ ] **Step 1:** En la consola → **Authentication** → **Comenzar** (si no estaba activado).
- [ ] **Step 2:** Pestaña **Sign-in method** → habilitar **Correo electrónico/contraseña** → Guardar.
- [ ] **Step 3:** Pestaña **Users** → **Agregar usuario**: crear el de Lauti (ej. `lauti@vnsstudio.com`) con una contraseña que elijas. Repetir para Santi (ej. `santi@vnsstudio.com`).

  > Los emails pueden ser inventados/internos; no hace falta que existan de verdad. Lo importante es que cada barbero recuerde el suyo y su contraseña.

- [ ] **Step 4:** Copiar el **UID** de cada usuario (columna "User UID" en la tabla). Guardalos: los necesitás en la Task 2. Ej:
  - Lauti → `aB3x...`
  - Santi → `kL9p...`

**Verificación:** en la pestaña Users aparecen las dos cuentas con su UID.

---

### Task 2: Cargar el mapa `/barberos` [Lucas]

**Objetivo:** decirle a Firebase qué nombre corresponde a cada cuenta.

- [ ] **Step 1:** Consola → **Realtime Database** → pestaña **Datos**.
- [ ] **Step 2:** Sobre el nodo raíz, crear esta estructura (con el botón **+**), usando los UID de la Task 1:

  ```
  barberos
    ├─ <UID-de-Lauti>: "Lauti"
    └─ <UID-de-Santi>: "Santi"
  ```

  El resultado en JSON debe verse así:
  ```json
  {
    "barberos": {
      "aB3x...": "Lauti",
      "kL9p...": "Santi"
    }
  }
  ```

**Verificación:** en Datos se ve el nodo `barberos` con las dos entradas UID → nombre. (El nodo `ocupados` existente queda intacto.)

---

### Task 3: Obtener la `firebaseConfig` [Lucas → Claude]

**Objetivo:** conseguir los datos de conexión que Auth necesita.

- [ ] **Step 1:** Consola → ⚙️ **Configuración del proyecto** → pestaña **General** → bajar hasta **Tus apps** → la app web → **Configuración de SDK** → opción **Configuración**.
- [ ] **Step 2:** Copiar el bloque `const firebaseConfig = { ... }` completo (tiene `apiKey`, `authDomain`, `databaseURL`, `projectId`, `appId`, etc.).
- [ ] **Step 3:** Pegárselo a Claude en el chat. Con eso Claude completa la Task 4.

  > Si no hay ninguna "app web" registrada, crear una con el botón **</>** (Agregar app → Web), ponerle un apodo (ej. "panel"), y copiar la config que aparece.

---

### Task 4: Código — agregar el SDK de Auth y la config [Claude]

**Files:**
- Modify: `vns-studio/vns_admin.html` (head, ~líneas 9-10 y 161-164)

- [ ] **Step 1:** Agregar el SDK de Auth justo después del de database (línea ~10):

  ```html
  <script src="https://www.gstatic.com/firebasejs/8.10.1/firebase-app.js"></script>
  <script src="https://www.gstatic.com/firebasejs/8.10.1/firebase-database.js"></script>
  <script src="https://www.gstatic.com/firebasejs/8.10.1/firebase-auth.js"></script>
  ```

- [ ] **Step 2:** Reemplazar la config mínima actual (`var firebaseConfig = { databaseURL: "..." };`) por la completa del paso 3:

  ```js
  var firebaseConfig = {
    apiKey: "PEGAR_DEL_PASO_3",
    authDomain: "PEGAR_DEL_PASO_3",
    databaseURL: "https://vns-studio-default-rtdb.firebaseio.com",
    projectId: "PEGAR_DEL_PASO_3",
    appId: "PEGAR_DEL_PASO_3"
  };
  ```

  (Claude reemplaza los `PEGAR_DEL_PASO_3` con los valores reales que pasó Lucas.)

- [ ] **Step 3 (verificación manual):** Abrir `vns_admin.html` en el navegador. La consola del navegador (F12) **no debe** mostrar errores de Firebase al cargar. La pantalla de login aparece normal.

---

### Task 5: Código — reescribir login, identidad y logout [Claude]

**Files:**
- Modify: `vns-studio/vns_admin.html` (formulario de login ~84-104; `var CREDS` ~181; `doLogin` ~197-217; `logout` ~225-236)

**Interfaces:**
- Consume: `db` (Firebase database), `firebase.auth()`, nodo `/barberos/{uid}`.
- Produce: `userActivo` = nombre del barbero (string), igual que antes, así el
  resto del panel (`getOcBarbero`, `guardarEnFirebase`) no cambia.

- [ ] **Step 1:** En el formulario de login, reemplazar el bloque del `<select id="loginUser">` (con su `<label>Barbero</label>`) por un input de email:

  ```html
  <label class="login-label">Email</label>
  <input class="login-input" type="email" id="loginEmail" placeholder="tu-email" autocomplete="username">
  <label class="login-label">Contraseña</label>
  <input class="login-input" type="password" id="loginPass" placeholder="••••••••" autocomplete="current-password" onkeydown="if(event.key==='Enter')doLogin()">
  ```

- [ ] **Step 2:** Borrar la línea de credenciales falsas:

  ```js
  var CREDS = { 'Lauti': 'vnsstudio', 'Santi': 'vnsstudio' };   // ← ELIMINAR
  ```

- [ ] **Step 3:** Reemplazar `doLogin()` entero por la versión con Firebase Auth:

  ```js
  function doLogin() {
    var email = document.getElementById('loginEmail').value.trim();
    var pass  = document.getElementById('loginPass').value;
    if (!email || !pass) { mostrarError('Completá email y contraseña.'); return; }
    firebase.auth().signInWithEmailAndPassword(email, pass)
      .then(function(cred) {
        return db.ref('barberos/' + cred.user.uid).once('value');
      })
      .then(function(snap) {
        var nombre = snap.val();
        if (!nombre) { firebase.auth().signOut(); mostrarError('Tu cuenta no tiene un barbero asignado.'); return; }
        userActivo = nombre;
        document.getElementById('loginWrap').style.display = 'none';
        document.getElementById('panel').style.display = 'block';
        document.getElementById('userLabel').textContent = nombre;
        document.getElementById('panelTitle').textContent = 'Mi agenda — ' + nombre;
        dbListener = db.ref('ocupados').on('value', function(s) {
          fbData = s.val() || {};
          renderCal();
          if (diaS) renderSlots();
        });
        renderCal();
      })
      .catch(function(err) {
        var msg = 'Email o contraseña incorrectos.';
        if (err.code === 'auth/too-many-requests') msg = 'Demasiados intentos. Esperá unos minutos.';
        mostrarError(msg);
      });
  }
  ```

- [ ] **Step 4:** Reemplazar `logout()` para que cierre sesión en Firebase y limpie el email:

  ```js
  function logout() {
    if (dbListener) db.ref('ocupados').off('value', dbListener);
    firebase.auth().signOut();
    userActivo = null; diaS = null; fbData = {};
    document.getElementById('loginPass').value = '';
    document.getElementById('loginEmail').value = '';
    document.getElementById('loginError').style.display = 'none';
    document.getElementById('panel').style.display = 'none';
    document.getElementById('loginWrap').style.display = 'flex';
    document.getElementById('slotsContenido').innerHTML = '';
    document.getElementById('slotsFecha').textContent = '← Seleccioná un día';
    document.getElementById('slotsHint').textContent = 'para gestionar tus turnos';
  }
  ```

- [ ] **Step 5 (verificación manual, reglas TODAVÍA viejas):** Abrir `vns_admin.html` local. Probar:
  - Login con email/contraseña **incorrectos** → muestra el error.
  - Login con la cuenta de Lauti → entra, arriba dice "Mi agenda — Lauti", y se ven los turnos.
  - Marcar/desmarcar un horario → se guarda (aparece "Guardado en Firebase").
  - **Logout** → vuelve al login.

---

### Task 6: Desplegar el panel nuevo [Lucas]

**Objetivo:** subir el código a GitHub Pages. Todavía con las reglas viejas, así nada se rompe.

- [ ] **Step 1:** En la carpeta `vns-studio/`:
  ```bash
  git add vns_admin.html docs/superpowers/
  git commit -m "Seguridad: login del panel con Firebase Auth"
  git push
  ```
- [ ] **Step 2:** Esperar ~1 min a que GitHub Pages publique. Abrir el panel **en vivo** (`https://lucascolari.github.io/vns-studio/vns_admin.html`) y repetir las pruebas del paso 5 de la Task 5 sobre el sitio publicado.

**Verificación:** el login real funciona en el sitio en vivo.

---

### Task 7: Publicar las reglas nuevas [Lucas]

**Objetivo:** cerrar la base. Recién ahora, con el panel nuevo ya desplegado.

- [ ] **Step 1:** Consola → Realtime Database → **Reglas** → reemplazar TODO por:

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
- [ ] **Step 2:** Botón **Publicar**.

**Verificación:** las reglas se publican sin error de sintaxis.

---

### Task 8: Verificación final [Lucas + Claude]

**Objetivo:** confirmar que todo quedó seguro y funcionando.

- [ ] **Step 1 — La web pública sigue OK:** abrir `index.html` en vivo y confirmar que muestra la disponibilidad de turnos como siempre (lectura pública intacta).
- [ ] **Step 2 — Un barbero edita lo suyo:** loguear como Lauti en el panel en vivo, marcar un horario → se guarda OK.
- [ ] **Step 3 — Un barbero NO puede tocar lo del otro:** estando logueado como Lauti, en la consola del navegador (F12) ejecutar:
  ```js
  db.ref('ocupados/Santi_2099-01-01').set(['9:00'])
    .then(()=>console.log('PROBLEMA: lo permitió'))
    .catch(e=>console.log('OK, rechazado:', e.code));
  ```
  Esperado: **`OK, rechazado: PERMISSION_DENIED`**.
- [ ] **Step 4 — Un extraño sin login NO puede escribir:** en una ventana de incógnito (sin login), ejecutar en la consola del navegador:
  ```js
  fetch('https://vns-studio-default-rtdb.firebaseio.com/ocupados/hack_2099.json', {method:'PUT', body:'["9:00"]'})
    .then(r=>console.log('status', r.status));
  ```
  Esperado: **status `401`** (o similar de permiso denegado). La escritura no se hace.
- [ ] **Step 5 — Contraseña vieja muerta:** confirmar que `vnsstudio` ya no sirve para entrar (no existe más como login).

**Si todo esto pasa, VNS queda seguro.** Cierre del trabajo de VNS; neaboats va aparte.
