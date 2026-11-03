# A2. PHP, MariaDB y Vault

Equivale a los pasos [2](../1-instalacion-manual/02-paquetes.md), [3](../1-instalacion-manual/03-php.md) y [4](../1-instalacion-manual/04-mariadb.md).

## A2.1 Plantillas y handlers: el rol `php`

Las plantillas son los ficheros de configuración con huecos. En vez de escribir a mano `99-ilias.ini` o el pool de FPM, hay una plantilla
(`roles/php/templates/*.j2`) y las variables rellenan los huecos:

```ini
pm.max_children = {{ php_fpm_max_children }}
session.cookie_secure = {{ php_cookie_secure }}
```

Para pasar a HTTPS se cambia `php_cookie_secure: "On"` en `main.yml`, no el fichero. `{{ ansible_managed }}` deja un comentario avisando
de que el fichero lo gestiona Ansible y que los cambios a mano se pierden.

**Handler.** Cuando una plantilla cambia, PHP-FPM hay que reiniciarlo; si no cambió, no. La tarea lleva `notify: Restart php-fpm` y el reinicio
está en `handlers/main.yml`. Los handlers:

- solo se ejecutan si alguna tarea que los notifica **cambió** algo;
- se ejecutan **una vez**, al final (aunque lo notifiquen cinco tareas);
- si no hay cambios, no hay reinicio. Eso es lo que hace que la segunda ejecución sea limpia.

También hay un `loop` (la plantilla `99-ilias.ini` se escribe para `fpm` y para `cli`), la lista de extensiones como variable
(`php_extensions`) y `command` con `removes` para desactivar `www.conf` solo si existe.

## A2.2 El rol `mariadb`

- Instala `mariadb-server` y **`python3-pymysql`**: los módulos de MariaDB de Ansible se ejecutan en el servidor y necesitan ese conector.
- `meta: flush_handlers` fuerza que el reinicio de MariaDB ocurra **antes** de crear la base, no al final del playbook.
- La base y el usuario se crean con `mysql_db` y `mysql_user`, conectando por el **socket de root** (`login_unix_socket`), sin contraseña de administrador.
- `priv: "ilias.*:ALL/*.*:LOCK TABLES"`: todos los permisos sobre su base y solo `LOCK TABLES` global, para los backups. Mínimo privilegio.
- `no_log: true` en la tarea del usuario: la contraseña no aparece en la salida de Ansible.

Los módulos `mysql_*` avisan de que dejarán de dar soporte a MariaDB en la versión 6; el reemplazo es la colección `ansible.mariadb`.
Es una mejora pendiente para el futuro, no un fallo.

## A2.3 Vault: los secretos

Las contraseñas no pueden estar en texto plano en el repositorio. Ansible Vault cifra un fichero con una contraseña maestra:

```bash
cd ansible
openssl rand -hex 24 > ../.vault-pass && chmod 600 ../.vault-pass       # contraseña maestra (ignorada por git)
cp inventory/group_vars/all/vault.example.yml inventory/group_vars/all/vault.yml
# edita vault.yml con tus contraseñas y luego:
ansible-vault encrypt inventory/group_vars/all/vault.yml
```

Uso diario:

```bash
export ANSIBLE_VAULT_PASSWORD_FILE=$PWD/../.vault-pass
ansible-vault view inventory/group_vars/all/vault.yml     # ver
ansible-vault edit inventory/group_vars/all/vault.yml     # editar
```

- Las variables del Vault se llaman `vault_*` y se usan en las plantillas como cualquier otra (`{{ vault_ilias_db_password }}`).
- La ruta de la contraseña maestra va en una **variable de entorno**, no en `ansible.cfg`: si el fichero no existiera, Ansible fallaría
  en *todos* los comandos, incluso sin usar secretos.
- `vault.yml` y `.vault-pass` están en `.gitignore`. En un repo público ni siquiera se sube el fichero cifrado;
  solo `vault.example.yml`.
- Las contraseñas de este laboratorio (la de la base y la de `root` de ILIAS) las generó `openssl` al azar.
  Para ver la de `root` y entrar en ILIAS: `ansible-vault view inventory/group_vars/all/vault.yml`.
