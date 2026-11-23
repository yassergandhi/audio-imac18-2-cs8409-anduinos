# Audio interno Cirrus CS8409 en Apple iMac18,2 — AnduinOS 2.0.6

**Estado:** funcionamiento físico de los altavoces internos **confirmado por el usuario el 23 de noviembre de 2026**.

Este documento reproduce la instalación mediante el paquete Debian (`.deb`) **preparador** y su posterior generación e instalación del paquete **DKMS** del controlador de audio. Está dirigido a la configuración concreta que funcionó; no certifica compatibilidad con otros iMac, distribuciones o versiones del kernel.

## 1. Equipo y resultado documentado

| Elemento | Valor |
|---|---|
| Equipo | Apple iMac de 2017, **iMac18,2** |
| CPU / arquitectura | Intel Core i5-7400 / `x86_64` (`amd64`) |
| Sistema operativo | **AnduinOS 2.0.6** |
| Kernel probado | **`7.0.0-44-generic`** |
| Códec de audio | Cirrus Logic **CS8409**, a través de Intel HDA |
| Módulo Linux | `snd_hda_codec_cs8409` |
| Método | Controlador alternativo compilado localmente e instalado con **DKMS** |
| Resultado | Salida estéreo por los **altavoces físicos internos** confirmada por el usuario |

**Evidencia aportada por el usuario:**

```console
$ modinfo -n snd_hda_codec_cs8409
/lib/modules/7.0.0-44-generic/updates/dkms/snd-hda-codec-cs8409.ko.zst

$ speaker-test -c 2 -t wav -l 1
El dispositivo de reproducción es default
Los parámetros del flujo son 48000Hz, S16_LE, 2 canales
0 - Front Left
1 - Front Right
```

La ruta bajo `updates/dkms/` indica que `modinfo` resuelve al módulo alternativo instalado por DKMS, no al controlador incluido en la distribución. La salida de `speaker-test` confirma que ALSA envió una prueba estéreo; la **audición física** se considera confirmada por el testimonio directo del usuario, no por la salida del comando por sí sola. No se ha documentado todavía el funcionamiento del micrófono, de la salida de auriculares ni de otros kernels.

## 2. Archivos y origen del controlador

Hay **dos paquetes distintos**:

1. `imac18-2-cs8409-anduinos-preparador_1.0-1_all.deb`: paquete inicial que instala la herramienta `imac-cs8409-anduinos`. **No incluye un módulo precompilado**.
2. `snd-hda-codec-cs8409-apple-dkms_0.1+gitb08d074-1_all.deb`: paquete del **controlador DKMS** que el preparador construye en el propio iMac después de comprobar el hardware, el kernel y la compatibilidad de estructuras internas (ABI/BTF).

La fuente del segundo paquete procede de:

- Repositorio: <https://github.com/armin-haghi/imac-cs8409-linux-audio>
- Revisión fijada: `b08d0745e1b78d6949d5a231d91351e73ddf8079`

**No confundir** este procedimiento con el kit anterior `CS8409-AnduinOS-Debian-kit.zip`, que empleaba el repositorio `nullaroni/imac-cs8409` y generaba un paquete distinto, sin DKMS. El procedimiento que se documenta aquí utiliza el **preparador DKMS**, identificado por la ruta de módulo observada tras la instalación.

## 3. Instalación desde cero

> **No ejecutar de nuevo sobre el equipo que actualmente tiene audio funcional.** Estas instrucciones sirven para una instalación limpia, una reinstalación controlada o para archivar el procedimiento probado.
>
> **Condición de aplicación:** `iMac18,2`, AnduinOS 2.0.6, kernel **exacto** `7.0.0-44-generic`. Si cambian estos datos, detenerse y volver a validar compatibilidad.

### Paso 1. Confirmar el entorno

```bash
cat /sys/class/dmi/id/product_name
cat /etc/os-release
uname -r
dpkg --print-architecture
grep -H 'Codec:' /proc/asound/card*/codec* 2>/dev/null
```

Se espera encontrar `iMac18,2`, AnduinOS, `7.0.0-44-generic`, `amd64` y el códec CS8409. La aparición del dispositivo en ALSA **no garantiza** que suenen los altavoces internos.

### Paso 2. Preparar APT y las cabeceras del kernel

```bash
sudo apt update
sudo apt install -y "linux-headers-$(uname -r)"
```

Si APT no encuentra las cabeceras **exactas** para el kernel iniciado, **no usar cabeceras de otro kernel** ni saltar esta comprobación.

### Paso 3. Instalar el `.deb` preparador

Descargar el archivo `imac18-2-cs8409-anduinos-preparador_1.0-1_all.deb` de esta conversación y colocarlo en `~/Descargas` (o cambiar de directorio a su ubicación real). Después:

```bash
cd ~/Descargas
sudo apt install ./imac18-2-cs8409-anduinos-preparador_1.0-1_all.deb
```

APT instalará también las dependencias declaradas por el paquete, entre ellas `git`, `build-essential`, `dkms`, `dwarves`, `zstd`, `mokutil` y `xdg-user-dirs`. **No** instala aún el controlador de sonido.

### Paso 4. Diagnosticar antes de compilar

```bash
imac-cs8409-anduinos diagnostico
```

Revisar especialmente: modelo, versión de kernel, códec, disponibilidad de cabeceras, BTF, Secure Boot y módulo actualmente seleccionado.

### Paso 5. Compilar, generar e instalar el `.deb` del controlador

**Opción directa (la más breve):**

```bash
imac-cs8409-anduinos instalar
```

**Opción en dos pasos (para conservar e inspeccionar el `.deb` antes de instalarlo):**

```bash
imac-cs8409-anduinos construir

# La ruta depende de la carpeta de descargas configurada en GNOME.
# En AnduinOS en español, normalmente es ~/Descargas.
sudo apt install ~/Descargas/snd-hda-codec-cs8409-apple-dkms_0.1+gitb08d074-1_all.deb
```

**Elegir solo una opción; no ejecutar ambas.** El comando se inicia como usuario ordinario, **sin `sudo` delante de `imac-cs8409-anduinos`**; la herramienta solicita privilegios únicamente al instalar con APT.

El preparador:

1. Rechaza hardware, arquitectura o kernel diferentes de los previstos.
2. Comprueba el códec, las cabeceras, BTF y el estado de Secure Boot.
3. Descarga el código fuente del repositorio indicado y fija el commit señalado.
4. Ejecuta la comprobación ABI/BTF (`verify-abi.py`).
5. Compila el módulo contra el kernel iniciado y comprueba su `vermagic`.
6. Construye el paquete `.deb` DKMS y, en modo `instalar`, lo instala mediante APT.

**Si la comprobación ABI falla, detener el procedimiento.** No establecer `CS8409_SKIP_ABI_CHECK=1`, no forzar el módulo ni sustituir archivos directamente en `/lib/modules/`.

### Paso 6. Reiniciar

Solo si la instalación del controlador concluyó correctamente:

```bash
sudo reboot
```

### Paso 7. Verificar y escuchar

```bash
# Ruta del controlador seleccionado
modinfo -n snd_hda_codec_cs8409

# Estado del paquete DKMS
dkms status -m snd-hda-codec-cs8409-apple

# Comprobación de paquetes APT
dpkg-query -W imac18-2-cs8409-anduinos-preparador \
  snd-hda-codec-cs8409-apple-dkms

# Prueba estéreo por la salida predeterminada
speaker-test -c 2 -t wav -l 1
```

La ruta del módulo debe corresponder a la versión **actual** del kernel y apuntar a `/updates/dkms/`, como en el resultado ya confirmado. Se deben oír físicamente las locuciones izquierda y derecha por los **altavoces internos** del iMac. Si la salida predeterminada no es la interna, comprobar el dispositivo seleccionado en **Configuración → Sonido** antes de extraer conclusiones.

## 4. Comprobaciones en la instalación que YA funciona

No es necesario reinstalar. Para documentar el estado del equipo:

```bash
uname -r
modinfo -n snd_hda_codec_cs8409
dkms status -m snd-hda-codec-cs8409-apple
dpkg-query -W imac18-2-cs8409-anduinos-preparador \
  snd-hda-codec-cs8409-apple-dkms
speaker-test -c 2 -t wav -l 1
```

Se recomienda conservar ambos `.deb`, este `README.md` y, cuando se obtengan, las salidas de `dkms status` y `dpkg-query -W`. La ruta de `modinfo` **ya consta**; las versiones exactas reportadas por DKMS y APT todavía no se han recibido como evidencia independiente.

## 5. Actualizaciones del kernel: precaución importante

Este procedimiento se probó **únicamente** con `7.0.0-44-generic`. DKMS está diseñado para recompilar módulos cuando se instalan otros kernels, pero **no existe garantía de que un kernel nuevo conserve una ABI compatible ni de que el audio continúe funcionando**. El proyecto incorpora una comprobación de ABI que debe bloquear la compilación si no puede verificarla o detecta incompatibilidad.

Antes de reiniciar tras una actualización importante, comprobar:

```bash
uname -r
ls /lib/modules/
dkms status -m snd-hda-codec-cs8409-apple
```

Si el nuevo kernel aparece sin un módulo DKMS instalado, no intentar copiar el `.ko.zst` del kernel anterior. Conservar el kernel funcional `7.0.0-44-generic` como posibilidad de recuperación hasta probar el nuevo; no eliminarlo automáticamente junto con los antiguos.

El preparador `1.0-1` **restringe expresamente la construcción inicial al kernel `7.0.0-44-generic`**. Para preparar otro kernel se necesita una revisión específica del procedimiento; no basta con cambiar su número en los scripts.

## 6. Desinstalación y recuperación

### Retirar el controlador alternativo

```bash
sudo apt remove snd-hda-codec-cs8409-apple-dkms
sudo reboot
```

Esto retira el controlador DKMS instalado, **sin modificar particiones ni entradas EFI**. El audio interno puede dejar de funcionar con el controlador estándar.

Opcionalmente, para retirar también la herramienta preparadora:

```bash
sudo apt remove imac18-2-cs8409-anduinos-preparador
```

### Si el sistema no arranca tras cargar el módulo

En el menú de GRUB, editar temporalmente la entrada de arranque mediante `e`. Al final de la línea que comienza por `linux`, añadir:

```text
modprobe.blacklist=snd_hda_codec_cs8409
```

Arrancar la entrada editada con `Ctrl+X`; si se inicia el sistema, retirar el paquete DKMS como se indicó arriba. **Esta es una medida de recuperación, no una configuración permanente.** El acceso al menú de GRUB depende del arranque EFI instalado en el equipo y puede requerir seleccionar manualmente el sistema o un kernel anterior. Si no es posible entrar en GRUB, usar un medio de rescate adecuado; no borrar particiones ni entradas de arranque para resolver un problema del módulo de audio.

## 7. Alcance, seguridad y trazabilidad

- **[P] Evidencia técnica aportada:** `modinfo -n` resolvió a `/lib/modules/7.0.0-44-generic/updates/dkms/snd-hda-codec-cs8409.ko.zst`, y `speaker-test` emitió dos canales a 48 kHz, S16_LE.
- **[T] Funcionamiento confirmado:** el usuario declaró que los altavoces internos del iMac **sí reproducen sonido físicamente**.
- **[V] Pendientes:** comprobar si el sonido permanece tras futuras actualizaciones del kernel; verificar micrófono, auriculares y otras salidas/entradas si se necesitan.
- **Restricciones:** no modificar ni formatear el disco interno Apple, su partición `DATOS`, las instalaciones Linux anteriores ni las entradas EFI existentes. La instalación descrita solo actúa sobre paquetes y módulos del sistema operativo iniciado.
- **Secure Boot:** el preparador bloquea el proceso si detecta Secure Boot habilitado, pues el controlador alternativo no se proporciona firmado. No se recomienda desactivar controles de firmware automáticamente.
- **Reproducibilidad:** usar los nombres y versiones exactos de los paquetes y conservar la revisión de código fuente indicada. Descargar la rama más reciente de otro repositorio **no reproduce** esta instalación.

---

**Registro:** 23 de noviembre de 2026 · Apple iMac18,2 · AnduinOS 2.0.6 · Linux `7.0.0-44-generic` · audio interno funcional.
