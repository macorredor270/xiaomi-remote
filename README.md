<p align="center">
  <b>XiaomiRemote</b><br>
  Convierte tu iPhone en el mando de tu TV Xiaomi / Android TV — sin nube, sin cuenta Mi Home,
  sin servidor de por medio.
</p>

<p align="center">
  <a href="https://github.com/macorredor270/xiaomi-remote/releases/latest"><img alt="Release" src="https://img.shields.io/github/v/release/macorredor270/xiaomi-remote?label=versi%C3%B3n&color=00d4ff"></a>
  <img alt="Plataforma" src="https://img.shields.io/badge/iOS-16%2B-00d4ff">
  <img alt="Swift" src="https://img.shields.io/badge/swift-5.9%20%2F%20SwiftUI-00d4ff">
  <img alt="Protocolo" src="https://img.shields.io/badge/protocolo-Android%20TV%20Remote%20v2-00d4ff">
</p>

---

App nativa de iOS que habla directamente con la TV por Wi-Fi usando el protocolo
**Android TV Remote v2** — el mismo que usa la app oficial de Android TV, implementado desde cero
en Swift. Funciona con cualquier Xiaomi TV, Google TV o Android TV que tenga ese servicio activo.

Nació porque perdía el mando físico todo el rato y me pareció más rápido escribir uno que
levantarme a buscarlo.

## Qué hace

- **Descubre TVs en la red local** vía Bonjour (`_androidtvremote2._tcp`).
- **Empareja con la TV**: implementa el *handshake* "polo" en el puerto `6467` — la TV muestra un
  código de 6 dígitos en pantalla, lo escribes en la app, y un reto SHA-256 sobre las claves de
  certificado de ambos dispositivos confirma el emparejamiento.
- **Envía los comandos** por una conexión TLS persistente en el puerto `6466`: encendido, D-pad,
  OK, teclado numérico, volumen/silencio, atrás/inicio, botón Mi, teclas de color, lista de
  entradas/canales/ajustes/teletexto y el botón de asistente (micrófono).
- **Abre apps** en la TV por deep link (Netflix y Prime Video ya están enlazados en la interfaz).
- Reconecta solo con backoff si se cae la conexión, y pide a iOS un margen de gracia en segundo
  plano para que cambiar de app rápido no mate el socket.
- Recuerda las TVs ya emparejadas (por host, vía `UserDefaults`), así que reabrir la app reconecta
  en silencio en vez de pedir emparejar otra vez.

## Cómo funciona el protocolo

Android TV Remote v2 autentica al cliente con TLS mutuo. La app necesita una identidad TLS propia
(certificado autofirmado + clave privada) para el *handshake* de emparejamiento y para la conexión
de control. No hay librería externa de protobuf: `ProtobufEncoder.swift` es un codificador propio,
pequeño, que escribe solo los campos varint / length-delimited / message que necesita este
protocolo, y `TVRemoteMessages.swift` construye sobre él los frames de emparejamiento e inyección
de teclas.

### Sobre la identidad TLS (por qué no está en el repo)

Antes esta identidad (`tv_identity.p12`) iba comprometida dentro del repositorio. Ya no: es una
clave privada, y publicarla habría dejado a cualquiera con la clave en condiciones de suplantar un
cliente ya emparejado con una TV que la confíe. Ahora **no se versiona** y cada quien genera la
suya:

```sh
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -days 3650 -nodes -subj "/CN=XiaomiRemote"
openssl pkcs12 -legacy -export -out XiaomiRemote/Resources/tv_identity.p12 \
  -inkey key.pem -in cert.pem -passout pass:xiaomiremote
rm key.pem cert.pem
```

La contraseña (`xiaomiremote`) está harcodeada en `TVIdentity.swift` porque el `.p12` va dentro del
propio bundle de la app — no protege nada que ya no esté en tu iPhone. En CI, el workflow
inyecta su propia identidad desde un secreto de GitHub Actions (`TV_P12_BASE64`) en vez de usar
ninguna comprometida en el repo.

## Descargas

En la página de [**Releases**](https://github.com/macorredor270/xiaomi-remote/releases/latest)
hay un IPA ya compilado y firmado con `zsign` — **pero está firmado ad-hoc para mi propio iPhone**
(el certificado de desarrollo registra un UDID concreto), así que no se instalará sin más en otro
dispositivo. Para probarlo en el tuyo, dos caminos:

- **[AltStore](https://altstore.io) / [Sideloadly](https://sideloadly.io)** con tu propia cuenta de
  Apple (gratis): te vale el IPA sin firmar que genera la CI en cada push, o compilarlo tú mismo
  (abajo) y dejar que la herramienta lo firme con tu cuenta.
- **Compilarlo y ejecutarlo** directamente desde Xcode en tu dispositivo (ver más abajo).

## Stack técnico

- **Swift 5.9 / SwiftUI**, iOS 16+
- **Network.framework** (`NWConnection`, `NWBrowser`) para los sockets TLS y el descubrimiento Bonjour
- **CryptoKit** / **Security** para el hash del emparejamiento y la identidad TLS del cliente
- **[XcodeGen](https://github.com/yonaskolb/XcodeGen)** — el proyecto de Xcode se genera desde
  `project.yml`, no está en el repositorio
- **Fastlane** — firma y registro de dispositivos personal (`fastlane/Fastfile`)
- **GitHub Actions** — construye un IPA sin firmar en cada push a `main`, y otro workflow arranca
  el Simulador de iOS para capturar pantallas de la interfaz

También hay una carpeta `preview-swift/`: una vista previa SwiftUI independiente (vía
[Tokamak](https://github.com/TokamakUI/Tokamak)) compilada a WebAssembly, para iterar el diseño del
mando en el navegador sin necesitar un Mac. Ver `preview-swift/README.md`.

## Requisitos

- Xcode 15+ (objetivo iOS 16.0)
- [XcodeGen](https://github.com/yonaskolb/XcodeGen) (`brew install xcodegen`)
- Una Xiaomi TV / Google TV / Android TV con el servicio Android TV Remote activo, en la misma
  Wi-Fi que el iPhone

## Compilar y ejecutar

```sh
git clone https://github.com/macorredor270/xiaomi-remote.git
cd xiaomi-remote
```

Genera tu propia identidad TLS (ver arriba, [Sobre la identidad TLS](#sobre-la-identidad-tls-por-qué-no-está-en-el-repo)):

```sh
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -days 3650 -nodes -subj "/CN=XiaomiRemote"
openssl pkcs12 -legacy -export -out XiaomiRemote/Resources/tv_identity.p12 -inkey key.pem -in cert.pem -passout pass:xiaomiremote
rm key.pem cert.pem
```

Genera el proyecto y ábrelo:

```sh
xcodegen generate
open XiaomiRemote.xcodeproj
```

Compila y ejecuta el esquema `XiaomiRemote` en el Simulador o en un dispositivo. La firma está
desactivada por defecto en `project.yml` (`CODE_SIGNING_ALLOWED: NO`), suficiente para el
Simulador; para un iPhone físico apunta tu propia cuenta/equipo en *Signing & Capabilities* de
Xcode.

En el primer arranque: abre **Ajustes**, pulsa **Buscar dispositivos** para descubrir TVs en tu red
(o mete una IP a mano), luego **Conectar / Emparejar** y escribe el código que aparece en la
pantalla de la TV.

> La interfaz está en español (el idioma del autor).

## Estructura del proyecto

```
XiaomiRemote/
  App/                  Punto de entrada
  Models/               Estado de la app (TVState/UserDefaults), mapeo de teclas
  Network/               Descubrimiento Bonjour, clientes TLS de emparejamiento y control,
                          codificador protobuf, constructor de mensajes, identidad TLS
  Views/                Pantallas de mando, ajustes y emparejamiento (SwiftUI)
  Resources/             Identidad TLS (tv_identity.p12 — no versionada, ver arriba)
preview-swift/           Vista previa SwiftUI en navegador (Tokamak + Wasm)
fastlane/                Lane de firma personal
scripts/firmar.sh        Ayudante personal de sideloading (AltServer/Sideloader)
project.yml               Definición del proyecto para XcodeGen
```

## Sobre el proyecto

Implementación del protocolo Android TV Remote v2 hecha desde cero: sin SDK oficial, sin librería
de protobuf externa, con la identidad TLS y el *handshake* de emparejamiento escritos a mano
siguiendo el propio protocolo. Pensada para mi TV, así que algunas cosas (IP por defecto, textos en
español) están ajustadas a ese uso, pero la implementación del protocolo debería funcionar con
cualquier dispositivo compatible con Android TV Remote v2.

## Licencia

Sin licencia explícita todavía — el código es visible para consulta y estudio. Si quieres usarlo en
otro proyecto, abre un issue.
