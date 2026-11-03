# A5. Desplegar addons (plugins) de ILIAS

Es lo que Surlabs quiere automatizar: poner un addon en ILIAS de forma repetible. Ejemplo real: **GeoGebraForILIAS**, rama `release_11`.

## A5.1 Cómo se instala un plugin a mano

Según el README del plugin y la documentación de ILIAS 11, son cuatro pasos:

```bash
# 1. Clonar el plugin donde ILIAS lo busca: Customizing/global/plugins/<tipo>/<Nombre>
cd /var/www/ilias/public/Customizing/global/plugins/Services/COPage/PageComponent
sudo -u www-data git clone --branch release_11 https://github.com/surlabs/GeoGebraForILIAS.git GeoGebra

# 2. Regenerar el classmap de composer, para que PHP encuentre las clases nuevas
cd /var/www/ilias && sudo -u www-data composer du

# 3. Instalar y activar el plugin en la base de datos de ILIAS
sudo -u www-data php cli/setup.php install --legacy-plugin GeoGebra -y

# 4. Reiniciar PHP-FPM
sudo systemctl restart php8.3-fpm
```

- La **ruta** depende del tipo de plugin: `Services/COPage/PageComponent` (componente de página), `Services/Repository/RepositoryObject` (objeto de repositorio)... Cada README la indica.
- `--legacy-plugin` es la opción de `setup.php` (existe en ILIAS 11.5, la comprobamos en su código) que instala **y actualiza** un solo plugin. Solo funciona si el plugin declara `$supports_cli_setup = true` en su `plugin.php`; el de GeoGebra lo hace.
- Antes de instalar, la rama debe corresponder a tu versión de ILIAS: `release_11` para ILIAS 11.

## A5.2 El rol `ilias_plugins`

Los plugins son **datos**: una lista en [main.yml](../../../ansible/inventory/group_vars/all/main.yml). El rol la recorre.

```yaml
ilias_plugins_list:
  - name: GeoGebra                  # carpeta, y nombre que usa --legacy-plugin
    id: srgg                        # el $id de su plugin.php
    repo: https://github.com/surlabs/GeoGebraForILIAS.git
    version: release_11             # rama, tag o commit
    slot: Services/COPage/PageComponent
```

Para desplegar otro addon se añade otra entrada y se ejecuta `ansible-playbook playbooks/site.yml --tags plugins`. No se toca ningún rol.

El rol ([tasks/main.yml](../../../ansible/roles/ilias_plugins/tasks/main.yml)) hace los cuatro pasos, y cada uno solo cuando hace falta:

| Tarea | Idempotencia |
|---|---|
| Crear la carpeta del tipo, como `www-data` | `file` con `state: directory` |
| `git` con `version:` | Cambia solo si el repositorio no está o la versión es otra |
| `composer du` | Solo si algún plugin cambió (`when: ...changed`) |
| `setup.php install --legacy-plugin` | Solo si el plugin cambió **o** no figura como activo en la tabla `il_plugin` |
| Reiniciar PHP-FPM | *Handler*: solo si hubo instalación |

Cómo sabe si un plugin ya está instalado: ILIAS guarda cada plugin activo en la tabla `il_plugin` con su `id`. El rol hace un `SELECT`
(con `check_mode: false`, porque no cambia nada) y compara. Igual que con la instalación de ILIAS: se mira el **estado real**, no un fichero "marca".

## A5.3 Comprobación hecha

1. Con el plugin instalado a mano, el rol no hizo nada (`changed=0`).
2. Borré su carpeta y su fila en `il_plugin` y lancé `--tags plugins`: clonó, regeneró el classmap, lo instaló, lo activó y reinició PHP-FPM.
3. Segunda ejecución: `changed=0`. La web siguió respondiendo.

```bash
ansible ilias --become -m command -a 'mariadb ilias -e "SELECT plugin_id, active, last_update_version FROM il_plugin"'
# srgg   1   11.0.2
```

En el navegador: edita una página de un curso y, en el editor, GeoGebra aparece entre los elementos que puedes insertar.

## A5.4 Qué cambia con otros addons de Surlabs

| Addon | Particularidad |
|---|---|
| **GeoGebra**, **JSXGraph**, **SageCell**, **AIPic** | *Page components*; no necesitan nada más. Solo se añade la entrada |
| **Whiteboard**, **LiveVoting** | Objetos de repositorio. Whiteboard necesita además un servidor WebSocket propio (`WhiteboardWebsocketServer`): otro rol |
| **STACK** | Necesita un servidor Maxima (`goemaxima`), con su propio servicio |
| Plugins con configuración | Algunos piden ajustes en la interfaz de administración tras instalarlos; eso queda fuera de este rol |

Para **actualizar** un plugin: cambia `version` y ejecuta el playbook; `git` detecta el cambio y se repite la instalación.
Para **ILIAS mismo** (`ilias_version`), la documentación pide repetir `composer du` y `setup.php update`; es una mejora pendiente.
