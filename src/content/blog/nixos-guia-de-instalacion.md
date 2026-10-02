---
title: "Instalación de NixOS encriptado con LUKS"
description: "Instalación segura de NixOS en un disco encriptado con LUKS y sistema de archivos BTRFS. Un esquema muy viable para escritorios de uso personal."
date: 2026-08-01
tags: ["nixos", "linux", "luks", "btrfs"]
---

NixOS es una distro de Linux que aprovecha al máximo el lenguaje puramente funcional **Nix**[^1] para que todas
las operaciones del sistema sean declarativas y atómicas, con posibilidad de rollback ante
cualquier pérdida.

De por sí, lo que habilita Nix y el gestor de paquetes del mismo nombre es crear **derivaciones** las cuales describen exactamente las dependencias, versiones y arquitectura para la cual se quiere instalar un programa. Esto garantiza reproducibilidad inmediata en cualquier máquina. Es como docker, sin la virtualización.

En esta distro, todo el fichero / (root) es de sólo lectura y se configura
declarativamente a través de un solo archivo (.nix) o varios módulos que se pueden
separar.

La idea de este artículo es enseñar a hacer una instalación básica de NixOS con un sistema
de archivos btrfs encriptado con LUKS para mayor seguridad. Sí recomiendo experiencia
previa con el lenguaje Nix o por lo menos haber leído su documentación antes de comenzar.

## Preparación previa: Algunas cuestiones

Esta guía es para la instalación de una ISO mínima. La ISO se puede conseguir para la
arquitectura 64 bit Intel/AMD con el [siguiente enlace](https://channels.nixos.org/nixos-26.05/latest-nixos-minimal-x86_64-linux.iso). Suponiendo que ya tengas la ISO grabada en un USB mediante cualquier método (dd, Rufus, Ventoy), solo falta conectarlo y bootear al USB mediante la tecla asociada a tu BIOS.

Al principio te loggearás como el usuario nixos, con contraseña vacia. Esto es bueno para
usar sudo sin contraseña. Usalo para cambiar la distribución del teclado al español con
loadkeys es. Si el texto es demasiado pequeño en la TTY, utiliza **setfont -d** para
incrementar el tamaño de la fuente.

### Conexión a una red

Es necesario tener una conexión a internet para poder realizar la instalación de NixOS. Lo
recomendable es configurar el servidor DHCP para no tener que asignar una ip manual. Si no
usas conexión por ethernet, es viable usar _NetworkManager_ con el comando **nmtui**.

## Particionado

El particionado en la instalación mínima es manual. Voy a dar de ejemplo el esquema que
uso yo en mi disco, mostrando cómo tengo configurando las snapshots, swap, y subvolumenes.
El particionado es para un sistema UEFI con tabla GPT. [^2]

1. Crear una tabla de particiones GPT (tener en cuenta que el disco se puede llamar distinto a sda)

```sh
parted /dev/sda -- mklabel gpt
```

2. Primero creamos la partición boot usando unos 512 MB en el primer sector del disco

```sh
parted /dev/sda -- mkpart ESP fat32 0% 512MB
parted /dev/sda -- set 1 esp on
```

3.  Creamos una partición swap (opcional, también se puede usar el método de zram)

```sh
parted /dev/sda -- mkpart swap linux-swap 512MB 2GB
```

Nota: El tamaño de la partición varia acorde a la RAM. Normalmente está bien con que sea
la mitad de la RAM.

Nota: Debería notarse que el SWAP va a estar desencriptado en este caso. Esto solo es
comprometedor si se planea realizar hibernación porque el contenido de RAM estará en texto
plano.

4.  Luego creamos otra partición donde tendremos btrfs + LUKS, con el tamaño restante del disco

```sh
parted /dev/sda -- mkpart root btrfs 2GB 100%
```

Nota: Si no querés swap, simplemente debes marcar 512MB 100% al final

Al final, deberíamos tener una partición como la siguiente:

```sh
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda    253:0    0   20G  0 disk
├─sda1 253:1    0  487M  0 part
├─sda2 253:2    0  1.5G  0 part
└─sda3 253:3    0   18G  0 part
```

## Formateo

Formateamos con los siguientes comandos:

1. Partición de boot

```sh
mkfs.fat -F 32 -n boot /dev/sda1
```

2. Partición swap

```sh
mkswap -L swap /dev/sda2
```

3. Para particionar root, en realidad vamos a usar LUKS y subvolumenes btrfs para tener root y home separados

```sh
# La partición encriptada
cryptsetup luksFormat -y --label="Encriptado" /dev/sda3
# abrimos la partición
cryptsetup luksOpen /dev/sda3 cryptroot
# Formateamos la partición creada con btrfs
mkfs.btrfs -L "root-encriptado" /dev/mapper/cryptroot
# Montar temporalmente el volumen raíz
mount /dev/mapper/cryptroot /mnt
# Crear subvolúmenes
# Esta es la estructura recomendada para tener todo ordenado y distintos snapshots
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@.snapshots
# Desmontar
umount /mnt
# Montar para nixos-install
mount -o subvol=@,compress=zstd:1,noatime /dev/mapper/cryptroot /mnt
mount -m -o subvol=@home,compress=zstd:1,noatime /dev/mapper/cryptroot /mnt/home
mount -m -o subvol=@log,compress=zstd:1,noatime  /dev/mapper/cryptroot /mnt/.snapshots
# Montar la partición EFI y swap
mount -m /dev/sda1 /mnt/boot
swapon /dev/sda2
```

Ya vamos a configurar las snapshots más adelante usando snapper.

Al final debería quedarnos algo similar:

```sh
NAME          MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda           253:0    0   20G  0 disk
├─sda1        253:1    0  487M  0 part  /mnt/boot
├─sda2        253:2    0  1.5G  0 part  [SWAP]
└─sda3        253:3    0   18G  0 part
  └─cryptroot 254:0    0   18G  0 crypt /mnt/.snapshots
                                        /mnt/home
                                        /mnt
```

Si todo está bien, corremos el comando

```sh
nixos-generate-config --root /mnt
```

Continuamos en el siguiente post para terminar con la instalación del sistema.

## Configuración del sistema

Se van a generar dos archivos: /mnt/etc/nixos/hardware-configuration.nix y
/mnt/etc/nixos/configuration.nix. No deberías editar el primero ya que contiene toda la
información sobre el hardware, módulos de kernel y particionado que acabamos de realizar
en nuestro esquema de particionado.

Luego, deberías editar configuration.nix con tu editor de preferencia

```sh
vim /mnt/etc/nixos/configuration.nix
```

Normalmente esta parte queda a disposición del usuario porque cada quien tendrá requisitos
y demandas distintas para una instalación de NixOS. Puedo dejar programas[^3] y algunos
dotfiles básicos para inspirarse a la hora de realizar la instalación[^4]

La referencia para todos los programas que se pueden declarar en configuration.nix está en
el [siguiente enlace](https://search.nixos.org/packages). Declarar un programa, por
ejemplo, hyprland, debería verse así:

```nix
  environment.systemPackages = with.pkgs; [
      wget
      git
  ];
```

También podemos activar hyprland u otros programas directamente y es recomendable activarlo dentro de configuration.nix (a nivel de sistema, y no de usuario). Esto nos ahorra la necesidad de declararlo en pkgs.

```nix
  programs.hyprland = {
      enable = true;
      xwayland.enable = true;
  };

  programs.firefox.enable = true;

  services.displayManager.ly.enable
```

Para configurar más opciones del programa, podemos [entrar aquí](https://search.nixos.org/options?channel=26.05). Esto nos será especialmente útil más adelante con home manager.

### SWAP en ZRAM[^5]

El swap en zram es un módulo de la kernel que se puede activar y es mucho más eficiente
que el swap tradicional porque a diferencia de este último, este swap es un sector
comprimido de la RAM.

Con esto también nos ahorramos tener que encriptar el swap, que de hecho, no lo hicimos.

Lo único malo de esto es que no podremos realizar hibernación utilizando zram. Pero la
mayoría de las veces no se necesita a no ser que uses un server.

En configuration.nix:

```nix
zramSwap = {
  enable = true;
  algorithm = "zstd";
  memoryPercent = 50;
};
```

## Flakes y Home Manager

Los **flakes** son una característica reciente de Nix, reproducible y similar a las derivaciones que nos servirán para instalar distintos paquetes de una manera más consistente.[^6]

Suponiendo que ya tengas las opciones del sistema, locales, servicios esenciales y
programas básicos declarados en configuration.nix, ahora podemos pasar a desbloquear los
flakes para poder instalar home-manager.

Agrega esta línea para habilitar los flakes.

```nix
nix.settings.experimental-features = [ "nix-command" "flakes" ];
```

Ahora, vamos al directorio /mnt/etc/nixos/ y creamos el archivo flakes.nix. Aquí una configuración básica de flakes con home manager para la versión 26.05. Tomate tu tiempo para leerla y digerirla.

```nix
{
  description = "Configuración NixOS con Home Manager y Hyprland";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-26.05";
    home-manager = {
      url = "github:nix-community/home-manager/release-26.05";
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };

  outputs = { self, nixpkgs, home-manager, ... }: {
    nixosConfigurations.miequipo = nixpkgs.lib.nixosSystem {
      system = "x86_64-linux";
      modules = [
        ./configuration.nix
        home-manager.nixosModules.home-manager
        {
            home-manager = {
                useGlobalPkgs = true;
                useUserPackages = true;
                users.usuario = import ./home.nix;
                backupFileExtension = "backup";
            };
        };
      ];
    };
  };
}
```

Los inputs son los programas recibidos que se encuentran en el repositorio nixpkgs. Cambiar miequipo por el hostname real y usuario por el usuario que se creó en la instalación, es decir, en configuration.nix.

### Configurar algunos programas (opcional)

Ahora, en el mismo directorio que los flakes y el resto de configuración, creamos home.nix. Aquí una configuración mínima de home manager que instalará rofi, neovim, unas fuentes y hyprland. Recordar cambiar usuario por tu nombre de usuario.

```nix
{ config, pkgs, ... }:

home = {
  username = "usuario";
  homeDirectory = "/home/usuario";
  stateVersion = "26.05";

  packages = with pkgs; [
    rofi
    neovim
    nerd-fonts.fira-code
    hyprland
  ];

};

programs.home-manager.enable = true;

```

Finalmente, después de haber configurado home manager. Podemos proceder con la
instalación haciendo. No debería haber ningún error.

```sh
nixos-install --flake /mnt/etc/nixos#miequipo
```

Al final, nixos-install preguntará por tu contraseña root. Luego de ingresarla,
ingresarás la contraseña de tu usuario antes de hacer un reinicio.

```sh
nixos-enter --root /mnt -c 'passwd usuario'
```

Hacemos reboot y cuando entremos a la nueva instalación, podríamos seguir uno de los
siguientes esquemas, copiando todo desde /etc/nixos/ hacia home.

Una estructura recomendable para organizar el contenido post-instalación es la siguiente,
por ejemplo:

```sh

~/nixos-config/
├── flake.nix
├── flake.lock
├── hosts/
│   └── miequipo/
│       ├── configuration.nix
│       └── hardware-configuration.nix
└── modules/
    └── home-manager/
        └── programs/
            └── hyprland/
            │   └── default.nix
            └── zsh/
            │   └── default.nix
            └── foot/
                └── default.nix
```

Si te interesa organizar los dotfiles aún más, siguiendo una mejor estructura, deberías
ver estos siguientes[^7] repositorios:

- [https://github.com/etu/nixconfig
  ](https://github.com/etu/nixconfig)
- [https://github.com/matthewbauer/nixiosk
  ](https://github.com/matthewbauer/nixiosk)

## Artículos interesantes

Ver [^8], [^9], [^10]

## Bibliografía

[^1]:
    Nix Manual. "Introduction".
    https://nix.dev/manual/nix/2.28/introduction

[^2]:
    NixOS Wiki. "Full Disk Encryption with Swap Volume and Unencrypted Boot".
    https://nixos.wiki/wiki/Full_Disk_Encryption#Set_Up_Full_Disk_Encryption_with_Swap_Volume_and_Unencrypted_Boot

[^3]:
    NixOS Wiki. "Category:Desktop".
    https://wiki.nixos.org/wiki/Category:Desktop

[^4]:
    Andrey0189. "nixos-config" (configuración de referencia).
    https://github.com/Andrey0189/nixos-config

[^5]:
    NixOS Wiki. "Swap".
    https://nixos.wiki/wiki/Swap

[^6]:
    Flakes en Nixos
    https://nixos.wiki/wiki/flakes

[^7]:
    Templates de dotfiles en nixos para empezar a organizar
    https://github.com/Misterio77/nix-starter-configs

[^8]:
    NixOS. "Nix Pills: Our First Derivation".
    https://nixos.org/guides/nix-pills/06-our-first-derivation

[^9]:
    Zero to Nix.
    https://zero-to-nix.com/

[^10]:
    Home Manager Options.
    https://home-manager-options.extranix.com/
