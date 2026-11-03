# A3. ILIAS, nginx y mantenimiento

Equivale a los pasos [5 y 6](../1-instalacion-manual/05-codigo-ilias.md), [7](../1-instalacion-manual/07-nginx.md),
[8](../1-instalacion-manual/08-cron.md) y [9](../1-instalacion-manual/09-comprobaciones-y-backup.md).

## A3.1 El rol `ilias`

1. **Node.js desde NodeSource.** `deb822_repository` añade el repositorio y descarga su clave (`signed_by`). Necesita `python3-debian` en el servidor.
2. **Código.** El módulo `git` clona el tag `{{ ilias_version }}` con `depth: 1`. Subir de versión es cambiar esa variable.
   `community.general.git_config` marca el directorio como seguro, porque su dueño (`www-data`) no es quien ejecuta git.
3. **Dependencias.** `npm clean-install` y `composer install` se ejecutan como `www-data` (`become_user`) y dejan un fichero
   `.built-v11.5`. Con `creates:` sobre ese fichero, solo se repiten si cambia la versión.
4. **Configuración.** `ilias.json` sale de una plantilla con la contraseña del Vault, con `mode: 0600`, dueño `www-data` y `no_log`.
5. **Instalación, solo si hace falta.** Se cuenta cuántas tablas tiene la base (`SELECT COUNT(*)`):
   si es 0, se ejecuta `setup.php install`; si no, se salta. Es mejor que un fichero "marca" porque **mira el estado real**:
   si alguien borra la base, Ansible la reinstala.
6. **Contraseña de `root` de ILIAS.** Tras instalar, ILIAS deja `root`/`homer`. Se calcula un hash `bcrypt` con PHP y se guarda
   en `usr_data` con el tipo `bcryptphp`. Solo ocurre en la instalación (`when: ilias_installed is changed`).
   Comprobado con un inicio de sesión real: entra con la contraseña del Vault y `homer` ya no.
7. `client.ini.php` pasa a 0600, porque guarda la contraseña de la base (el mismo cuidado del paso 6.4b manual).

`ilias_url` sale de `http://{{ ansible_host }}`, así que no hay IP escrita en ningún rol. Con dominio y HTTPS se cambia esa variable.

El cron de ILIAS usa `ilias_cron_user: root`, porque en el manual el usuario `cron-user` se creó a mano en el navegador y no hay forma
de crearlo por línea de comandos. Para producción conviene un usuario dedicado.

## A3.2 El rol `nginx`

Plantilla del virtual host (la misma del paso manual 7, con `{{ ilias_root }}`), un enlace en `sites-enabled`, borrar el sitio `default`
y **`nginx -t` antes de recargar**. Si la configuración está mal, `nginx -t` falla, el playbook se detiene
y el handler `Reload nginx` no llega a ejecutarse: el nginx en marcha no se toca.

## A3.3 El rol `ilias_maintenance`

- **Cron** de ILIAS cada 5 minutos con el módulo `cron` (`cron_file: ilias` crea `/etc/cron.d/ilias`), como `www-data`.
- **logrotate** por plantilla.
- **Backup:** plantilla `ilias-backup.sh.j2` con las rutas y retención como variables, en `/usr/local/sbin` (0700, root), y su cron a las 02:30.

Es el mismo script del paso 9 manual; ahora las rutas salen de las variables y no hay que mantenerlo en dos sitios.

## A3.4 Qué comprobar

```bash
ansible-playbook playbooks/site.yml          # 1ª vez: instala todo (unos 5 min)
ansible-playbook playbooks/site.yml          # 2ª vez: changed=0
curl -sI http://IP/ | head -1                # 200
curl -s -o /dev/null -w "%{http_code}\n" http://IP/.git/config    # 404
```
