# 7. nginx

Objetivo: publicar ILIAS en el puerto 80 y poder entrar por el navegador.
Todo se ejecuta dentro del servidor, salvo el 7.4 y el 7.5, que se hacen en el navegador.

**Antes de empezar:** el puerto 80 del security group debe seguir restringido a "Mi IP" (paso 6.0).

## Cómo encaja

```
navegador ─ :80 ─> nginx ─ /run/php/ilias.sock ─> PHP-FPM (pool ilias)
                     │
                     ├─ ficheros estáticos (css, js, imágenes): los sirve nginx directamente desde public/
                     └─ *.php: se los pasa a PHP-FPM
```

nginx no ejecuta PHP: sirve ficheros y reenvía los `.php` al pool que creamos en el paso 3.

## 7.1 El virtual host

El virtual host es la configuración de un sitio. Se escribe en `sites-available/` y se activa con un enlace en `sites-enabled/`.

```bash
sudo tee /etc/nginx/sites-available/ilias > /dev/null <<'EOF'
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;

    root /var/www/ilias/public;
    index index.php;

    client_max_body_size 256M;
    server_tokens off;
    add_header X-Content-Type-Options nosniff always;
    add_header Referrer-Policy strict-origin-when-cross-origin always;

    # Ficheros ocultos y areas internas (equivale al RedirectMatch 404 del .htaccess de ILIAS)
    location ~ /\.(?!well-known) { return 404; }
    location ~ ^/(patches|\.git|\.github) { return 404; }

    # Control de acceso a ficheros (Web Access Checker): data/<client>/<x>/<fichero> -> wac.php
    location ~ ^/data/[^/]+/[^/]+/.+ {
        rewrite ^ /wac.php last;
    }

    # Atajos goto
    rewrite ^/go/(.*)$ /goto.php/$1 last;
    rewrite ^/goto_(.*)_(wiki_([0-9]+|wpage)(.*))\.html$ /goto.php?client_id=$1&target=$2 last;
    rewrite ^/goto_(.*)_(usr_([a-z]+))\.html$ /goto.php?client_id=$1&target=$2 last;
    rewrite ^/goto_(.*)_([a-z]+_[0-9]+(.*))\.html$ /goto.php?client_id=$1&target=$2 last;

    location / {
        try_files $uri $uri/ @shortlink;
    }
    location @shortlink {
        rewrite ^/([A-Za-z0-9_-]+)/?$ /goto.php/shortlink/$1 last;
        return 404;
    }

    location ~ \.php(/|$) {
        fastcgi_split_path_info ^(.+?\.php)(/.*)$;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
        fastcgi_param HTTP_AUTHORIZATION $http_authorization;
        fastcgi_pass unix:/run/php/ilias.sock;
        fastcgi_read_timeout 600s;
    }
}
EOF
```

Qué hace cada parte:

| Parte | Por qué |
|---|---|
| `root /var/www/ilias/public` | Solo `public/` es accesible desde la web. El código, `files/` y `config/` quedan fuera |
| `location ~ /\.(?!well-known)` | Devuelve 404 a cualquier fichero oculto (`.git`, `.env`, `.htaccess`...) |
| `location ~ ^/data/...` → `wac.php` | Los ficheros subidos no se sirven directamente: pasan por el Web Access Checker, que comprueba si el usuario tiene permiso |
| `fastcgi_split_path_info` + `PATH_INFO` | ILIAS usa URLs del tipo `/script.php/parametros`; sin esto darían 404 |
| `client_max_body_size 256M` | Tamaño máximo de subida: debe ser igual que `upload_max_filesize` de PHP |
| `fastcgi_read_timeout 600s` | Operaciones largas (importaciones, exportaciones) sin cortar la conexión |
| `server_tokens off` | No anuncia la versión de nginx |

A diferencia de la plantilla de Ansible del repo, aquí **no hay HTTPS ni redirección 80→443 ni `fastcgi_param HTTPS on`**:
es la versión para la PoC con HTTP. Con dominio y certificado (paso 10) se vuelve a añadir.

## 7.2 Activar el sitio

El sitio `default` de Ubuntu (la página "Welcome to nginx") también escucha en el 80 y chocaría con el nuestro:

```bash
sudo ln -s /etc/nginx/sites-available/ilias /etc/nginx/sites-enabled/ilias
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
```

Esperado: `syntax is ok` y `test is successful`. **Si hay error, no recargues**: pégame la salida.

```bash
sudo systemctl reload nginx
```

`reload` aplica la configuración sin cortar las conexiones activas; `restart` las corta.

## 7.3 Comprobar desde el servidor

```bash
curl -sI http://localhost/ | head -5
curl -s -o /dev/null -w "%{http_code}\n" http://localhost/.git/config
curl -s -o /dev/null -w "%{http_code}\n" http://localhost/data/pocilias/x/y
sudo tail -5 /var/log/nginx/error.log
```

Esperado:
- La primera: una respuesta `HTTP/1.1 302` (redirige a `login.php`) o `200`.
- La segunda: `404`, porque `.git` está bloqueado.
- La tercera: `404` o `403`, nunca el contenido de un fichero.
- Sin errores nuevos en `error.log`.

## 7.4 Abrir en el navegador

Desde tu Mac: `http://IP_DEL_SERVIDOR` (desde la IP que permitiste en el security group).

Deberías ver la pantalla de login de ILIAS. Si no carga:
    
| Síntoma | Causa probable |
|---|---|
| Se queda cargando (*timed out*) | Tu IP ha cambiado y ya no coincide con la regla del puerto 80 del security group |
| `502 Bad Gateway` | PHP-FPM no responde: `systemctl status php8.3-fpm` y `sudo tail /var/log/nginx/error.log` |
| `404` o `403` | Ruta de `root` mal escrita, o dueño de los ficheros |
| Página en blanco | Error de PHP: mira `/var/www/logs/php-fpm-error.log` |

## 7.5 Cambiar la contraseña de `root` (inmediatamente)

1. En el navegador entra con usuario **`root`** y contraseña **`homer`**.
2. Arriba a la derecha, el menú de tu usuario → **Configuración** → **Contraseña**.
3. Pon una contraseña larga y única, y guárdala en tu gestor de contraseñas.
4. Cierra sesión y vuelve a entrar con la nueva, para comprobar que funciona.

**Solo después de esto**, vuelve al security group `ilias-sg` y deja el puerto 80 en *Anywhere IPv4* si quieres que sea
accesible desde fuera (o déjalo en tu IP mientras sigas probando).

## Comprobación final del paso 7

```bash
sudo nginx -t
systemctl is-active nginx php8.3-fpm mariadb
sudo ss -ltnp | grep -E ':(80|3306)\s'      # nginx en 0.0.0.0:80; mariadb solo en 127.0.0.1:3306
```
