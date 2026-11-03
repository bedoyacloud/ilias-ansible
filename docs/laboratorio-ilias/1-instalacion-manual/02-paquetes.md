# 2. Paquetes

Objetivo: instalar todo el software que ILIAS necesita. Todavía no se configura nada: eso va en los pasos 3 y 4.
Todo se ejecuta dentro del servidor.

| Pieza | Para qué sirve |
|---|---|
| nginx | Servidor web: recibe las peticiones HTTP |
| PHP 8.3 + FPM | ILIAS está escrito en PHP; FPM es el proceso que ejecuta el código y habla con nginx |
| MariaDB | Base de datos (ILIAS no soporta PostgreSQL) |
| Node 22 | Solo para compilar dependencias JavaScript de ILIAS (`npm`) |
| composer | Gestor de dependencias PHP |
| ImageMagick + Ghostscript | Miniaturas de imágenes y previsualización de PDF |
| git, unzip, acl | Descargar el código y descomprimir paquetes |

## 2.1 Paquetes de Ubuntu

```bash
sudo apt update
sudo apt install -y acl git unzip curl ca-certificates composer imagemagick ghostscript \
  nginx mariadb-server \
  php8.3-{fpm,cli,gd,xml,xsl,mysql,curl,zip,mbstring,imagick,soap,ldap,intl}
```

- `php8.3-{fpm,cli,...}` lo expande la shell en `php8.3-fpm php8.3-cli ...`.
- La guía original de `ilias-poc` no incluye `nginx` en esta línea, pero lo necesitamos: se añade aquí.
- Tarda unos minutos. Si pregunta por reiniciar servicios, deja lo marcado y pulsa Enter.

## 2.2 Node 22 (NodeSource)

Ubuntu 24.04 trae Node 18, que es demasiado viejo. Node 22 se instala desde el repositorio oficial de NodeSource:

```bash
sudo mkdir -p /etc/apt/keyrings
sudo curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key -o /etc/apt/keyrings/nodesource.asc
echo "deb [signed-by=/etc/apt/keyrings/nodesource.asc] https://deb.nodesource.com/node_22.x nodistro main" \
  | sudo tee /etc/apt/sources.list.d/nodesource.list
sudo apt update && sudo apt install -y nodejs
```

`signed-by=` ata esa clave solo a ese repositorio: es la forma segura, frente al antiguo `apt-key`, que confiaba en la clave para todo.

## 2.3 Detener lo que no vamos a usar todavía

Al instalarse, nginx arranca con su página de bienvenida y MariaDB y PHP-FPM con la configuración por defecto.
Dejamos todo como está hasta configurarlo, pero comprobamos que arrancó:

```bash
systemctl is-active nginx mariadb php8.3-fpm
```

Esperado: tres líneas `active`.

## Comprobación final del paso 2

```bash
php -v | head -1                      # PHP 8.3.x
mariadb --version                     # 10.11.x
node -v                               # v22.x
composer --version | head -1          # Composer 2.7.x
nginx -v                              # nginx/1.24.x
convert -version | head -1            # ImageMagick 6.9.x
gs --version                          # 10.x
free -h | grep -E 'Mem|Swap'          # para ver cuánta memoria queda
```

Si `convert` o `gs` no coinciden con lo esperado, pégame la salida: ILIAS 11 pide ImageMagick 6.9 y Ghostscript 10.
