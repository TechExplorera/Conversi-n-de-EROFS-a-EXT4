# Conversión de SYSTEM EROFS a EXT4

## 1. Descripción

Este proyecto documenta el procedimiento para extraer una imagen `system.img` que usa el sistema de archivos EROFS y reconstruirla como una imagen EXT4.

El objetivo es permitir la modificación de archivos dentro de una ROM Android y, posteriormente, preparar una imagen lista para su instalación mediante Fastboot.

Importante: la reconstrucción a EXT4 no garantiza, por sí sola, la compatibilidad con Android. Deben conservarse los permisos, propietarios, atributos extendidos y contextos SELinux originales.

## 2. Estructura del proyecto

```text
system_mod/
├── original/
│   └── system.img
├── extraido/
│   └── Archivos extraídos de EROFS
├── montaje/
│   └── Punto de montaje EXT4
├── nuevo/
│   └── Archivos de salida
└── system_ext4.img
```

## 3. Herramientas necesarias

En Ubuntu o Debian:

```bash
sudo apt update
sudo apt install erofs-utils e2fsprogs
```

### Herramientas utilizadas

- `fsck.erofs`: extracción de imágenes EROFS.
- `mkfs.ext4`: creación de sistemas de archivos EXT4.
- `mount`: montaje de imágenes.
- `cp`: copia de archivos.
- `file`: identificación del sistema de archivos.
- `du` y `df`: comprobación del espacio.

## 4. Preparar los directorios

```bash
mkdir -p ~/system_mod/original
mkdir -p ~/system_mod/extraido
mkdir -p ~/system_mod/montaje
mkdir -p ~/system_mod/nuevo
```

Copiar la imagen original:

```bash
cp system.img ~/system_mod/original/
```

## 5. Identificar la imagen original

```bash
file ~/system_mod/original/system.img
```

Una imagen EROFS puede mostrar algo como esto:

```text
EROFS filesystem
blocksize=12
```

`blocksize=12` significa que utiliza bloques de 4096 bytes (`2^12`).

## 6. Extraer EROFS

Ejecutar:

```bash
fsck.erofs \
  --extract=/home/peta/system_mod/extraido \
  /home/peta/system_mod/original/system.img
```

Si el directorio no existe, crearlo primero:

```bash
mkdir -p ~/system_mod/extraido
```

Comprobar los archivos:

```bash
ls -lah ~/system_mod/extraido
du -sh ~/system_mod/extraido
```

## 7. Crear la imagen EXT4

Primero, determinar el tamaño necesario.

Ejemplo: crear una imagen de 9 GB:

```bash
cd ~/system_mod

dd if=/dev/zero \
  of=system_ext4.img \
  bs=1M count=9216

mkfs.ext4 -F -L system system_ext4.img
```

Nota: el tamaño debe ser suficiente para almacenar todos los archivos, metadatos y espacio adicional. También debe respetar el límite de la partición de destino.

## 8. Montar la imagen EXT4

```bash
mkdir -p ~/system_mod/montaje

sudo mount -o loop \
  ~/system_mod/system_ext4.img \
  ~/system_mod/montaje
```

## 9. Copiar los archivos extraídos

```bash
sudo cp -a \
  ~/system_mod/extraido/. \
  ~/system_mod/montaje/
```

La opción `-a` conserva los atributos que pueden preservarse durante la copia, como permisos, propietarios y enlaces.

Advertencia: esta operación no garantiza que se hayan conservado todos los metadatos específicos de Android, especialmente los contextos SELinux y ciertos atributos extendidos.

## 10. Comprobar el contenido

```bash
sudo du -sh ~/system_mod/montaje
df -h ~/system_mod/montaje
```

Comprobar que los archivos están presentes:

```bash
ls -lah ~/system_mod/montaje
```

## 11. Sincronizar y desmontar

```bash
sudo sync
sudo umount ~/system_mod/montaje
```

No continuar hasta que el desmontaje termine correctamente.

## 12. Verificar la imagen EXT4

```bash
file ~/system_mod/system_ext4.img
```

El resultado esperado debe identificar algo como esto:

```text
Linux rev 1.0 ext4 filesystem data
```

También se puede comprobar la integridad:

```bash
sudo e2fsck -fn ~/system_mod/system_ext4.img
```

La imagen debe estar desmontada antes de ejecutar esta comprobación.

## 13. Comprobar tamaños

```bash
ls -lh ~/system_mod/system_ext4.img
du -h ~/system_mod/system_ext4.img
du -sh ~/system_mod/extraido
```

Diferencias:

- `ls -lh`: tamaño lógico del archivo.
- `du -h`: espacio físico ocupado en disco.
- `du -sh extraido`: espacio ocupado por los archivos extraídos.

## 14. Consideraciones para Android y Fastboot

Antes de flashear la imagen, comprobar:

- Compatibilidad de EXT4 con la partición de destino.
- Tamaño máximo permitido para `SYSTEM`.
- Si el dispositivo utiliza particiones dinámicas.
- Si necesita reconstrucción de `super.img`.
- Propietarios y permisos originales.
- Contextos SELinux (`file_contexts`).
- Atributos extendidos y capacidades de archivos.
- Compatibilidad con AVB (Android Verified Boot).

Importante: no utilizar directamente `fastboot flash system system_ext4.img` sin verificar primero la arquitectura de particiones y el formato de imagen esperado por el dispositivo.

## 15. Estado del proyecto

| Elemento | Estado |
|---|---|
| Imagen EROFS original | Disponible |
| Extracción de EROFS | Completada |
| Creación de EXT4 | Completada |
| Copia de archivos | Completada |
| Desmontaje | Completado |
| Verificación EXT4 | Completada |
| Restauración de metadatos Android | Pendiente de verificar |
| Preparación para Fastboot | Pendiente |
| Flasheo | Pendiente |

## 16. Próximos pasos

1. Identificar la marca y el modelo del dispositivo.
2. Analizar la estructura de particiones.
3. Revisar los metadatos originales.
4. Recuperar los contextos SELinux.
5. Modificar los archivos de `SYSTEM`.
6. Reconstruir la imagen definitiva.
7. Verificar compatibilidad con Fastboot.

- Proyecto: Android EROFS to EXT4
- Sistema de archivos original: EROFS
- Sistema de archivos de destino: EXT4
- Herramientas: `erofs-utils` / `e2fsprogs`
- Objetivo: modificación y reconstrucción de una ROM Android.
