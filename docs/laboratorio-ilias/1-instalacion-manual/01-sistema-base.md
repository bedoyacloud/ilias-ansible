# 1. Sistema base

Objetivo: un servidor actualizado, con swap y con un firewall propio, antes de instalar nada de ILIAS.
Todo se ejecuta dentro del servidor (`ssh -i ~/.ssh/mi-llave.pem ubuntu@IP_DEL_SERVIDOR`).

## 1.1 Actualizar el sistema

```bash
sudo apt update
sudo apt full-upgrade -y
```

- `update` descarga la lista de paquetes; `full-upgrade` instala las versiones nuevas.
- Si aparece una pantalla preguntando qué servicios reiniciar, deja lo marcado y pulsa Enter.

¿Pide reinicio? (por ejemplo, si se actualizó el kernel):

```bash
[ -f /var/run/reboot-required ] && echo "REINICIAR" || echo "no hace falta"
```

Si dice `REINICIAR`: `sudo reboot`, espera unos 30 segundos y vuelve a entrar por SSH.

## 1.2 Swap de 2 GB

Swap es disco usado como memoria de reserva. Con 2 GB de RAM, evita que `composer` o `npm` mueran
por falta de memoria durante la instalación. Es lento: es un colchón, no RAM de verdad.

```bash
sudo fallocate -l 2G /swapfile        # crea un fichero de 2 GB
sudo chmod 600 /swapfile              # solo root puede leerlo (puede contener datos de memoria)
sudo mkswap /swapfile                 # le da formato de swap
sudo swapon /swapfile                 # lo activa ahora
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab               # y en cada arranque
echo 'vm.swappiness=10' | sudo tee /etc/sysctl.d/99-swap.conf             # usar swap solo si hace falta
sudo sysctl --system | grep swappiness
```

`vm.swappiness=10`: el kernel prefiere la RAM y solo tira de swap cuando se queda corto (por defecto es 60).

## 1.3 Actualizaciones de seguridad automáticas

La AMI de Ubuntu ya trae `unattended-upgrades` activo. Solo lo comprobamos:

```bash
systemctl is-enabled unattended-upgrades
cat /etc/apt/apt.conf.d/20auto-upgrades
```

Esperado: `enabled`, y las dos líneas con `"1"`. Instala cada día los parches de seguridad.

## 1.4 Firewall (UFW)

El security group de AWS ya filtra. UFW es una segunda capa dentro del servidor (defensa en profundidad):
si alguien abre de más el security group, el servidor sigue protegido.

**El orden importa**: primero permitir SSH y después activar. Al revés, te quedas fuera.

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw enable                       # responde y
sudo ufw status verbose
```

Esperado: `Status: active`, entrada por defecto `deny`, y permitidos `22/tcp` y `80/tcp` (IPv4 e IPv6).

## 1.5 fail2ban

Lee los intentos fallidos de SSH y bloquea temporalmente la IP que insiste.

```bash
sudo apt install -y fail2ban
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
```

Esperado: `Status for the jail: sshd`, con contadores a 0.

## 1.6 SSH: solo llave y sin root

La AMI de Ubuntu ya viene así. Lo comprobamos leyendo la configuración efectiva:

```bash
sudo sshd -T | grep -E '^(passwordauthentication|permitrootlogin|pubkeyauthentication)'
```

Esperado: `passwordauthentication no` y `pubkeyauthentication yes`. `permitrootlogin` sale `without-password`,
pero la AMI además bloquea la llave de root (`/root/.ssh/authorized_keys` solo muestra un mensaje),
así que en la práctica no se entra como root.

## Zona horaria

Se queda en **UTC**. Es la práctica habitual en servidores: los logs no saltan con el horario de verano y
se cruzan fácil entre máquinas. La hora que ven los usuarios la fija ILIAS en su configuración.

## Comprobación final del paso 1

```bash
free -h | grep Swap                   # Swap: 2.0Gi
swapon --show                         # /swapfile file 2G
sudo ufw status | head -1             # Status: active
systemctl is-active fail2ban          # active
```
