---
title: "Armando un NAS con NixOS y ZFS"
description: "Cómo instalé NixOS sobre un pool ZFS en mirror para armar un NAS casero, con NFS y Samba, y los problemas con los que me choqué en el camino."
date: 2026-09-26
tags: ["nixos", "zfs", "nas", "linux", "self-hosting", "home-lab"]
---

Hace poco me puse a armar un NAS casero, y decidí usar NixOS con ZFS como filesystem en vez de la típica combinación mdadm + ext4/btrfs. Esta entrada documenta cómo lo armé, paso a paso, incluyendo los problemas que tuve en el camino.

Por ahora todo esto lo hice en una máquina virtual, para probar el proceso completo antes de tocar hardware real. La idea es que cuando tenga los discos físicos, pueda reproducir absolutamente todo el sistema a través de los archivos de configuración de NixOS.

## Por qué ZFS

ZFS combina el manejo de volúmenes, RAID y filesystem en una sola capa, a diferencia de mdadm (que arma el RAID a nivel de bloques) más un filesystem tradicional encima. La ventaja concreta: ZFS conoce la estructura de los datos, así que puede detectar y reparar corrupción silenciosa (bit roto) con un `scrub`, algo que mdadm no puede hacer porque solo ve bloques crudos.

El RAID está permitido gracias a la unificación de distintas unidades físicas que se
pueden combinar en unidades lógicas (**vdevs**) como mirrors para RAID. Una vez
combinados, tenemos lo que ZFS llama como **pool**. Los pools contienen datasets (se verá
más adelante)

Además de todo esto, ZFS también presenta **Copy-on-Write** (Cow), como era de esperarse.
Por lo que vamos a tener transacciones atómicas que escriben el bloque que fue modificado
en espacio libre en disco, para luego hacer una referencia al bloque viejo que fue
modificado hasta escribir todos los cambios. Esto facilita las snapshots.

## Encriptación

Otra cosa es que ZFS tiene encriptado nativo, lo cual nos ahorra la necesidad de usar LUKS
y se integra muy bien a todo el pool de discos que tenemos. No solo eso, sino que podemos
ademas formar el set de discos agrupándolos en volúmenes lógicos como se haría con LVM.
Tres mundos distintos que chocan y se unen en uno solo. RAID, LVM, LUKS.

## Topología: mirror

Para el NAS elegí mirror en vez de RAIDZ1/RAIDZ2. Con 2 discos es la opción más simple de entender y administrar, y es lo que también usan los NAS domésticos de 2 bahías (Synology, QNAP). Si en algún momento paso a más discos en el hardware real, ahí evalúo si conviene RAIDZ.

Un punto importante que aprendí en el camino: **la topología no se puede cambiar in-place**. Se puede agregar un disco a un mirror existente (`zpool attach`) o convertir un disco simple en mirror, pero pasar de mirror a RAIDZ (o viceversa) requiere crear un pool nuevo y migrar los datos con `zfs send/receive`. Mejor definirla bien desde el principio.

## Particionado

Instalación normal de NixOS, sin flakes ni home-manager (para este proyecto en particular preferí mantenerlo simple).

Disco de sistema separado del pool de datos. Notar que el disco que estoy usando en la VM
es bastante pequeño, de unos 4 GB.

```bash
parted /dev/sda -- mklabel gpt
parted /dev/sda -- mkpart ESP fat32 1MiB 1GiB
parted /dev/sda -- set 1 esp on
parted /dev/sda -- mkpart primary 1GiB 100%

mkfs.fat -F 32 -n boot /dev/sda1
mkswap /dev/sda2
swapon /dev/sda2
```

La segunda partición del disco de sistema la usé para swap (partición dedicada, no swapfile).

Los discos del mirror (`sdb` y `sdc` en la VM, 10GB cada uno) van sin particionar, se le pasan enteros a ZFS.

## Creando el pool

```bash
zpool create \
    -o ashift=12 \
    -O encryption=on \
    -O keyformat=passphrase \
    -O keylocation=prompt \
    -O compression=lz4 \
    -O atime=off \
    -O mountpoint=none \
    -O xattr=sa \
    -O acltype=posixacl \
    naspool \
    mirror \
    /dev/sdb \
    /dev/sdc
```

Voy a explicar en detalle las opciones en este [artículo sobre ZFS.](/blog/zfs-flags-ventajas)

## Datasets

A diferencia de los sistemas de archivos tradicionales, como ext4, ZFS viene con una
característica importante llamada **dataset**. Un dataset me permite dividir el sistema de
archivos en pequeños bloques los cuales no dejan memoria prealocada, esto nos da la
versatilidad de poder disminuir o aumentar el espacio en ellos a medida que sea necesario.

```bash
zfs create -o refreservation=1G -o mountpoint=none naspool/reserve

zfs create -o mountpoint=legacy naspool/root
zfs create -o mountpoint=legacy naspool/home
zfs create -o mountpoint=legacy naspool/nix
zfs create -o mountpoint=legacy naspool/var
```

El dataset `reserve` con `refreservation=1G` es un colchón de espacio reservado: si el pool se llena, tener ese margen evita quedarse sin espacio para operaciones básicas de ZFS (como borrar cosas para liberar espacio).

Usé `mountpoint=legacy` en todos los datasets del sistema, así los monto yo a mano (y quedan declarados en la config de NixOS) en vez de que ZFS los gestione automáticamente. `compression`, `xattr` y `acltype` no hace falta repetirlos en cada dataset porque se heredan del pool.

## Montaje y generación de la config

```bash
mount -t zfs naspool/root /mnt
mkdir -p /mnt/{home,nix,var,boot}
mount -t zfs naspool/home /mnt/home
mount -t zfs naspool/nix /mnt/nix
mount -t zfs naspool/var /mnt/var
mount /dev/vda1 /mnt/boot

nixos-generate-config --root /mnt
```

## networking.hostId

Algo que casi me olvido: ZFS en NixOS requiere `networking.hostId`. Es un mecanismo anti-import-simultáneo: evita que dos máquinas monten el mismo pool a la vez (protección pensada para clusters/failover, pero obligatoria igual en un sistema single-node).

```bash
head -c4 /dev/urandom | od -A none -t x4
```

Y el resultado va en la config:

```nix
networking.hostId = "a1b2c3d4";
```

Importante: una vez instalado, no se cambia. Si lo cambias después, ZFS puede rechazar importar el pool en el siguiente boot porque "no reconoce" la máquina.

## El primer boot falló

Después de instalar y reiniciar, el sistema no arrancaba: se quedaba en emergency mode con esto en la consola:

```
[FAILED] Failed to start Import ZFS pool "naspool"...
[DEPEND] Dependency failed for /sysroot.
```

La solución fue decirle al boot loader que buscara los discos por una ruta estable en vez de los nombres de los discos.

```nix
boot.zfs.devNodes = "/dev/disk/by-path";
```

Para aplicar el fix tuve que volver a bootear desde el ISO del instalador, re-importar el pool manualmente (con `zpool import -N naspool` y `zfs load-key naspool`, porque al estar encriptado no monta nada sin cargar la clave primero), remontar todo, editar la config, y correr `nixos-install` de nuevo para regenerar el initrd. No hizo falta reinstalar desde cero, el fix se aplica sobre lo ya instalado.

## Compartiendo archivos con NFS y Samba

Para compartir archivos en red, configuré NFS y SMB siguiendo esta sección de la guía de
instalación de NixOS con ZFS.

https://nixos.wiki/wiki/ZFS#NFS_shares

Quizás no tan sorprendente viniendo de la distro que es puramente declarativa en casi
todo, habilitar los servicios es bastante sencillo. No basta más que una línea para
habilitar tanto NFS como SMB.

En `configuration.nix`:

```nix
services.nfs.server.enable = true;
networking.firewall.allowedTCPPorts = [ 2049 ];
networking.firewall.allowedUDPPorts = [ 2049 ];

services.samba = {
  enable = true;
  settings.global = {
    "usershare path" = "/var/lib/samba/usershares";
    "usershare max shares" = "100";
    "usershare allow guests" = "yes";
    "usershare owner only" = "no";
  };
  openFirewall = true;
};
```

Como bien dice en la guía, el path de samba está hardcodeado a **/var/lib/samba/usershares**.

Y para exportar el dataset `naspool/home`:

```bash
zfs set sharenfs="rw=192.168.1.0/24,no_root_squash" naspool/home

mkdir -p /var/lib/samba/usershares
chmod 1775 /var/lib/samba/usershares
zfs set sharesmb=on naspool/home
```

El `chmod 1775` incluye el sticky bit (el `1` adelante): en un directorio, hace que solo el dueño de un archivo (o root) pueda borrarlo o renombrarlo, aunque otros tengan permiso de escritura sobre el directorio. Es el mismo mecanismo que usa `/tmp`.

## Dejando la puerta abierta a GUI

No tengo pensado usar entorno gráfico en el NAS, pero por las dudas dejé `services.xserver.enable = true` y agregué `ly` como display manager, deshabilitado:

```nix
services.displayManager.ly.enable = false;
```

Mientras no estén el xserver y display manager habilitados, podré correr NixOS en lo que
es equivalente a un **runlevel 3** (server). Bastante conveniente para un NAS, el cual no
necesita nada más.

## Estado actual

Con esto, el NAS ya bootea correctamente, con el pool en mirror montado, encriptado, compartiendo `home` por NFS y Samba. Los próximos pasos van a ser pasar todo esto a hardware real y ajustar la topología ZFS según cuántos discos termine usando.
