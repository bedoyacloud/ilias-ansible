# 6. Instalar ILIAS (`setup.php install`)

Objetivo: crear el fichero de configuración e instalar la aplicación: `setup.php` crea las tablas en MariaDB y
las carpetas de datos. Todavía no se puede abrir en el navegador: falta nginx (paso 7).
Todo se ejecuta dentro del servidor.

## 6.0 Antes de empezar: cierra el puerto 80 al mundo

Al terminar este paso, ILIAS existe con el usuario `root` y la contraseña **`homer`**, que es pública
(está en la documentación). Entre que nginx publica la web (paso 7) y que cambias esa contraseña pasan unos minutos.

Para que en ese rato nadie pueda entrar: EC2 → Grupos de seguridad → `ilias-sg` → Reglas de entrada →
HTTP 80: cambia el origen de *Anywhere* a **Mi IP**. Cuando cambies la contraseña de `root` (paso 7.5), vuelves a
poner *Anywhere IPv4*.

## 6.1 Fichero de configuración

`ilias.json` contiene la contraseña de la base de datos, así que vive en `/var/www/config` (fuera del webroot)
y solo lo lee `www-data`. Parte de la plantilla que trae ILIAS:

```bash
sudo -u www-data cp /var/www/ilias/components/ILIAS/setup_/minimal-config.json /var/www/config/ilias.json
sudo chmod 600 /var/www/config/ilias.json
sudo -u www-data nano /var/www/config/ilias.json
```

Sustituye todo el contenido por esto, cambiando lo que está en mayúsculas:

```json
{
  "common": { "client_id": "pocilias" },
  "database": {
    "user": "ilias",
    "password": "LA_CONTRASEÑA_DE_LA_BD",
    "database": "ilias",
    "create_database": false
  },
  "filesystem": { "data_dir": "/var/www/files" },
  "logging": {
    "enable": true,
    "path_to_logfile": "/var/www/logs/ilias.log",
    "errorlog_dir": "/var/www/logs"
  },
  "http": { "path": "http://IP_DEL_SERVIDOR" },
  "systemfolder": {
    "contact": {
      "firstname": "TU_NOMBRE",
      "lastname": "TU_APELLIDO",
      "email": "tu@correo.com"
    }
  },
  "utilities": { "path_to_convert": "/usr/bin/convert" },
  "preview": { "path_to_ghostscript": "/usr/bin/gs" }
}
```

Para guardar en nano: `Ctrl+O`, Enter, `Ctrl+X`.

| Campo | Qué es |
|---|---|
| `client_id` | Nombre interno de la instalación. **No se puede cambiar después**, y nombra la carpeta de datos |
| `database.password` | La contraseña que generaste en el paso 4.3 |
| `create_database: false` | La base `ilias` ya la creamos nosotros, con utf8mb3 |
| `data_dir` | Dónde guarda ILIAS los ficheros subidos: fuera del webroot |
| `http.path` | La URL con la que el navegador abrirá ILIAS. **Debe coincidir exactamente** (aquí, la IP elástica y HTTP) |
| `path_to_convert`, `path_to_ghostscript` | Rutas de ImageMagick y Ghostscript, para miniaturas y previsualización de PDF |

Si más adelante cambias de IP, hay que cambiar `http.path` y volver a ejecutar `setup.php update`.
Con un dominio (paso 10) esto desaparece.

## 6.2 Validar el fichero

El fichero es `-rw-------` y de `www-data`: tu usuario `ubuntu` no puede leerlo (es lo que queremos).
Por eso la validación se ejecuta como `www-data`:

```bash
sudo -u www-data python3 -m json.tool /var/www/config/ilias.json > /dev/null && echo "JSON válido"
ls -l /var/www/config/ilias.json          # -rw------- www-data www-data
```

Si `ls` muestra otro dueño: `sudo chown www-data:www-data /var/www/config/ilias.json`.
Si el JSON es inválido, el error indica la línea; suele ser una coma de más o de menos, o comillas tipográficas.

## 6.3 Instalar

```bash
cd /var/www/ilias
sudo -u www-data php cli/setup.php install /var/www/config/ilias.json
```

- Si pregunta algo parecido a "¿continuar?", responde `y`.
- Tarda varios minutos: crea más de 1000 tablas, carga los idiomas y construye los ficheros de arranque.
- No cierres la sesión SSH. Si se interrumpe, no lo repitas sin más: pégame primero la salida.

## 6.4 Comprobar

**Cuidado:** `setup.php status` imprime toda la configuración al final, **incluida la contraseña de la base de datos**.
Filtra siempre su salida, y no la pegues entera en chats ni tickets:

```bash
sudo -u www-data php cli/setup.php status | grep -B2 'update_required: true'   # sin salida = todo al día
ls /var/www/files                         # una carpeta llamada pocilias
mariadb -u ilias -p ilias -e "SELECT COUNT(*) AS tablas FROM information_schema.tables WHERE table_schema='ilias';"
sudo tail -5 /var/www/logs/ilias.log      # no existe hasta que ILIAS escribe su primer mensaje
```

Esperado: ningún `update_required: true`, existe `/var/www/files/pocilias` y hay unas mil tablas o más.

Si aparecen pasos pendientes, aplícalos con `sudo -u www-data php cli/setup.php update`.

### Dos avisos que `status` mostrará siempre en una instalación nueva de v11.5

Tras `install` y tras `update`, `status` sigue marcando estos dos como `update_required: true`. Lo comprobamos en el servidor:

| Aviso | Qué pasa | Efecto |
|---|---|---|
| `ilBackgroundTasksDB80` (versión 0, disponible 1) | El agente de BackgroundTasks, en su `getUpdateObjective`, no ejecuta sus pasos de BD (solo guarda la configuración), pero `getStatusObjective` sí los cuenta. Nunca se aplica | Falta un índice en `il_bt_bucket(user_id)`: solo rendimiento |
| `Test10DBUpdateSteps` (versión 17, disponible 13) | La BD base que carga `install` viene con pasos 15 a 17 ya registrados (de marzo y abril de 2026) y el código de la v11.5 solo tiene 13. `update` solo ejecuta pasos que faltan, no tiene nada que hacer | No lo hemos podido determinar; no hay errores |

Es una inconsistencia del propio ILIAS, no de nuestra instalación. Si dejaran de aparecer o aparecieran otros, es señal para mirarlo.
Lo que sí debe ser cierto: `update` termina con `Update complete` y no hay errores.

## 6.4b Dónde queda guardada la contraseña de la BD

ILIAS copia la contraseña en un segundo fichero: `/var/www/ilias/public/data/pocilias/client.ini.php`.
Lo crea con permisos `-rw-rw-r--` (legible por cualquiera del sistema). Ciérralo:

```bash
sudo chmod 600 /var/www/ilias/public/data/pocilias/client.ini.php
```

Si alguna vez cambias la contraseña de MariaDB, hay que cambiarla en **tres sitios**: MariaDB (`ALTER USER`),
`/var/www/config/ilias.json` y ese `client.ini.php`.

## 6.5 La contraseña de `root`

El usuario `root` de **ILIAS** (no el del sistema ni el de MariaDB) nace con `homer`. La cambiaremos en cuanto
podamos entrar por el navegador, en el paso 7.5. Hasta entonces, el puerto 80 sigue cerrado a tu IP (6.0).
