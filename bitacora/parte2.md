\# Parte 2. Espacio de usuario y dispositivos



\## Requisitos del entorno



\## Actividad 1. Compilar BusyBox

\### P1



El archivo `busybox` pesa aproximadamente 2,5 MB y la instalación creó 402 enlaces simbólicos.



En la Parte 1, el programa `init` compilado pesaba aproximadamente 822 KB, por lo que BusyBox es más grande. La diferencia se debe a que BusyBox incluye cientos de comandos y utilidades dentro de un solo ejecutable, mientras que el `init` de la Parte 1 era un programa pequeño creado únicamente para iniciar el sistema y mostrar información básica.



\### P2



`rootfs/bin/ls` apunta al archivo `busybox`.



BusyBox es un binario multipropósito que contiene varios comandos dentro del mismo ejecutable. Los archivos como `ls`, `cp` y `mount` son enlaces simbólicos que apuntan a `busybox`. Cuando se ejecuta uno de esos nombres, BusyBox identifica con qué nombre fue llamado y ejecuta la función correspondiente.



Por ejemplo, si se ejecuta `ls`, BusyBox actúa como el comando `ls`; si se ejecuta `cp`, actúa como `cp`; y si se ejecuta `mount`, ejecuta la función de montaje.

\#	



\## Actividad 2. Armar la jerarquía y escribir /init

\### P3



Las carpetas creadas en `rootfs` tienen funciones diferentes:



\- `/dev`: contiene los archivos que representan dispositivos.

\- `/proc`: muestra información de los procesos y del estado del sistema.

\- `/sys`: expone información sobre dispositivos, controladores y hardware reconocido por el kernel.

\- `/etc`: se utiliza para archivos de configuración del sistema.

\- `/tmp`: se utiliza para archivos temporales.

\- `/root`: es el directorio personal del usuario `root`.



En el archivo `rootfs.cpio.gz`, las carpetas `/proc`, `/sys` y `/dev` estarán inicialmente vacías. Durante el arranque, el script `/init` monta `proc`, `sysfs` y `devtmpfs` sobre esas carpetas, y el kernel genera su contenido en ese momento.



\### P4



El script utiliza `exec` en la última línea para reemplazar el proceso que está ejecutando `/init` por la terminal `sh`, en lugar de crear un proceso nuevo.



Como `/init` es ejecutado inicialmente como PID 1, al usar `exec` la terminal conserva ese mismo PID. Por lo tanto, se espera que la terminal `sh` tenga \*\*PID 1\*\*.





\## Actividad 3. Arrancar y comprobar quién es PID 1



\### P5



El comando `echo $$` muestra \*\*1\*\*, por lo que la terminal `sh` está ejecutándose como PID 1. Esto coincide con la predicción de la P4, ya que `exec` reemplazó el proceso `/init` por la shell sin crear un proceso nuevo.



Al ejecutar `ps` aparecen la shell `sh`, el propio comando `ps` y varios procesos internos del kernel. Los nombres que aparecen entre corchetes, por ejemplo `\[kworker/...]`, `\[oom\_reaper]` o `\[kcompactd0]`, corresponden a hilos del kernel y no a programas normales del espacio de usuario.



\###P6



Al escribir exit, la terminal sh terminó y apareció el mensaje:



Kernel panic - not syncing: Attempted to kill init!



Esto ocurre porque la terminal estaba ejecutándose como PID 1. Al terminar el proceso inicial del espacio de usuario, el kernel no puede continuar normalmente.



Es el mismo comportamiento observado en la Parte 1 cuando el programa /init terminó: en ambos casos el sistema pierde su proceso PID 1 y el kernel entra en panic.



\### P7

\### P7



La salida de `mount` mostró que el script `/init` montó tres sistemas de archivos virtuales:



\- `proc` en `/proc`

\- `sysfs` en `/sys`

\- `devtmpfs` en `/dev`



`/proc` permite consultar información sobre los procesos y el estado del sistema. `/sys` muestra información del modelo de dispositivos y controladores del kernel. `/dev` contiene los archivos que representan los dispositivos detectados por el kernel.



Estos tres montajes fueron realizados por las líneas `mount` del script `/init` durante el arranque.





\## Actividad 4. Explorar /proc y /sys



\### P8



Después de ejecutar `umount /proc`, el comando `ps` dejó de mostrar los procesos y solo apareció el encabezado de la tabla.



Esto ocurre porque `ps` obtiene la información de los procesos desde el sistema de archivos virtual `/proc`. Al desmontarlo, esa información deja de estar disponible.



Después de volver a montar `/proc` con `mount -t proc proc /proc`, el comando `ps` volvió a mostrar los procesos normalmente.



\### P9



En `/proc/1/status` el proceso PID 1 aparece con:



\- `Name`: `sh`

\- `State`: `R (running)`

\- `PPid`: `0`

\- `VmRSS`:  `1804 kB`



Esto confirma que la terminal `sh` es actualmente el proceso PID 1 del sistema. Su proceso padre es 0 porque fue iniciada directamente como proceso inicial del espacio de usuario, y está usando aproximadamente 1804 kB de memoria física.



\### P10



`ls -l /proc/meminfo` muestra que el archivo tiene tamaño 0, pero al leerlo con `cat` o `head` sí aparece información.



Esto ocurre porque `/proc` es un sistema de archivos virtual. El archivo no guarda datos físicamente como un archivo normal del disco. El kernel genera su contenido en el momento en que el archivo es leído, por eso puede tener tamaño 0 y aun así mostrar información.



\### P11



QEMU fue configurado con 256 MB de memoria, pero `/proc/meminfo` reportó:



`MemTotal: 220076 kB`



El valor es menor porque una parte de la memoria queda reservada para el propio kernel, estructuras internas y recursos necesarios durante el arranque. Por eso no toda la memoria asignada por QEMU aparece como memoria disponible para los procesos.



\### P12



En `/sys/class/net` aparecen las interfaces de red:



\- `eth0`

\- `lo`

\- `sit0`



La interfaz `eth0` corresponde a la interfaz de red principal del sistema, `lo` es la interfaz de loopback y `sit0` es una interfaz virtual relacionada con túneles.



En `/sys/block` aparecen varios dispositivos de bloque:



\- `loop0` a `loop7`

\- `sr0`



Estos dispositivos no fueron creados manualmente por mí. El kernel los detectó o creó durante el arranque y los expone a través de `sysfs`, que está montado en `/sys`.



\## Actividad 5. Archivos de dispositivo

\### P13



Con `devtmpfs` montado había \*\*111 archivos\*\* en `/dev`.



Después de desmontarlo con `umount -l /dev`, quedó solamente el archivo `console`. Este archivo ya estaba incluido dentro del `rootfs.cpio.gz`, porque fue creado manualmente antes de empaquetar el sistema.



Después de volver a montar `devtmpfs`, `/dev` volvió a tener \*\*111 archivos\*\*, ya que el kernel volvió a crear automáticamente los dispositivos detectados.



\### P14



Aunque `/dev/mi\_consola` no se llama `console`, al escribir sobre él el texto apareció en la pantalla.



Esto ocurre porque un dispositivo no se identifica realmente por el nombre del archivo, sino por su tipo y por sus números mayor y menor.



Tanto `/dev/console` como `/dev/mi\_consola` eran dispositivos de carácter con los números `5,1`, por lo que ambos representaban el mismo dispositivo de consola.



\### P15



Con `mknod` los archivos de dispositivo deben crearse manualmente indicando su tipo y sus números mayor y menor. En cambio, `devtmpfs` permite que el kernel cree automáticamente los archivos correspondientes a los dispositivos que detecta.



En un sistema con cientos de dispositivos, crearlos manualmente sería poco práctico y propenso a errores, ya que habría que conocer y configurar correctamente cada dispositivo. `devtmpfs` automatiza este proceso.



\## Actividad 6. Entrada/salida e interrupciones



\### P16

\### P16



El contador de interrupciones de `ttyS0` pasó de \*\*3513\*\* a \*\*3894\*\*, por lo que aumentó en \*\*381 interrupciones\*\*.



El puerto serie `ttyS0` utiliza la \*\*IRQ 4\*\*.



Cada vez que se genera una interrupción, el hardware avisa al procesador de que ocurrió un evento. En este caso, al presionar teclas en la consola serie, el kernel atiende esas interrupciones para recibir los caracteres enviados por `ttyS0`.



\### P17



El primer comando `dd` reportó una velocidad aproximada de \*\*1,7 GB/s\*\*.



La velocidad es tan alta porque los datos se transfieren entre `/dev/zero` y `/dev/null`, que son dispositivos virtuales de carácter. No se está leyendo ni escribiendo en un disco físico: `/dev/zero` genera ceros y `/dev/null` descarta inmediatamente todo lo que recibe.



\### P18



El segundo comando creó un archivo de \*\*20 MB\*\* en `/tmp`.



Como `/tmp` no tiene un disco montado detrás, ese archivo quedó almacenado en la memoria RAM del sistema, dentro del `rootfs`.



Antes de crear el archivo, `MemFree` era aproximadamente \*\*205648 kB\*\* y después quedó en \*\*185356 kB\*\*. La diferencia fue:



`205648 - 185356 = 20292 kB`



Esto corresponde aproximadamente a 20 MB, por lo que la disminución de memoria libre confirma que el archivo fue almacenado en RAM.



\## Trabajo autónomo



\### 1. Jerarquía del `rootfs`



El árbol de directorios obtenido con `find rootfs -maxdepth 2 -type d` fue:



```text

rootfs

├── sbin

├── proc

├── etc

├── dev

├── bin

├── sys

├── usr

│   ├── sbin

│   └── bin

├── root

├── rootls

└── tmp

```



Cada carpeta cumple una función:



\- `rootfs/bin`: contiene comandos básicos de BusyBox, como `ls`, `cp`, `cat` y `sh`.

\- `rootfs/sbin`: contiene comandos de administración.

\- `rootfs/usr/bin`: contiene otros comandos disponibles para el usuario.

\- `rootfs/usr/sbin`: contiene utilidades adicionales de administración.

\- `rootfs/dev`: es el punto donde se montan los dispositivos mediante `devtmpfs`.

\- `rootfs/proc`: es el punto de montaje de `proc`, donde se consulta información de procesos y del sistema.

\- `rootfs/sys`: es el punto de montaje de `sysfs`, donde se expone información sobre dispositivos, controladores y hardware.

\- `rootfs/etc`: está destinado a archivos de configuración.

\- `rootfs/tmp`: se utiliza para archivos temporales.

\- `rootfs/root`: es el directorio personal del usuario `root`.



2\. Montajes realizados por /init

Punto de montaje	Sistema de archivos	Información que expone	Comando relacionado en la práctica

/proc	proc	Información de procesos y del estado del sistema	ps dejó de mostrar procesos cuando /proc fue desmontado

/sys	sysfs	Información sobre dispositivos, controladores, buses y hardware	ls /sys/class/net y ls /sys/block permitieron consultar dispositivos

/dev	devtmpfs	Archivos que representan los dispositivos detectados por el kernel	Al desmontarlo desaparecieron la mayoría de los dispositivos; al volver a montarlo reaparecieronEstos montajes permiten que el espacio de usuario consulte información que mantiene el kernel sin necesidad de almacenarla en archivos normales del disco.



3\. El archivo `rootfs/init` fue comentado línea por línea para explicar, con mis propias palabras, la función de cada instrucción utilizada durante el arranque del sistema.





