# Vroop Desktop

Aplicación de **escritorio (Windows)** de **Vroop**, un mensajero con **cifrado de extremo a extremo (E2EE)**. Este repo/proyecto contiene la app de escritorio y publica sus instaladores para la **auto-actualización**.

> Forma parte del proyecto Vroop: backend (Spring Boot), web/móvil (Angular + Capacitor) y **escritorio (Electron)**.

---

## ¿Qué es Vroop?

Vroop es una app de mensajería centrada en la privacidad:

- **Cifrado de extremo a extremo (E2EE)** con **MLS** (Messaging Layer Security) mediante [`ts-mls`](https://github.com/LukaJCB/ts-mls).
- El servidor **nunca ve el contenido** de los mensajes: solo transporta `envelopes` cifrados y metadatos mínimos.
- Cada **dispositivo** es una hoja del grupo MLS, lo que permite **varios dispositivos por usuario** con un solo cifrado por mensaje para todos los miembros.
- Emparejamiento de dispositivos e **importación de historial** por **QR/código**, y **revocación** de dispositivos.

La app de escritorio (este proyecto) es el cliente **Electron** que empaqueta el mismo frontend Angular que usan web y móvil.

---

## Objetivos

1. **Privacidad real**: mensajes cifrados de extremo a extremo; ni el backend ni terceros pueden leerlos.
2. **Multi-dispositivo**: el mismo usuario en móvil, web y escritorio, sincronizados.
3. **Experiencia nativa en escritorio**: ventana propia, auto-actualización silenciosa, sin instaladores manuales.
4. **Seguridad por defecto**: claves generadas y guardadas en el dispositivo, material sensible cifrado en reposo.

---

## Características

- Registro e inicio de sesión (Supabase Auth).
- **Mensajes directos (MD)** y **canales** de servidor, todos E2EE.
- **Amigos** y **solicitudes**.
- **Servidores** con canales y miembros, con **presencia** en vivo.
- **Adjuntos cifrados** (se suben como bytes ya cifrados; el servidor no puede leerlos).
- **Emparejamiento de dispositivos** e **importación de historial** (QR/código).
- **Revocación** de dispositivos y recuperación de grupos.
- **Auto-actualización**: se actualiza sola, sin diálogos ni instalador visible.

---

## Tecnología

| Capa | Tecnología |
|------|------------|
| Shell de escritorio | **Electron 26** + `@capacitor-community/electron` |
| UI | **Angular 22** (build web empaquetado) |
| E2EE | **MLS** con `ts-mls` + WebCrypto |
| Auto-update | `electron-updater` sobre **GitHub Releases** |
| Empaquetado | `electron-builder` (instalador **NSIS** para Windows) |

---

## Arquitectura (resumen)

```
electron/
├─ src/
│  ├─ index.ts       # Proceso principal: arranque, ventana, IPC y auto-actualización
│  ├─ setup.ts       # ElectronCapacitorApp: ventana principal, menú, CSP, modo por ruta
│  └─ preload.ts     # Puente seguro (expone la versión real de la app)
├─ assets/           # Iconos, splash y ventana de actualización (update.html)
├─ app/              # Build web de Angular empaquetado (generado; no editar a mano)
├─ build/            # JS compilado de TypeScript (generado)
├─ electron-builder.config.json   # Config de empaquetado y publicación
└─ package.json      # Versión de la app y dependencias
```

- **Proceso principal** (`index.ts`): espera a Electron, comprueba actualizaciones (con ventana propia), crea la ventana principal y expone la versión por IPC.
- **Renderer**: la app Angular (misma que web/móvil).
- **Auto-update**: `electron-updater` apunta al repo **`ROMOR12/vroop-desktop`** (público) donde se publican los instaladores y el `latest.yml`.

### Flujo de auto-actualización

1. Al abrir, se muestra una ventana pequeña con el logo y el estado.
2. Se consulta GitHub Releases:
   - Si hay versión nueva → `descargando… %` → `instalando…` → se instala **en silencio** y se reinicia solo.
   - Si está al día → la ventana se cierra y abre la app con normalidad.

---

## Seguridad

- **E2EE (MLS)**: el texto se cifra en el cliente; el servidor solo ve `envelopes`.
- **Identidad por dispositivo**: cada dispositivo tiene su par de claves y su propia hoja en el grupo.
- **Cifrado en reposo**: el keystore local (IndexedDB) cifra con AES-GCM las claves privadas MLS y el estado de las sesiones, usando una clave **no extraíble** guardada aparte.
- **CSP** configurada en Electron para limitar orígenes.
- Sin secretos en el repositorio (las variables sensibles van por entorno / `.env`).

> Nota: durante el desarrollo pueden verse avisos de **Windows SmartScreen** (la app no está firmada con un certificado de pago). Es normal en apps no firmadas.

---

## Instalación

1. Ve a **Releases** de este repositorio: `https://github.com/ROMOR12/vroop-desktop/releases`.
2. Descarga el instalador más reciente (`Vroop Setup <versión>.exe`).
3. Ejecútalo. Si aparece SmartScreen: **Más información → Ejecutar de todas formas**.
4. Abre **Vroop** y a partir de ahí se actualizará solo.

---

## Desarrollo

### Requisitos

- **Node.js** 18+ y **npm**.
- El frontend Angular compilado (se copia a `electron/app` con Capacitor).

### Estructura de trabajo

El proyecto de escritorio vive dentro del repo del frontend, en `vroop-frontend/electron`.

### Comandos (en `electron/`)

```bash
npm install                 # dependencias
npm run build               # compila TypeScript (src -> build)
npm run electron:start      # compila y arranca en modo desarrollo (con DevTools)
npm run electron:pack       # empaqueta sin instalador (carpeta dist/win-unpacked)
npm run electron:make       # genera el instalador (dist/Vroop Setup <versión>.exe)
```

### Publicar una nueva versión

1. Sube la versión en `package.json` (p. ej. `1.0.9`).
2. Compila el frontend Angular y cópialo al proyecto Electron (`npx cap copy @capacitor-community/electron`).
3. Publica:

```bash
# En Windows PowerShell
$env:GH_TOKEN = (& "C:\Program Files\GitHub CLI\gh.exe" auth token).Trim()
npm run build
npx electron-builder build -c ./electron-builder.config.json --publish always
```

Esto crea la release en GitHub con el instalador y el `latest.yml`; los clientes instalados se actualizan solos.

---

## Estado

- Versión actual: **1.0.8**.
- Plataforma principal: **Windows** (hay configuración base de macOS, sin firmar/testear).

---

## Licencia

MIT.
