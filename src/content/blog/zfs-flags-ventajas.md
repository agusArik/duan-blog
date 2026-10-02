---
title: "ZFS: ventajas y los flags que usé en mi NAS"
description: "Por qué elegí ZFS en vez de mdadm + ext4, y una explicación en detalle de cada opción que usé al crear el pool y los datasets."
date: 2026-10-01
tags: ["zfs", "nixos", "storage", "linux", "self-hosting", "nas"]
---

En este artículo vamos a explorar algunas de las opciones más útiles que se suelen pasar a
la hora de crear un pool en ZFS.

## Por qué ZFS y no mdadm

mdadm arma el RAID a nivel de bloques crudos, sin saber nada sobre el filesystem que corre encima. ZFS hace todo en una sola capa: maneja volúmenes, redundancia (RAID) y filesystem juntos, y por eso conoce la estructura real de los datos.

Esa diferencia no es solo arquitectónica, trae ventajas concretas:

- **Checksums de todo**: cada bloque de datos tiene un checksum. En una lectura, si el checksum no coincide, ZFS lo detecta al instante.
- **Scrubs**: con `zpool scrub`, ZFS recorre todo el pool verificando checksums contra las copias redundantes (en mi caso, el mirror) y repara automáticamente cualquier corrupción silenciosa (bit rot) que encuentre. mdadm no puede hacer esto, porque no sabe qué bloques son datos válidos y cuáles no.
- **Snapshots baratos**: copy-on-write nativo, un snapshot es casi instantáneo y no duplica datos hasta que algo cambia.
- **Compresión transparente**: a nivel de filesystem, sin pasos extra.
- **Encriptación nativa**: sin necesitar LUKS por fuera.

Todo esto administrado con una sola herramienta (`zfs`/`zpool`), sin tener que coordinar mdadm + LVM + filesystem por separado.

## El comando de creación del pool

Como vimos en el [artículo anterior](/blog/nixos-nas-zfs), había utilizado este comando para crear el zpool que
utilizaría en el NAS.

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
    /dev/vdb \
    /dev/vdc
```

Una aclaración antes de entrar en cada flag: las opciones con `-o` (minúscula) son
propiedades del **pool** en sí (como `ashift`), mientras que las opciones con `-O`
(mayúscula) son propiedades del **dataset raíz** que se crea junto con el pool, y se
heredan por default a todos los datasets hijos a menos que los pises explícitamente. Para
más información recomiendo también la siguiente página para [configurar los zpools.](https://jrs-s.net/2018/08/17/zfs-tuning-cheat-sheet/)

### `ashift=12`

Define el tamaño de bloque que ZFS asume para el almacenamiento físico, como potencia de 2 (`2^12 = 4096` bytes). Los discos modernos usan sectores físicos de 4K, aunque muchos siguen reportando 512 bytes por compatibilidad (512e). Si ZFS asume 512 bytes y el disco en realidad usa 4K, cada escritura puede terminar pisando sectores parciales, con un golpe serio de rendimiento.

`ashift` se define **una sola vez, al crear el pool**, y no se puede cambiar después sin recrearlo. Por eso conviene setearlo explícito en vez de confiar en el auto-detect (que a veces falla, sobre todo en discos virtuales o detrás de adaptadores).

### `encryption=on`, `keyformat=passphrase`, `keylocation=prompt`

Encriptación nativa de ZFS, a nivel de dataset (no de disco completo como LUKS). Las tres opciones van juntas:

- `encryption=on` habilita la encriptación con el algoritmo default (AES-256-GCM).
- `keyformat=passphrase` dice que la clave se deriva de una passphrase que tenés que tipear, en vez de un archivo de clave binario.
- `keylocation=prompt` le dice a ZFS que pida la passphrase interactivamente (por terminal) en vez de leerla de un archivo.

La alternativa sería `keylocation=file:///ruta/a/key`, que permite desbloqueo automático en el boot sin intervención manual — útil para un NAS headless, pero significa que la clave vive en algún lado del sistema (normalmente en una partición o pendrive aparte). Con `prompt`, cada reinicio real pide la contraseña a mano. Es un trade-off entre comodidad y seguridad; para un NAS que no se reinicia todo el tiempo, prefiero tipear la passphrase.

Una ventaja de la encriptación nativa de ZFS sobre LUKS: es por dataset, no por disco entero. Esto significa que podés tener datasets encriptados y no encriptados en el mismo pool, con claves distintas por dataset si hace falta.

### `compression=lz4`

Compresión transparente, a nivel de bloque. LZ4 es el algoritmo default recomendado en casi todos los setups de ZFS hoy en día, por una razón simple: es extremadamente rápido (más rápido que la velocidad de muchos discos), así que casi nunca es un cuello de botella, y además detecta datos incompresibles y los deja pasar sin overhead.

Las alternativas más conocidas:

- **zstd**: mejor ratio de compresión que lz4, pero más costoso en CPU. Tiene sentido si el CPU sobra y el objetivo es maximizar espacio (por ejemplo, backups fríos).
- **gzip**: más lento que ambos, generalmente no se recomienda salvo casos puntuales.

Para un NAS de uso general, donde no quiero que la compresión le pegue a la latencia de lectura/escritura, lz4 es la opción razonable por default.

### `atime=off`

`atime` es el timestamp de "último acceso" que Linux actualiza cada vez que se **lee** un archivo (no solo cuando se escribe). Mantenerlo activo implica una escritura en disco por cada lectura, lo cual es puro overhead si no tenés un caso de uso que dependa de saber cuándo se leyó un archivo por última vez (algún gestor de cache muy específico, por ejemplo).

Lo seteé a nivel de pool para que todos los datasets lo hereden, en vez de desactivarlo dataset por dataset.

### `mountpoint=none`

Por default, cuando creas un pool, ZFS monta automáticamente el dataset raíz en `/<nombre-del-pool>`. Con `mountpoint=none`, el dataset raíz del pool queda sin punto de montaje — no se monta nada ahí.

Esto fuerza a que cada dataset hijo (en mi caso `root`, `home`, `nix`, `var`) tenga su propio mountpoint explícito, en vez de terminar todo colgando automáticamente de una carpeta que ZFS decidió por mí. Para los datasets del sistema, usé `mountpoint=legacy`, que es otra cosa distinta: significa que ZFS no gestiona el montaje en absoluto, y lo haces mano con `mount -t zfs` (o, en mi caso, queda declarado en la config de NixOS).

### `xattr=sa`

Los atributos extendidos (xattrs) son metadata adicional que se le puede asociar a un archivo, más allá de los permisos Unix tradicionales — las ACLs, por ejemplo, se implementan como xattrs.

ZFS tiene dos formas de guardarlos:

- `xattr=on` (default): los guarda como archivos ocultos dentro de un directorio oculto del dataset. Funciona, pero cada acceso a un xattr implica una operación de I/O extra para leer ese archivo oculto.
- `xattr=sa`: los guarda directamente en el inodo del archivo (System Attribute), como metadata inline. Mucho más rápido, especialmente cuando hay muchos atributos (como pasa con ACLs de Samba).

Para un NAS que va a compartir carpetas por red con permisos tipo Windows (Samba/NFS con ACLs), `xattr=sa` es la opción correcta casi siempre.

### `acltype=posixacl`

Habilita soporte de ACLs estilo POSIX (las mismas que entiende `setfacl`/`getfacl` en Linux). Sin esto, ZFS solo soporta los permisos Unix tradicionales (dueño/grupo/otros), que no alcanzan para compartir carpetas con permisos granulares por usuario — algo que casi siempre hace falta en un NAS con varios usuarios.

### `naspool` y `mirror`

`naspool` es simplemente el nombre que le di al pool (podría ser cualquier string). `mirror` seguido de los dos discos (`/dev/vdb`, `/dev/vdc`) le dice a ZFS que cree un vdev en mirror (RAID1): cada disco es una copia exacta del otro, tolera la pérdida de 1 disco sin perder datos.

## Las opciones a nivel de dataset

Después del pool, creé los datasets del sistema:

```bash
zfs create -o refreservation=1G -o mountpoint=none naspool/reserve

zfs create -o mountpoint=legacy naspool/root
zfs create -o mountpoint=legacy naspool/home
zfs create -o mountpoint=legacy naspool/nix
zfs create -o mountpoint=legacy naspool/var
```

`root`, `home`, `nix` y `var` no necesitan repetir `compression`, `atime`, `xattr` ni `acltype` — los heredan directamente del pool. Solo necesitan su propio `mountpoint=legacy`, porque eso sí es específico de cada dataset.

### `refreservation=1G` en `reserve`

Este dataset no guarda nada útil — es un truco clásico de ZFS. `refreservation=1G` reserva 1GB de espacio que **no puede ser usado por ningún otro dataset del pool**, ni siquiera si el pool se llena por completo.

¿Para qué sirve? Si un pool ZFS se llena al 100%, podés quedarte sin espacio ni siquiera para borrar archivos (algunas operaciones de ZFS necesitan espacio libre para funcionar, incluso un `rm`). Tener este colchón reservado garantiza que, en el peor caso, siempre haya margen para hacer limpieza y recuperar el pool, en vez de quedar en un estado en el que ni siquiera podés liberar espacio.

## Resumen

ZFS resuelve en una sola herramienta lo que normalmente requeriría mdadm + LVM + un filesystem aparte, y de paso suma checksums, scrubs, snapshots, compresión y encriptación nativas. Los flags que usé no son arbitrarios: cada uno resuelve un problema puntual (alineación física, seguridad, rendimiento, compatibilidad con ACLs de red, o protección contra quedarse sin espacio). Si estás por armar algo similar, la receta de arriba es un buen punto de partida — ajustando `compression` y `encryption` según tu caso de uso.
