# 9. Comprobaciones finales y backup

Objetivo: demostrar que la instalación aguanta (reinicio, seguridad) y que se puede **recuperar**.
Un backup que nunca se ha restaurado no es un backup.
Todo se ejecuta dentro del servidor, salvo lo indicado.

## 9.1 Qué hay que guardar

| Qué | Dónde | Por qué |
|---|---|---|
| Base de datos | MariaDB, base `ilias` | Usuarios, cursos, estructura, resultados |
| Ficheros subidos | `/var/www/files` | Lo que suben los usuarios. **Sin esto, la BD apunta a ficheros que no existen** |
| Configuración de ILIAS | `/var/www/config/ilias.json` | Lleva la contraseña de la BD |
| Configuración del sistema | nginx, PHP, MariaDB, cron | Para reconstruir el servidor igual |

El código de ILIAS **no** se guarda: se vuelve a descargar con `git clone --branch v11.5`.

Los tres primeros deben guardarse **a la vez**: BD y ficheros de un mismo instante, si no, quedan descuadrados.

## 9.2 Script de backup

Se ejecuta como root, porque root entra en MariaDB por socket sin contraseña y así no hay que guardarla en el script.

```bash
sudo tee /usr/local/sbin/ilias-backup.sh > /dev/null <<'EOF'
#!/bin/bash
# Backup de ILIAS: base de datos + ficheros + configuracion
set -euo pipefail
umask 077                                   # todo lo que se cree es solo de root

DEST=/var/backups/ilias
KEEP_DAYS=7
TS=$(date -u +%Y%m%dT%H%M%SZ)
OUT="$DEST/$TS"
mkdir -p "$OUT"

# 1) Base de datos (--single-transaction: copia consistente sin bloquear la web)
mariadb-dump --single-transaction --routines --triggers ilias | gzip > "$OUT/db.sql.gz"

# 2) Ficheros subidos y configuracion de ILIAS
tar -czf "$OUT/files.tar.gz"  -C / var/www/files
tar -czf "$OUT/config.tar.gz" -C / var/www/config

# 3) Configuracion del sistema
tar -czf "$OUT/sistema.tar.gz" -C / \
  etc/nginx/sites-available/ilias \
  etc/php/8.3/fpm/conf.d/99-ilias.ini \
  etc/php/8.3/fpm/pool.d/ilias.conf \
  etc/mysql/mariadb.conf.d/60-ilias.cnf \
  etc/cron.d/ilias

# 4) Sumas de verificacion para detectar corrupcion
( cd "$OUT" && sha256sum *.gz > SHA256SUMS )

# 5) Borrar copias antiguas
find "$DEST" -mindepth 1 -maxdepth 1 -type d -mtime +"$KEEP_DAYS" -exec rm -rf {} +

echo "Backup OK: $OUT ($(du -sh "$OUT" | cut -f1))"
EOF
sudo chmod 700 /usr/local/sbin/ilias-backup.sh
sudo mkdir -p /var/backups/ilias && sudo chmod 700 /var/backups/ilias
```

Puntos clave:
- `set -euo pipefail`: si **cualquier** paso falla, el script se detiene. Sin eso, un `mariadb-dump` roto dejaría un fichero vacío "con éxito".
- `umask 077`: los backups contienen datos y contraseñas, así que solo root los lee.
- `--single-transaction`: la copia es consistente con la web funcionando.
- `sha256sum`: permite comprobar más tarde que el fichero no se corrompió.

Ejecútalo una vez a mano:

```bash
sudo /usr/local/sbin/ilias-backup.sh
sudo sh -c 'ls -lh /var/backups/ilias/*/'
```

Esperado: `db.sql.gz`, `files.tar.gz`, `config.tar.gz`, `sistema.tar.gz` y `SHA256SUMS`.

Ojo: `/var/backups/ilias` es de root con permisos `700`. Un comodín (`*`) lo expande **tu shell**, que no puede
leer esa carpeta, así que `sudo ls /var/backups/ilias/*/` falla. Por eso el comando va dentro de `sudo sh -c '...'`:
así el comodín lo expande root.

## 9.3 Programarlo

```bash
sudo tee /etc/cron.d/ilias-backup > /dev/null <<'EOF'
# Backup diario de ILIAS a las 02:30 UTC
30 2 * * * root /usr/local/sbin/ilias-backup.sh >> /var/log/ilias-backup.log 2>&1
EOF
sudo chmod 644 /etc/cron.d/ilias-backup
```

## 9.4 Probar la restauración de la base de datos

Restauramos en una base **temporal** para no tocar la real:

```bash
LAST=/var/backups/ilias/$(sudo ls -1 /var/backups/ilias | tail -1)/    # sin comodines: los expandiria tu shell, no root
echo $LAST
sudo sh -c "cd $LAST && sha256sum -c SHA256SUMS"          # todas deben decir OK
sudo mariadb -e "CREATE DATABASE ilias_restore_test CHARACTER SET utf8mb3 COLLATE utf8mb3_general_ci;"
sudo sh -c "gunzip -c ${LAST}db.sql.gz | mariadb ilias_restore_test"
sudo mariadb -e "SELECT table_schema AS bd, COUNT(*) AS tablas FROM information_schema.tables WHERE table_schema IN ('ilias','ilias_restore_test') GROUP BY table_schema;"
sudo mariadb -e "DROP DATABASE ilias_restore_test;"
```

Esperado: las dos bases con el **mismo número de tablas**. Después, la temporal se borra.

Y comprueba que los tar se pueden leer:

```bash
sudo tar -tzf ${LAST}config.tar.gz
sudo tar -tzf ${LAST}files.tar.gz | head -5
```

## 9.5 Copia fuera del servidor

Todo lo anterior vive en **el mismo servidor**: si se pierde el disco, se pierde también el backup.
Desde tu Mac, descarga la última copia:

```bash
mkdir -p ~/backups-ilias
ssh -i ~/.ssh/mi-llave.pem ubuntu@IP_DEL_SERVIDOR \
  'sudo tar -czf - -C /var/backups/ilias "$(sudo ls -1 /var/backups/ilias | tail -1)"' \
  > ~/backups-ilias/ilias-$(date +%Y%m%d).tar.gz
ls -lh ~/backups-ilias/
```

Para producción lo normal es subir las copias a **S3** (con un rol de IAM solo de escritura) o usar `restic`.
Además, en AWS: **EC2 → Volúmenes → tu volumen → Acciones → Crear instantánea** guarda una imagen de todo el disco.

## 9.6 Prueba de reinicio

Comprueba que todo arranca solo, sin tocar nada a mano:

```bash
sudo reboot
```

Espera 1 minuto, vuelve a entrar por SSH y ejecuta:

```bash
systemctl is-active nginx php8.3-fpm mariadb fail2ban cron
free -h | grep Swap                    # Swap: 2.0Gi (esta en fstab)
sudo ufw status | head -1              # active
curl -sI http://localhost/ | head -1   # 200
```

Si la IP **no** es una Elastic IP, habrá cambiado: cámbiala en tu navegador, en `inventario.ini` y en `http.path` de `ilias.json`.

## 9.7 Checklist final

```bash
curl -s -o /dev/null -w ".git: %{http_code}\n"     http://localhost/.git/config          # 404
curl -s -o /dev/null -w "data: %{http_code}\n"     http://localhost/data/pocilias/x/y    # 404
curl -s -o /dev/null -w "ilias.json: %{http_code}\n" http://localhost/config/ilias.json  # 404
sudo ss -ltnp | grep 3306                                                                # solo 127.0.0.1
sudo -u www-data php /var/www/ilias/cli/setup.php status | grep -B2 'update_required: true'   # los 2 conocidos
df -h / | tail -1                                                                        # espacio libre
```
