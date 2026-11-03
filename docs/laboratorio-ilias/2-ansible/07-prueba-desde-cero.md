# A7. Prueba desde cero: un servidor vacío, un comando

La prueba que demuestra que el despliegue es reproducible: no depende de nada que hiciéramos a mano en `ilias-01`.

## A7.1 El punto de partida

Una EC2 recién creada (Ubuntu 24.04, `t3.small`, 30 GiB), sin tocar. Antes de empezar se comprobó:

| | Estado |
|---|---|
| Sistema | Ubuntu 24.04.5 LTS, 2 GB de RAM, 29 GB de disco |
| Paquetes de ILIAS (nginx, MariaDB, PHP) | 0 instalados |
| Swap | Ninguna |
| Firewall (UFW) | Inactivo |

## A7.2 Qué se hace

1. Se añade el servidor al inventario (`hosts.yml`, que no se sube al repo) y se acepta su huella SSH:
   ```yaml
   ilias:
     hosts:
       ilias-01:
         ansible_host: IP_DE_ILIAS_01
       ilias-fresh:
         ansible_host: IP_DEL_SERVIDOR_NUEVO
   ```
2. Un solo comando, limitado al servidor nuevo para no tocar el otro:
   ```bash
   cd ansible
   export ANSIBLE_VAULT_PASSWORD_FILE=$PWD/../.vault-pass
   ansible-playbook playbooks/site.yml --limit ilias-fresh
   ```
3. El mismo comando otra vez.

## A7.3 Resultados

| Ejecución | Resultado |
|---|---|
| 1.ª (servidor vacío) | `ok=59 changed=46 failed=0`: instaló y configuró todo, incluido el addon GeoGebra |
| 2.ª | `ok=46 changed=0 failed=0`, en **56 segundos** |

La primera ejecución ya se había cronometrado antes sin el addon: unos 5 minutos. En esta no se registró el tiempo exacto.

Comprobaciones posteriores en el servidor nuevo:

| Comprobación | Resultado |
|---|---|
| Web (`login.php`) | 200 |
| `/.git/config` y `/data/...` | 404 (no se exponen) |
| Entrar como `root` con la contraseña del Vault | Entra |
| Entrar como `root` con la contraseña por defecto (`homer`) | No entra |
| `ilias.json` | `600`, dueño `www-data` |
| MariaDB | Solo en `127.0.0.1` |
| nginx, PHP-FPM, MariaDB, fail2ban, cron y UFW | Activos |
| Swap | 2 GB |
| Plugin GeoGebra | Registrado y activo (`srgg`, `active=1`) |
| Cron de ILIAS a mano | Sale con 0 |
| Backup | Se genera correctamente |

## A7.4 Qué demuestra

- **Reproducibilidad:** del servidor vacío a ILIAS con el addon funcionando, sin ningún paso manual.
- **Idempotencia:** repetir el comando no cambia nada, de modo que es seguro volver a lanzarlo, por ejemplo desde el pipeline.
- **Seguridad por defecto:** cada servidor sale con firewall, fail2ban, secretos con permisos restrictivos y la contraseña inicial de ILIAS cambiada.
- **Portabilidad:** el código solo supone Ubuntu 24.04 y SSH. Cambiar de AWS a un servidor dedicado de Hetzner es cambiar la IP del inventario
  (más un paso previo de *bootstrap* si solo hay acceso como `root`).

## A7.5 Limpieza

Cuando termines, **detén o elimina** la instancia de prueba en AWS: cada instancia encendida cuesta dinero.
