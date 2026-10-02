---
title: 'Sacar /var/log de mis snapshots de btrfs'
description: 'Convertir /var/log y /var/cache en subvolúmenes propios para que snapper deje de arrastrar logs y cachés de paquetes en cada snapshot.'
date: 2026-04-18
tags: ['btrfs', 'linux', 'snapshots']
---

Snapper sobre un único subvolumen raíz funciona, pero implica que cada snapshot arrastra lo que sea que haya en `/var/log` y `/var/cache` en ese momento — y ninguno de los dos es algo a lo que realmente quiera poder volver.

## La solución: subvolúmenes separados

El layout en el que terminé trata al sistema de archivos raíz como un conjunto de subvolúmenes, no como un bloque único:

- `@` — raíz
- `@home` — home
- `@log` — `/var/log`
- `@pkg` — caché de paquetes

`@log` y `@pkg` quedan totalmente afuera de la política de snapshots, así que un rollback restaura el sistema sin también rebobinar los logs o forzar un repoblado de la caché de paquetes.

## Frecuencia de snapshots

También recorté snapper a snapshots solo diarios, sobreescribiendo el timer por defecto con una unit de systemd en vez de pelear directamente con la config del paquete. Los snapshots por hora eran básicamente ruido para un desktop de un solo usuario.

## De dónde salió esto

Todo este ejercicio arrancó porque quería una partición btrfs encriptada con LUKS2 con el mismo layout de subvolúmenes, probado primero en una VM antes de tocar el disco real — vale la pena hacerlo en ese orden.
