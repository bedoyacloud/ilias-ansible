# 5. Código de ILIAS y dependencias

Objetivo: descargar ILIAS 11.5 e instalar sus dependencias PHP y JavaScript. Todavía no se instala la aplicación:
eso es el paso 6.
Todo se ejecuta dentro del servidor.

## 5.1 Estructura de carpetas

```
/var/www/
├── ilias/      código de ILIAS. Su webroot es ilias/public
├── files/      datos subidos por los usuarios (fuera del webroot)
├── config/     ilias.json con la contraseña de la BD (fuera del webroot)
├── logs/       logs de ILIAS y PHP (ya creada en el paso 3)
└── .npm/       caché de npm del usuario www-data
```

Datos y configuración van **fuera** de `ilias/public`: así nginx nunca puede servirlos por error.

```bash
sudo mkdir -p /var/www/{files,config,.npm}
```

## 5.2 Descargar el código

```bash
sudo git clone --depth 1 --branch v11.5 https://github.com/ILIAS-eLearning/ILIAS.git /var/www/ilias
```

- `--branch v11.5`: la versión exacta, sin que la rama cambie debajo de ti.
- `--depth 1`: solo la última versión, sin el historial completo. Es mucho más rápido y ocupa menos disco.

## 5.3 Dueño de los ficheros

PHP corre como `www-data` y ILIAS escribe en su propio árbol (ficheros de plugins, personalización), así que ese usuario debe ser el dueño:

```bash
sudo chown -R www-data:www-data /var/www
```

Desde aquí, los comandos que instalan dependencias se ejecutan **como `www-data`** (`sudo -u www-data`), nunca como root.
Si `composer` o `npm` se ejecutan como root, dejan ficheros con dueño root y luego PHP no puede escribir en ellos.

## 5.4 Dependencias JavaScript

```bash
cd /var/www/ilias
sudo -u www-data npm clean-install --omit=dev --ignore-scripts
```

- `clean-install` (`npm ci`): instala exactamente las versiones del `package-lock.json`, sin cambiarlas.
- `--omit=dev`: salta las herramientas de desarrollo, que producción no necesita.
- `--ignore-scripts`: no ejecuta scripts arbitrarios de los paquetes durante la instalación. Es una medida contra ataques a la cadena de suministro.

Tarda uno o dos minutos. Puede mostrar avisos `npm warn`: no son errores.

## 5.5 Dependencias PHP

```bash
sudo -u www-data composer install --no-dev
```

- `--no-dev`: sin paquetes de desarrollo ni de pruebas.
- Tarda unos minutos y descarga bastante. Si el servidor se queda sin RAM, el swap lo cubre, pero irá más lento.
- Puede avisar de que `composer` se ejecuta con un dueño distinto o de paquetes abandonados: son avisos, no errores.

Mantén la sesión SSH abierta mientras corre. Si se corta a medias, vuelve a ejecutarlo: es seguro repetirlo.

## Comprobación final del paso 5

```bash
ls /var/www/ilias | head -20                       # cli, components, public, vendor, node_modules...
ls -ld /var/www/ilias/vendor /var/www/ilias/node_modules
ls /var/www/ilias/public | head                    # index.php, login.php, ...
ls -ld /var/www /var/www/*                         # todo de www-data
cd /var/www/ilias && sudo -u www-data git describe --tags    # v11.5
du -sh /var/www/ilias                              # tamaño total
df -h / | tail -1                                  # disco libre
free -h | grep -E 'Mem|Swap'
```

Esperado: existen `vendor` y `node_modules`, la versión es `v11.5` y los dueños son `www-data`.
Si `git describe` dice `fatal: detected dubious ownership`, es un aviso de git por el dueño del repositorio: usa
`sudo -u www-data` como en el comando de arriba.
