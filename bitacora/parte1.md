# Parte 1 - Kernel y arranque mínimo

## Actividad 1. Explorar la configuración del kernel

### P1
En General setup → Local version aparece el valor `-lorena`.
Este valor se había configurado anteriormente mediante `LOCALVERSION`.

### P2
En `Processor type and features`, el valor configurado para `Maximum number of CPUs` es **64**. Esto significa que esta configuración del kernel permite manejar hasta 64 CPU.

### P3
En `File systems`, el sistema de archivos ext4 aparece marcado con `<*>`, lo que significa que su soporte está incluido directamente en el kernel.

Si ext4 estuviera configurado como módulo `<M>` y el disco raíz utilizara ext4, el kernel tendría que cargar ese módulo antes de poder montar el sistema de archivos raíz. Para que eso funcione, el módulo tendría que estar disponible en el `initramfs`; de lo contrario, el sistema no podría montar la raíz y el arranque fallaría.


### P4

`BLK_DEV_INITRD` está en `General setup → Initial RAM filesystem and RAM disk (initramfs/initrd) support` y tiene valor `y`, por lo que está habilitado. Esta opción permite usar un `initramfs` durante el arranque, algo que necesitaremos para construir el sistema mínimo.

`SERIAL_8250_CONSOLE` está en `Device Drivers → Character devices → Enable TTY → Serial drivers → 8250/16550 and compatible serial support → Console on 8250/16550 and compatible serial port`. También tiene valor `y`, por lo que está habilitado.

Esta segunda opción permitirá usar la consola serial para ver la salida del kernel y del sistema cuando lo arranquemos en QEMU.

### P5

En la configuración `build-default` quedaron:

- **1635 opciones integradas directamente en el kernel** (`=y`).
- **12 opciones configuradas como módulos** (`=m`).

## Actividad 2. Compilación con tinyconfig

Para esta actividad se generó una segunda configuración del kernel usando `tinyconfig`. A diferencia de `defconfig`, esta configuración parte con casi todas las opciones desactivadas, por lo que fue necesario habilitar manualmente las opciones mínimas requeridas para el proyecto.

Se activaron las siguientes opciones:

- `64BIT`
- `PRINTK`
- `TTY`
- `SERIAL_8250`
- `SERIAL_8250_CONSOLE`
- `BLK_DEV_INITRD`
- `RD_GZIP`
- `BINFMT_ELF`
- `BINFMT_SCRIPT`
- `PROC_FS`
- `SYSFS`
- `DEVTMPFS`

Después se ejecutó `olddefconfig` para completar la configuración y resolver las dependencias necesarias.

Se verificó que las cinco opciones obligatorias quedaran habilitadas con valor `=y`:

```text
CONFIG_BLK_DEV_INITRD=y
CONFIG_PRINTK=y
CONFIG_64BIT=y
CONFIG_BINFMT_ELF=y
CONFIG_SERIAL_8250_CONSOLE=y
```

Luego se compiló el kernel `tinyconfig`. La compilación terminó correctamente y generó el archivo:

```text
build-tiny/arch/x86/boot/bzImage
```

El archivo generado tiene un tamaño aproximado de **977 KB**.

Inicialmente el kernel quedó con el sufijo `-equipo03-tiny`, correspondiente al ejemplo de la guía. Se corrigió la opción `LOCALVERSION` para identificar la compilación propia como:

```text
6.18.60-lorena-tiny
```

Finalmente, se volvió a compilar y se verificó con el comando `file` que el `bzImage` correspondiera a la versión `6.18.60-lorena-tiny`.

##Actividad 3. Arrancar el kernel sin espacio de usuario

### Incidencia durante la Actividad 3

Durante el primer intento de arranque del kernel en QEMU, la máquina virtual dejó de responder después de presentarse un `Kernel panic` relacionado con `IO-APIC + timer doesn't work`.

La terminal quedó capturada por QEMU y la interfaz de la máquina virtual no respondía a los clics, por lo que fue necesario cerrar QEMU o reiniciar la VM para continuar con la actividad.

Se conserva la captura como evidencia del error presentado antes de obtener el comportamiento esperado del taller.

### P6

En el archivo `logs/arranque-sin-init.log` aparecen las siguientes líneas:

```text
Linux version 6.18.60-lorena ...
Command line: console=ttyS0
Memory: 213060K/261624K available ...
```

La línea `Linux version` indica la versión del kernel que está arrancando. En este caso es `6.18.60-lorena`, que corresponde al kernel compilado previamente.

La línea `Command line` muestra los parámetros que QEMU le pasó al kernel durante el arranque. El parámetro `console=ttyS0` indica que los mensajes del kernel se deben mostrar por la consola serial.

La línea `Memory` muestra la memoria que el kernel detectó y la cantidad que queda disponible después de reservar lo necesario para el propio kernel y otras estructuras internas. En esta ejecución aparecen aproximadamente `213060 KB` disponibles de `261624 KB` detectados.

### P7

El kernel sí arrancó y alcanzó a reconocer el hardware, pero se detuvo porque no encontró un sistema de archivos raíz para montar.

En el log aparece el mensaje:

```text
VFS: Unable to mount root fs on unknown-block(0,0)
```

Esto indica que el kernel estaba buscando el sistema de archivos raíz `/`, que es donde normalmente se encuentran los archivos y programas necesarios para continuar el arranque. Como en esta prueba QEMU se ejecutó sin disco y sin `initramfs`, no había ningún sistema raíz disponible.

Por esa razón el kernel no pudo continuar y terminó en un `Kernel panic`.

### P8

No. Un kernel por sí solo no constituye un sistema operativo usable.

En la prueba se observó que el kernel pudo iniciar, reconocer el hardware y mostrar mensajes de arranque, pero después se detuvo porque no encontró un sistema de archivos raíz ni programas de usuario para continuar.

Por eso terminó en un `Kernel panic`. Esto muestra que el kernel necesita trabajar junto con otros componentes, como un sistema de archivos y un proceso inicial (`init`), para que el sistema pueda seguir funcionando y el usuario pueda ejecutar programas.
