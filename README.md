# Manual de generación de llaves SSH Ed25519 en GNU/Linux

Guía generalista aplicable a cualquier distribución de GNU/Linux (Debian, Ubuntu, Fedora, Arch, openSUSE, etc.) para crear un par de llaves SSH moderno, seguro y bien respaldado.

<p style="text-align: center"><img src="./assets/ssh-ed25519-01.png" style="width: 50%;" alt="SSH Ed25519 en Linux" /></p>

## a. Historial de cambios

- `[0.3]` – 2026-05-20
    - Revisión de seguridad: verificación de host key (MITM/TOFU), aviso en `ssh-keygen -R`, permisos del home del servidor, alternativa `ssh-copy-id` con `chmod` atómico, bloque `Host *` con `ForwardAgent no` / `AddKeysToAgent confirm` / `HashKnownHosts yes`, `StrictHostKeyChecking yes` en producción, QR a `/tmp` con `shred`, reparación de permisos tras restauración GPG, verificación de `sshd_config.d/`, advertencias de scrollback y `-vvv`
- `[0.2]` – 2026-04-20
    - Mejoras de seguridad expresado en notas, inlcusión de nuevas buenas prácticas, y uso de licencia CC
- `[0.1]` – 2026-04-20
    - Estructura inicial

## b. To-do

- Llaves respaldadas por hardware (YubiKey, Nitrokey, TPM), uso de `-t ed25519-sk`
- Gestión de la passphrase de una llave ya creada
- Certificados SSH (firmado centralizado de llaves)
- Firmado genérico de archivos con `ssh-keygen -Y`

---

## 1. Introducción

SSH (Secure Shell) utiliza criptografía de llave pública para autenticar usuarios sin necesidad de transmitir passphrases por la red. Este manual te guía para generar un par de llaves Ed25519, el algoritmo recomendado hoy en día por ser rápido, compacto y resistente.

El resultado serán dos archivos dentro de `~/.ssh/`:

- Llave privada — se queda en tu máquina. **Nunca se comparte**.
- Llave pública (`.pub`) — se copia al servidor remoto o se pega en servicios como GitHub, GitLab, etc.

En este manual usamos *passphrase* (frase de paso) en lugar de "contraseña" para enfatizar que debe ser una frase larga, no una palabra corta.

## 2. Requisitos previos

Verifica que OpenSSH esté instalado:

```bash
ssh -V
```

Deberías ver algo como `OpenSSH_9.x`. Si el comando no existe, instálalo:

| Distribución        | Comando de instalación                         |
|---------------------|------------------------------------------------|
| Debian / Ubuntu     | `sudo apt install openssh-client`              |
| Fedora / RHEL       | `sudo dnf install openssh-clients`             |
| Arch / Manjaro      | `sudo pacman -S openssh`                       |
| openSUSE            | `sudo zypper install openssh-clients`          |
| Alpine              | `sudo apk add openssh-client`                  |

Asegúrate también de que exista el directorio `~/.ssh` con los permisos correctos:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
```

## 3. Comparativa de algoritmos de llave SSH

Antes de generar la llave conviene entender por qué Ed25519 es la opción preferida:

| Algoritmo    | Tamaño típico    | Velocidad  | Seguridad actual                          | Compatibilidad            | Recomendación                            |
|--------------|------------------|------------|-------------------------------------------|---------------------------|------------------------------------------|
| **Ed25519**  | 256 bits         | Muy alta   | Excelente (curva Edwards moderna)         | OpenSSH ≥ 6.5 (2014+)     | ✅ **Opción por defecto hoy**            |
| RSA 4096     | 4096 bits        | Media-baja | Fuerte si ≥ 3072 bits                     | Universal (legado)        | ⚠️ Solo si el servidor es muy viejo      |
| RSA 2048     | 2048 bits        | Media      | Aceptable pero envejecida                 | Universal                 | ❌ Ya no se recomienda                   |
| ECDSA        | 256/384/521      | Alta       | Sólida, pero depende del RNG              | Amplia                    | ⚠️ Ed25519 es preferible                 |
| DSA          | 1024 bits        | Alta       | **Rota** — desactivada en OpenSSH         | Obsoleta                  | ❌ No usar nunca                         |
| Ed25519-sk   | 256 bits + token | Muy alta   | Excelente (llave nunca sale del hardware) | OpenSSH ≥ 8.2 (2020+)     | ✅ Máxima seguridad con YubiKey/Nitrokey |

Usa Ed25519 salvo que necesites interoperar con sistemas muy antiguos que solo soporten RSA.

## 4. Generación de la llave Ed25519

En lugar del nombre predeterminado `id_ed25519`, elige un nombre descriptivo, especialmente si usarás varias llaves:

```bash
~/.ssh/github_personal
~/.ssh/gitlab_trabajo
~/.ssh/servidor_produccion
~/.ssh/router_pruebas
```

Es una buena práctica tener una llave por propósito o por servicio. Si una se compromete, el daño queda acotado.

Una vez decidido, declara las dos variables según tu personalización

```bash
export MINOMBREDELLAVE=villonaco
export MICORREO=usuario@dominio.com
```

Reemplaza los valores que están a la derecha del igual "=". Estas variables se usan en el resto del manual. Ejecútalas una sola vez en tu terminal y no cierres esa ventana hasta terminar. Si abres una nueva o cierras sesión, vuelve a declararlas.

### 4.1 Comando recomendado

```bash
ssh-keygen -t ed25519 -a 100 -C "${MICORREO}" -f ~/.ssh/${MINOMBREDELLAVE}
```

### 4.2 Desglose de cada opción

| Opción                         | Qué hace                                                                                                              |
|--------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| `-t ed25519`                   | Selecciona el algoritmo Ed25519.                                                                                      |
| `-a 100`                       | Aplica 100 rondas de KDF (bcrypt) sobre la passphrase. Hace mucho más costoso un ataque de fuerza bruta offline.      |
| `-C "${MICORREO}"`             | Comentario que queda incrustado en la llave pública. Útil para identificarla (usa correo, host, o `usuario@máquina`). |
| `-f ~/.ssh/${MINOMBREDELLAVE}` | Ruta y nombre personalizado del archivo. Evita sobrescribir `id_ed25519` si ya existe otra llave.                     |

> 💡 **Sobre `-a 100`:** el valor por defecto de `ssh-keygen` es 16 rondas. Subirlo a 100 endurece la derivación de clave a partir de tu passphrase con un coste imperceptible al desbloquear (unos pocos cientos de milisegundos) pero muy alto para un atacante que intente millones de passphrases.

### 4.3 Flujo interactivo esperado

```console
Generating public/private ed25519 key pair.
Enter passphrase (empty for no passphrase): ********
Enter same passphrase again: ********
Your identification has been saved in /home/usuario/.ssh/villonaco
Your public key has been saved in /home/usuario/.ssh/villonaco.pub
The key fingerprint is:
SHA256:QKxafE5eAEQ/2bP+0kHtvBCXMKFsFtfAuuTIBB98dAA usuario@dominio.com
The key's randomart image is:
+--[ED25519 256]--+
|   o++Eo+o=+     |
|    .o=+.=+ .    |
|   . +=+B. + .   |
|    + ==+oo +    |
|   o B =So =     |
|  .   =.o o o    |
|        .. o .   |
|        ... .    |
|         ..      |
+----[SHA256]-----+
```

OpenSSH usa por defecto el formato moderno `OPENSSH PRIVATE KEY`. Para compatibilidad con herramientas antiguas puede exportarse a PEM con -m PEM.

### 4.4 Verificación tras la generación

```bash
# Ver la fingerprint de la llave recién generada
ssh-keygen -lf ~/.ssh/${MINOMBREDELLAVE}.pub

# Ver la llave pública completa (la que pegarás en GitHub, servidores, etc.)
cat ~/.ssh/${MINOMBREDELLAVE}.pub

# Confirmar que la privada está cifrada (debe pedir passphrase)
ssh-keygen -y -f ~/.ssh/${MINOMBREDELLAVE} > /dev/null
```

## 5. Sobre la passphrase

Usa siempre una passphrase. Una llave privada sin passphrase equivale a una copia de tus credenciales en texto plano: si alguien obtiene el archivo, obtiene tu acceso.

Recomendaciones:

- Mínimo 16 caracteres, idealmente una frase larga de 4-6 palabras aleatorias (estilo *diceware*).
- No reutilices la passphrase de otra cuenta.
- Combínala con `-a 100` para que resistir fuerza bruta sea inviable.

### Guardar la passphrase en un gestor de contraseñas

Guarda la passphrase en un gestor reputado. Opciones multiplataforma y libres:

- **KeePassXC** — local, base de datos cifrada, excelente para Linux.
- **Bitwarden** — nube cifrada de extremo a extremo con cliente local.
- **Pass** (`password-store`) — basado en GPG, integrable con scripts.
- **1Password / Dashlane** — comerciales, con buenas apps de escritorio.

> 🔐 **Ventaja adicional:** muchos gestores permiten también adjuntar archivos. Puedes almacenar ahí una copia cifrada de tu llave privada como respaldo de emergencia (ver sección 8).

## 6. Permisos correctos

SSH rechaza llaves con permisos demasiado abiertos. Ajústalos siempre:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/${MINOMBREDELLAVE}
chmod 644 ~/.ssh/${MINOMBREDELLAVE}.pub
```

Verificación:

```bash
ls -l ~/.ssh/
```

Debes ver `-rw-------` (600) para la privada y `-rw-r--r--` (644) para la pública.

Y también asegurar los permisos del lado del servidor:

```bash
chmod 750 ~                        # o 700 si no necesitas acceso de grupo
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

> ⚠️ SSH rechaza `authorized_keys` si el directorio home es escribible por el grupo (permisos 775, comunes en algunas distros). Asegúrate de que `~` tenga como máximo 750.

## 7. Uso de la llave

### 7.1 Cargar la llave en `ssh-agent`

Para no escribir la passphrase en cada conexión:

```bash
# En la mayoría de entornos de escritorio modernos ya hay un ssh-agent
# corriendo gestionado por systemd o el escritorio. Prueba primero:
ssh-add -t 1h ~/.ssh/${MINOMBREDELLAVE}

# Solo si obtienes "Could not open a connection to your authentication agent":
eval "$(ssh-agent -s)"
ssh-add -t 1h ~/.ssh/${MINOMBREDELLAVE}
```

El TTL de `-t 1h` reduce riesgo si se deja la sesión abierta y evita persistencia indefinida.

### 7.2 Copiar la llave pública a un servidor

> **Verifica el host key antes de la primera conexión.** SSH acepta automáticamente la primera clave que recibe (TOFU — Trust On First Use), lo que abre la puerta a ataques MITM. Obtén la fingerprint del servidor por un canal fuera de banda (consola IPMI, panel del proveedor VPS, acceso físico) y compárala antes de conectar:
>
> ```bash
> # En el servidor (consola local o panel del proveedor):
> ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
>
> # En tu máquina, para ver la fingerprint que SSH ha aceptado:
> ssh-keygen -lf ~/.ssh/known_hosts
> ```
>
> Solo si ambas coinciden puedes confiar en la conexión.

```bash
ssh-copy-id -i ~/.ssh/${MINOMBREDELLAVE}.pub usuario@servidor
```

En caso que no haya `ssh-copy-id` se puede usar:

```bash
cat ~/.ssh/${MINOMBREDELLAVE}.pub | ssh usuario@servidor \
  "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

> ⚠️ No uses la variante sin `chmod`: `mkdir -p ~/.ssh` crea el directorio con el umask del servidor (frecuentemente 755, legible por todos). Fijar los permisos en el mismo comando evita la ventana de exposición.

O manualmente, añadiendo el contenido de `${MINOMBREDELLAVE}.pub` a `~/.ssh/authorized_keys` del servidor.

### 7.3 Conectar usando una llave específica

```bash
ssh -i ~/.ssh/${MINOMBREDELLAVE} usuario@servidor
```

### 7.4 Configurar `~/.ssh/config` para varios hosts

Si manejas varias llaves, este archivo te ahorra problemas:

```ssh-config
# Defaults seguros para todos los hosts
Host *
    ForwardAgent no          # Nunca reenvíes el agente salvo que sea imprescindible
    AddKeysToAgent confirm   # Pide confirmación antes de cada uso de la llave
    HashKnownHosts yes       # Hashea los hostnames en known_hosts

Host github
    HostName github.com
    User git
    IdentityFile ~/.ssh/github_personal
    IdentitiesOnly yes

Host miservidor
    HostName 192.168.1.50
    User usuario
    IdentityFile ~/.ssh/servidor_produccion
    IdentitiesOnly yes
    StrictHostKeyChecking yes   # Rechaza cualquier cambio de host key sin intervención manual
```

Permisos del archivo: `chmod 600 ~/.ssh/config`.

> 💡 `ForwardAgent no` como valor por defecto evita que un `ssh -A` accidental exponga tu agente a un servidor comprometido. Actívalo explícitamente solo en hosts de confianza donde sea necesario (`ForwardAgent yes` en su bloque `Host`).
>
> `AddKeysToAgent confirm` hace que el agente pida confirmación en cada uso de la llave, lo que limita el daño si dejas la sesión desbloqueada.
>
> `StrictHostKeyChecking yes` en hosts de producción impide que SSH acepte automáticamente un cambio de host key, forzando intervención manual y protegiendo contra MITM.

### 7.5 Reinstalación o migración del servidor

> ⚠️ **Antes de eliminar la clave conocida, confirma que el cambio fue intencionado.** El mensaje `REMOTE HOST IDENTIFICATION HAS CHANGED` puede ser una señal de ataque MITM activo, no solo una reinstalación. Verifica con el administrador del servidor (por teléfono, chat interno u otro canal fuera de banda) que el cambio de host key era esperado. Solo entonces ejecuta:

```bash
# Cuando aparece "REMOTE HOST IDENTIFICATION HAS CHANGED":
ssh-keygen -R servidor.ejemplo.com
ssh-keygen -R 192.168.1.50
```

Después de reconectar, compara la nueva fingerprint con la que el servidor reporta por consola para confirmar que no hay impostor.

## 8. Opciones de respaldo de la llave

Si pierdes la llave privada, pierdes el acceso. Un buen respaldo es obligatorio.

### 8.1 Respaldo en un gestor de contraseñas

La opción más sencilla y segura para usuarios no avanzados:

1. Abre KeePassXC / Bitwarden / el gestor que uses.
2. Crea una entrada llamada, por ejemplo, *"Llave SSH – mi_llave_personal"*.
3. Guarda la passphrase como contraseña de la entrada.
4. Adjunta los archivos `${MINOMBREDELLAVE}` y `${MINOMBREDELLAVE}.pub` a esa entrada.
5. La base de datos del gestor ya está cifrada; asegúrate de tener su propia copia de seguridad sincronizada en nube o disco externo.

### 8.2 Respaldo cifrado en disco externo

Para quienes prefieren control local:

```bash
tar -czf - ~/.ssh/${MINOMBREDELLAVE} ~/.ssh/${MINOMBREDELLAVE}.pub \
  | gpg --symmetric --cipher-algo AES256 \
  -o /mnt/respaldo/${MINOMBREDELLAVE}.tar.gz.gpg
```

Para restaurar:

```bash
gpg -d /mnt/respaldo/${MINOMBREDELLAVE}.tar.gz.gpg | tar -xzf - -C ~
# Reparar permisos tras la restauración (tar puede extraer con el umask actual):
chmod 600 ~/.ssh/${MINOMBREDELLAVE}
chmod 644 ~/.ssh/${MINOMBREDELLAVE}.pub
chmod 700 ~/.ssh
```

Guarda el disco externo en un lugar físicamente seguro (cajón con llave, caja fuerte, casa de un familiar).

### 8.3 Respaldo impreso (*paper backup*)

Para la paranoia máxima y desastres catastróficos:

```bash
cat ~/.ssh/${MINOMBREDELLAVE}
```

Imprime el contenido (es texto ASCII empezando por `-----BEGIN OPENSSH PRIVATE KEY-----`) y guárdalo en un sobre sellado en un lugar físico seguro. Como la llave está cifrada por tu passphrase, el papel por sí solo no es suficiente para usarla.

> ⚠️ **Limpia el scrollback del terminal** tras mostrar la llave. `Ctrl+L` solo desplaza la vista; usa la opción específica de tu emulador (en GNOME Terminal: *Editar → Limpiar el registro*; en Konsole: *Editar → Limpiar historial*; en Alacritty/kitty: `Ctrl+Shift+K`). La llave puede quedar visible si alguien accede al buffer del terminal o a logs de sesión.

Puedes también convertirla a QR para facilitar el escaneo futuro:

```bash
# Instala qrencode según tu distro:
#   Debian/Ubuntu: sudo apt install qrencode
#   Fedora/RHEL:   sudo dnf install qrencode
#   Arch:          sudo pacman -S qrencode
#   openSUSE:      sudo zypper install qrencode

# Genera el QR en /tmp (no se sincroniza a la nube) con nombre único por llave:
qrencode -8 -r ~/.ssh/${MINOMBREDELLAVE} -o /tmp/${MINOMBREDELLAVE}_qr.png -s 4

# Imprime y borra de forma segura inmediatamente después:
shred -u /tmp/${MINOMBREDELLAVE}_qr.png
```

> ⚠️ No guardes el QR en el directorio actual (`$PWD`): puede estar sincronizándose con Dropbox, Nextcloud u otro servicio de nube sin que lo notes.

Para llaves Ed25519 el QR resultante es perfectamente escaneable. Para RSA 4096 puede ser necesario dividir la llave en dos QR o usar `paperkey/qrencode --8bit` con cuidado.

### 8.4 Qué **NO** hacer con el respaldo

- ❌ No la subas a Google Drive, Dropbox o iCloud sin cifrar.
- ❌ No la envíes por correo ni por WhatsApp.
- ❌ No la guardes en un repositorio git (público o privado).
- ❌ No la dejes en un USB sin cifrar olvidado en un cajón.

## 9. Buenas prácticas adicionales

- Rota tus llaves cada 1-2 años o inmediatamente si sospechas un compromiso.
- Revoca llaves del `authorized_keys` de los servidores cuando dejes de usar una máquina.
- Desactiva el login por contraseña en los servidores una vez validado el acceso por llave (`PasswordAuthentication no` en `/etc/ssh/sshd_config`). En Debian 12+, Ubuntu 22.04+ y Fedora, la configuración efectiva puede provenir de un archivo en `/etc/ssh/sshd_config.d/` que silenciosamente sobreescribe la directiva principal. Verifica el valor que realmente está activo:

  ```bash
  sshd -T | grep passwordauthentication
  ```

- Considera hardware tokens (YubiKey, Nitrokey) con llaves `ed25519-sk` para máxima seguridad: la llave nunca abandona el dispositivo físico.
- Registra la fingerprint de cada llave que generes (`ssh-keygen -lf ~/.ssh/${MINOMBREDELLAVE}.pub`) en tu gestor de passphrases para verificarla después.
- Para diagnosticar problemas puedes usar `ssh -vvv usuario@servidor`. La salida incluye rutas de archivos, versiones exactas de OpenSSH y detalles de negociación: no la pegues en tickets públicos ni logs compartidos.
- Evita usar `ssh -A` salvo que sea estrictamente necesario. `-A` permite que el servidor remoto utilice tu `ssh-agent` para autenticarse en otros sistemas en tu nombre. Si el servidor está comprometido, un atacante puede usar tu identidad para acceder a otros recursos. Alternativa recomendada: `ssh -J` (ProxyJump) que no expone tu agente. Para evitar activarlo accidentalmente, añade `ForwardAgent no` en el bloque `Host *` de `~/.ssh/config` (ver §7.4).
- El comentario `-C "${MICORREO}"` queda visible en el `authorized_keys` de cada servidor al que te conectas. Si prefieres no exponer tu correo a los sysadmins, usa `usuario@máquina` como comentario alternativo:

  ```bash
  -C "$(whoami)@$(hostname -s)"
  ```

## 10. Referencias

- `man ssh-keygen`
- [OpenSSH Release Notes](https://www.openssh.com/releasenotes.html)
- Daniel J. Bernstein et al., *High-speed high-security signatures* (Ed25519).
- Mozilla — [OpenSSH guidelines](https://infosec.mozilla.org/guidelines/openssh).

---

[Manual de generación de llaves SSH Ed25519 en GNU/Linux](https://github.com/noggalito/llave-ssh) © 2026 by [Calú](https://github.com/calu777) in [noggalito](https://noggalito.com/) is licensed under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)<img src="https://mirrors.creativecommons.org/presskit/icons/cc.svg" alt="" style="max-width: 1em;max-height:1em;margin-left: .2em;"><img src="https://mirrors.creativecommons.org/presskit/icons/by.svg" alt="" style="max-width: 1em;max-height:1em;margin-left: .2em;"><img src="https://mirrors.creativecommons.org/presskit/icons/sa.svg" alt="" style="max-width: 1em;max-height:1em;margin-left: .2em;">
