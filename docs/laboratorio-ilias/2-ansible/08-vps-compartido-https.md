# A8. Desplegar en un VPS compartido con HTTPS (Cloudflare)

Los servidores dedicados o VPS reales casi nunca están vacíos: ya alojan otros sitios, tienen su firewall y su HTTPS. Este caso
lo cubre con **los mismos roles**, cambiando solo variables. Es el escenario más parecido a un servidor dedicado de producción.

## A8.1 Qué se encontró (antes de tocar nada)

Antes de ejecutar nada se inspeccionó el servidor **en modo solo lectura**: IP pública, sistema, servicios, puertos, firewall, sitios
de nginx, bases de datos y contenedores. Lo que cambia respecto al servidor vacío de AWS:

| Hallazgo | Consecuencia |
|---|---|
| Hay otros sitios en el mismo nginx, y ya existe un `default_server` | El virtual host de ILIAS **no** puede ser `default_server` (`nginx_default_server: false`) |
| El firewall solo admite 80 y 443 desde los rangos de Cloudflare | El rol `base` **no debe ejecutarse**: abriría el puerto 80 a todo internet (`manage_system: false`) |
| HTTPS terminado con el certificado de origen de Cloudflare | La plantilla de nginx necesita un modo TLS (`nginx_tls: true`) |
| Ya había una instalación de ILIAS de demostración | Se retiró antes (ver A8.3) |

## A8.2 Qué se añadió al código

Tres variables nuevas en [main.yml](../../../ansible/inventory/group_vars/all/main.yml), con los valores por defecto del servidor de AWS (HTTP):

| Variable | Por defecto | En el VPS compartido |
|---|---|---|
| `manage_system` | `true` | `false`: el playbook se salta el rol `base` (la condición está en `site.yml`) |
| `nginx_tls`, `nginx_tls_cert`, `nginx_tls_key` | `false` | `true`, con el certificado de origen de Cloudflare: añade el bloque 443, la redirección 80 → 443 y `fastcgi_param HTTPS on` |
| `nginx_server_name`, `nginx_default_server` | `_`, `true` | el dominio, y `false` |

El virtual host sigue siendo **una sola plantilla**: con `nginx_tls: false` genera exactamente lo mismo que antes
(se comprobó en los servidores de AWS: `changed=0`).

Las variables propias de cada servidor van en el inventario (`hosts.yml`, que no se sube), en el propio host:

```yaml
ilias-vps:
  ansible_host: IP_DEL_SERVIDOR
  ansible_ssh_private_key_file: ~/.ssh/TU_LLAVE
  manage_system: false              # no tocar firewall ni sistema
  ilias_version: v11.4              # otra versión que en AWS
  ilias_url: https://ilias.ejemplo.com
  nginx_tls: true
  nginx_server_name: ilias.ejemplo.com
  nginx_default_server: false
  php_cookie_secure: "On"           # ahora hay HTTPS
  php_fpm_max_children: 20          # el servidor tiene más RAM
  mariadb_buffer_pool: 1G
```

Y sus **secretos propios**, en `inventory/host_vars/ilias-vps/vault.yml` (cifrado y fuera de git). Las variables del host
ganan a las de `group_vars`, así que cada servidor tiene contraseñas distintas.

## A8.3 Retirar la instalación anterior (con copia)

Cuando hay una instalación previa que se va a reemplazar, antes de borrar nada:

1. **Copia** de la base de datos y de la configuración a otra máquina (`mariadb-dump` y `tar`, por SSH), y comprobar que se pueden leer.
2. **Retirar solo lo de ILIAS**: su base y su usuario, su directorio, su virtual host y sus temporizadores. No el firewall, ni los otros sitios, ni Docker.
3. Comprobar que **los otros sitios siguen respondiendo**.

## A8.4 Desplegar y comprobar

```bash
cd ansible
export ANSIBLE_VAULT_PASSWORD_FILE=$PWD/../.vault-pass
ansible-playbook playbooks/site.yml --limit ilias-vps       # 1.ª vez
ansible-playbook playbooks/site.yml --limit ilias-vps       # 2.ª vez: changed=0
```

Resultados:

| Prueba | Resultado |
|---|---|
| 1.ª ejecución | `ok=45 changed=29 failed=0`, en unos 5 minutos. Las 14 tareas del rol `base` quedaron omitidas |
| 2.ª ejecución | `changed=0`, en unos 70 segundos |
| `https://ilias.ejemplo.com/login.php` | 200. `/.git/config` y `/data/...` dan 404 |
| `http://` | 301 hacia `https://` |
| Versión desplegada | `v11.4` |
| `root` con la contraseña de este servidor | Entra. Con la de AWS y con `homer`, no |
| Plugin GeoGebra | Activo |
| Otros sitios del servidor | Siguen en 200 |
| Firewall | No se tocó (el rol `base` no se ejecutó) |

## A8.5 Qué demuestra

El mismo código despliega en un servidor vacío con HTTP y en un VPS compartido con HTTPS, otra versión de ILIAS y secretos propios,
sin editar ningún rol: solo variables del inventario. Esa es la propiedad que importa al pasar a servidores dedicados de otro proveedor.
