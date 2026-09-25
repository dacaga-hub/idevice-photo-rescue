# ipod-photo-extractor

Extraer la fototeca **sincronizada** de un iPod Touch antiguo a tu PC, sin
herramientas de pago ni descargas dudosas, usando la CLI oficial de
[`pymobiledevice3`](https://github.com/doronz88/pymobiledevice3).

> **Estado: caso piloto.** Probado y funcionando sobre un **iPod Touch 3G con
> iOS 5.1.1**, del que se recuperaron 2880 fotos. Documenta lo que de verdad
> funcionó. La idea es ampliarlo a otros dispositivos Apple más adelante.

---

## Resultado clave

En este iPod, las fotos sincronizadas están como **JPG normales** dentro de
`/PhotoData/Sync/` (subcarpetas `NNNSYNCD`). **No hay formato propietario que
decodificar**: se copian tal cual con un solo comando. (En iPods *clásicos* de
rueda de clic el caso es distinto —usan archivos `.ithmb`—, pero eso queda fuera
de este piloto.)

---

## Lo que funcionó (el método)

Con el iPod conectado por USB, **desbloqueado** y en la pantalla de inicio, y el
entorno virtual activo:

```bash
# 1. ¿Conecta el dispositivo? (prueba go/no-go; imprime modelo, iOS, etc.)
python -m pymobiledevice3 lockdown info

# 2. Ver la raíz multimedia (/var/mobile/Media)
python -m pymobiledevice3 afc ls /

# 3. Localizar las fotos sincronizadas (JPGs en subcarpetas SYNCD)
python -m pymobiledevice3 afc ls -r /PhotoData

# 4. Copiar TODO al PC (-i = seguir aunque algún archivo falle)
python -m pymobiledevice3 afc pull /PhotoData/Sync ./out/fotos -i
```

`pull` **solo copia**; nunca borra ni modifica nada en el dispositivo. Los
comandos que escriben en el iPod son otros y explícitos (`rm`, `push`).

### Verificar la copia

```bash
# ¿Cuántos JPG se copiaron?
find ./out/fotos -iname '*.jpg' | wc -l

# ¿Alguno corrupto de 0 bytes? (debería no imprimir nada)
find ./out/fotos -iname '*.jpg' -size 0
```

Abre unos cuantos JPG de distintas carpetas para confirmar que se ven, y
**copia el resultado a un segundo sitio** (disco externo, nube) antes de dar el
rescate por terminado. Un rescate no está hecho hasta que hay dos copias.

---

## Requisitos

Probado en **Windows 11**. Los comandos `pymobiledevice3` son iguales en
macOS/Linux; solo cambian la activación del venv y los drivers.

1. **iTunes instalado** (Windows) — aporta el driver *Apple Mobile Device USB*
   y el servicio `usbmux` que `pymobiledevice3` necesita para hablar por USB.
   Sin ese driver, Windows deja el iPod en modo **MTP** y solo se ve `/DCIM`.
2. **Python 3.10+** y un entorno virtual:

```bash
python -m venv .venv

# Activar el venv:
source .venv/Scripts/activate     # Windows (Git Bash)
# .\.venv\Scripts\Activate.ps1    # Windows (PowerShell)
# source .venv/bin/activate       # macOS / Linux

pip install -r requirements.txt
```

En PowerShell, si bloquea la activación por política de scripts:
`Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned`.

Para salir del venv: `deactivate`.

---

## Si el iPod no aparece (Windows: modo MTP)

Comprueba en el **Administrador de dispositivos** que el iPod figure bajo
*Controladoras de bus serie universal* como **"Apple Mobile Device USB Driver"**.
Si en cambio aparece en *Dispositivos portátiles* como MTP, Windows no cogió el
driver de Apple y por AFC no llegarás a `/PhotoData` (solo verás `/DCIM`).

Lo que lo resolvió: Administrador de dispositivos → clic derecho en el iPod →
**Desinstalar** (marcando quitar el software del controlador) → desconectar →
**reiniciar el PC** → reconectar. Con iTunes instalado, al reconectar coge el
driver de Apple y pasa a *Controladoras de bus serie universal*.

---

## Qué aprendimos

- **Explora el sistema de archivos antes de asumir nada.** El plan inicial daba
  por hecho que las fotos serían archivos propietarios `.ithmb` que habría que
  decodificar. No lo eran: eran JPG. Un `afc ls -r /PhotoData` al principio lo
  habría enseñado y ahorrado medio diseño.
- **La CLI evita el lío async.** `pymobiledevice3` v11 usa una API asíncrona
  (`create_using_usbmux` es coroutine); llamarla como función síncrona falla con
  `coroutine ... was never awaited`. La CLI encapsula todo eso: para un rescate
  puntual, es la vía más simple y fiable.
- **No hacía falta software de pago ni jailbreak.** Solo el driver correcto y AFC.

---

## Notas

- **Calidad:** es la que metió iTunes al sincronizar (aquí, 1152×768). No hay
  original de mayor resolución dentro del iPod; para eso haría falta el ordenador
  desde el que se sincronizó en su día.
- **Fechas:** los nombres `IMG_XXXX.JPG` no garantizan orden cronológico y las
  EXIF pueden faltar. Ordenar por fecha real, si se quiere, es un paso aparte.
- **Privacidad:** `out/` está en `.gitignore`. No subas las fotos recuperadas a
  un repositorio público.

---

## Estructura del repo

```
ipod-photo-extractor/
├── README.md          # este archivo: la historia real + los comandos
├── requirements.txt   # pymobiledevice3
├── .gitignore         # ignora out/, .venv/, etc.
└── .vscode/           # recomendación de extensiones
```