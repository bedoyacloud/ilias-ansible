# A9. Backup y restauración (manual y automatizada)

## A9.1 Qué hay que proteger

| Pieza | Dónde vive | En el backup | Si se pierde |
|---|---|---|---|
| Base de datos | MariaDB | `db.sql.gz` | Se pierde todo el contenido |
| Ficheros subidos y `pwsalt.txt` | `ilias_data_dir` | `files.tar.gz` | Se pierden los archivos y las contraseñas con *salt* dejan de validar |
| `ilias.json` | `ilias_config_dir` | `config.tar.gz` | Se regenera con Ansible |
| Configuración de nginx, PHP, MariaDB (y el certificado de origen, si existe) | `/etc/...` | `sistema.tar.gz` | Se regenera con Ansible; el certificado **no** |
| Versión con la que se hizo | — | `MANIFEST` | — |
| Código de ILIAS y plugins | `ilias_root` | No hace falta | Se reconstruye con Ansible |
| Vault y `.vault-pass` | Solo en el equipo del administrador | **No** | Sin ellos no se pueden rehacer los servidores |

## A9.2 Manual

**Backup**
```bash
TS=$(date -u +%Y%m%dT%H%M%SZ); OUT=/var/backups/ilias/$TS
sudo install -d -m 700 $OUT
sudo sh -c "mariadb-dump --single-transaction --routines --triggers ilias | gzip > $OUT/db.sql.gz"
sudo tar -czf $OUT/files.tar.gz  -C / var/www/files
sudo tar -czf $OUT/config.tar.gz -C / var/www/config
sudo sh -c "cd $OUT && sha256sum *.gz > SHA256SUMS"
ssh servidor "sudo tar -czf - -C /var/backups/ilias $TS" > ilias-$TS.tar.gz     # sacarlo del servidor
```

**Restaurar**
1. `sha256sum -c SHA256SUMS`: la copia no está corrupta.
2. `sudo systemctl stop nginx cron`.
3. Un `mariadb-dump` de lo que hay ahora, por si hay que volver atrás.
4. `DROP DATABASE ilias; CREATE DATABASE ilias CHARACTER SET utf8mb3 COLLATE utf8mb3_general_ci;` y `gunzip -c db.sql.gz | mariadb ilias`. Los permisos del usuario sobreviven al `DROP`.
5. `tar -xzf files.tar.gz -C /` (como root conserva el dueño `www-data`).
6. `sudo systemctl start nginx cron` y verificar: nº de tablas, login, abrir un fichero.

Para un servidor perdido, antes de esto se levanta uno nuevo con el playbook. Hay dos condiciones: **la misma versión de ILIAS** que la copia y **las mismas contraseñas del Vault**.

## A9.3 Automatizado

### Backup
El rol `ilias_maintenance` instala el script y su cron (por defecto 02:30 UTC). Hace lo mismo que el manual, más:

| Mejora | Cómo |
|---|---|
| `MANIFEST` | Versión de ILIAS, cliente, PHP y nº de tablas. Es lo que permite que la restauración se niegue a mezclar versiones |
| Contenido opcional del sistema | Incluye el vhost de Grafana y `/etc/ssl/cloudflare` si existen en ese servidor |
| Copia cifrada fuera del servidor | `backup_restic_repo` (por ejemplo `s3:...` o `sftp:...`). Usa `restic`, con retención diaria y semanal. Vacío = desactivado |
| Aviso de éxito | `backup_heartbeat_url`: el script llama a esa URL al terminar bien; si no llega, el monitor avisa |

### Restaurar: [`playbooks/restore.yml`](../../../ansible/playbooks/restore.yml)
```bash
ansible-playbook playbooks/restore.yml --limit SERVIDOR \
  -e backup_dir=/var/backups/ilias/20261104T023001Z -e confirm_restore=yes
```
Antes de cambiar nada comprueba, y se niega si falla cualquiera:
- hay `--limit` (nunca restaura "a todos") y `confirm_restore=yes`;
- el backup tiene sus cuatro ficheros;
- la versión de ILIAS y el cliente del `MANIFEST` coinciden con los del servidor;
- las sumas `SHA256SUMS` son correctas.

Después: copia de seguridad de lo actual, para nginx y cron, recrea la base y carga el volcado, sustituye los ficheros, arranca, y comprueba que `login.php` da 200 y que las tablas coinciden con el `MANIFEST`.
**Si falla a medias deja nginx y cron parados a propósito** (no sirve datos a medio restaurar) y dice dónde está la copia previa.

## A9.4 Simulacro de desastre (hecho)

Se probó la ruta completa en `ilias-fresh`, con datos reconocibles:

| Paso | Qué se hizo |
|---|---|
| Antes | Una marca en la base de datos, otra en un fichero, y el hash de `pwsalt.txt`. Backup, y copia **fuera del servidor** |
| Desastre | Se eliminó la base, el código, los ficheros, la configuración y los backups locales. La web daba 404 |
| Reconstruir | `site.yml`: **305 s**. ILIAS nuevo y vacío (web 200, sin marcas, sin `pwsalt.txt`) |
| Restaurar | La copia subida desde fuera, y `restore.yml`: **52 s** |

| Comprobación | Antes | Después |
|---|---|---|
| Marca en la base | `marca-…` | la misma |
| Marca en el fichero | `marca-…` | la misma |
| `pwsalt.txt` (sha256) | `bb07349e…` | **el mismo** |
| Tablas | 1088 | 1088 |
| `root` con la contraseña del Vault / con `homer` | — | entra / no entra |
| Plugins | `srgg`, `xlvo` | `srgg`, `xlvo` |

Además, `restic` se probó aparte: crea un *snapshot* cifrado y una restauración desde él devuelve el `MANIFEST` y la base.

## A9.5 Límites de lo probado (honestidad)

- El "servidor perdido" se simuló **borrando la aplicación, los datos y los backups**, no creando una máquina virtual nueva: el sistema operativo y los paquetes seguían.
- `restic` se probó contra un repositorio **local**, no contra un destino real (S3, Hetzner Storage Box). El código es el mismo, pero el destino remoto y sus credenciales no.
- El backup semanal o diario sigue siendo local mientras no se active `backup_restic_repo`: si el disco muere, muere con él.
- No hay todavía una prueba de restauración **periódica** automática. La de arriba es puntual.
- Con una copia al día, se pueden perder hasta 24 horas de datos.

## A9.6 Problemas reales encontrados

| Síntoma | Causa | Solución |
|---|---|---|
| `src (or content) is required` al crear el fichero de credenciales de restic | Con el diccionario de credenciales vacío, el contenido sale vacío y `copy` lo rechaza | El contenido lleva siempre una línea de comentario |
| `ansible-lint`: `curl` en lugar de `uri` | La prueba de humo necesita `--resolve` para que nginx reciba el nombre del sitio (SNI) | Se mantiene `curl`, con una excepción explicada |
| La restauración no debe mezclar versiones | Un volcado de otra versión de ILIAS no es compatible con el código | El `MANIFEST` y una comprobación previa |
