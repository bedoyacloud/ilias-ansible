# 3. PHP y el pool de PHP-FPM

Objetivo: dar a PHP los valores que ILIAS recomienda y crear un pool de procesos propio para ILIAS.
Todo se ejecuta dentro del servidor.

## 3.1 Valores de PHP (`99-ilias.ini`)

PHP lee todos los ficheros de `conf.d/` en orden alfabético y gana el último. Con el prefijo `99-` nuestros
valores pisan los de Ubuntu sin tocar sus ficheros. Hay dos PHP separados y los dos necesitan la configuración:

- `fpm`: el que atiende la web.
- `cli`: el que usarán `setup.php` y el cron desde la terminal.

```bash
sudo tee /etc/php/8.3/fpm/conf.d/99-ilias.ini > /dev/null <<'EOF'
max_execution_time = 600
memory_limit = 512M
error_reporting = E_ALL & ~E_NOTICE & ~E_DEPRECATED & ~E_STRICT
display_errors = Off
post_max_size = 256M
upload_max_filesize = 256M
max_input_vars = 10000
allow_url_fopen = 1
session.gc_probability = 1
session.gc_divisor = 100
session.gc_maxlifetime = 14400
session.cookie_httponly = On
; PoC sin HTTPS: con On, el navegador no guarda la cookie de sesion y el login falla.
; Al poner dominio y HTTPS, cambiar a On.
session.cookie_secure = Off
session.save_handler = files
expose_php = Off
session.upload_progress.enabled = 0
EOF
sudo ln -sf /etc/php/8.3/fpm/conf.d/99-ilias.ini /etc/php/8.3/cli/conf.d/99-ilias.ini
```

Se crea una sola vez y el CLI usa un enlace simbólico al mismo fichero, así no hay dos copias que se desincronicen.

Qué hace cada valor:

| Valor | Por qué |
|---|---|
| `memory_limit = 512M` | Límite por petición. ILIAS es pesado; con menos, falla en tareas grandes |
| `post_max_size`, `upload_max_filesize = 256M` | Tamaño máximo de las subidas de ficheros |
| `max_input_vars = 10000` | Formularios largos (administración, tests) |
| `display_errors = Off` | Los errores van al log, no a la pantalla del usuario |
| `expose_php = Off` | No anuncia la versión de PHP en las cabeceras |
| `session.cookie_httponly = On` | JavaScript no puede leer la cookie de sesión |
| `session.cookie_secure` | **Off solo en esta PoC.** Con `On` la cookie solo viaja por HTTPS |

## 3.2 Carpeta de logs

El pool de abajo escribe su log en `/var/www/logs`. Esa carpeta debe existir antes de reiniciar PHP-FPM:

```bash
sudo mkdir -p /var/www/logs
sudo chown www-data:www-data /var/www/logs
```

## 3.3 Pool propio `ilias`

Un pool es un grupo de procesos PHP con su propio usuario, socket y límites. Tener uno para ILIAS lo aísla
de cualquier otra aplicación. Con 2 GB de RAM limitamos los procesos: cada uno puede llegar a usar
bastante memoria (el `memory_limit` es 512M), y 20 procesos a la vez agotarían el servidor.

```bash
sudo tee /etc/php/8.3/fpm/pool.d/ilias.conf > /dev/null <<'EOF'
[ilias]
user = www-data
group = www-data
listen = /run/php/ilias.sock
listen.owner = www-data
listen.group = www-data
pm = dynamic
pm.max_children = 6
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
pm.max_requests = 500
request_terminate_timeout = 600
php_admin_value[error_log] = /var/www/logs/php-fpm-error.log
php_admin_flag[log_errors] = on
EOF
```

- `listen = /run/php/ilias.sock`: nginx hablará con PHP por un socket de fichero, sin abrir ningún puerto de red.
- `pm = dynamic`: crea y destruye procesos según la carga.
- `pm.max_children = 6`: tope de peticiones PHP simultáneas. En producción sería mayor.
- `pm.max_requests = 500`: cada proceso se recicla tras 500 peticiones, para frenar fugas de memoria.

Desactiva el pool `www` que trae Ubuntu, porque no lo usamos y gasta memoria:

```bash
sudo mv /etc/php/8.3/fpm/pool.d/www.conf /etc/php/8.3/fpm/pool.d/www.conf.disabled
```

## 3.4 Comprobar y reiniciar

```bash
sudo php-fpm8.3 -t                    # valida la sintaxis: debe acabar en "test is successful"
sudo systemctl restart php8.3-fpm
systemctl is-active php8.3-fpm        # active
ls -l /run/php/ilias.sock             # el socket existe, dueño www-data
php -i | grep -E '^(memory_limit|upload_max_filesize|session.cookie_secure)'
```

Esperado en el último comando: `512M`, `256M` y `Off`. Si sale otro valor, el CLI no está leyendo el fichero.
