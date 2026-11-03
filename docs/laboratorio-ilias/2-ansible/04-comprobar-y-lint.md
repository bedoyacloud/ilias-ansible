# A4. Comprobar antes de aplicar: `--check`, `--diff`, tags y lint

Las herramientas con las que se confía en un playbook antes de lanzarlo a un servidor real.

```bash
cd ansible
export ANSIBLE_VAULT_PASSWORD_FILE=$PWD/../.vault-pass

ansible-playbook playbooks/site.yml --syntax-check          # ¿es YAML y Ansible válido?
ansible-lint playbooks roles                                 # buenas prácticas
ansible-playbook playbooks/site.yml --check --diff           # ¿qué cambiaría? (no toca nada)
ansible-playbook playbooks/site.yml                          # aplicar
ansible-playbook playbooks/site.yml --tags php               # solo un rol
```

| Herramienta | Qué responde |
|---|---|
| `--syntax-check` | Si el playbook se puede leer. No se conecta al servidor |
| `ansible-lint` | Si sigue las buenas prácticas (nombres, módulos, permisos). Este proyecto pasa el perfil `production` |
| `--check` | Qué tareas darían `changed`, sin cambiar nada |
| `--diff` | Con `--check`, muestra las líneas que cambiarían en cada fichero |
| `--tags` | Ejecuta solo los roles que indiques. Etiquetas: `base php mariadb ilias nginx maintenance` |

**Un caso real.** `--check` falló porque la consulta que cuenta las tablas no se ejecuta en ese modo y la tarea siguiente
usaba su resultado. La solución es `check_mode: false` en esa tarea, que es un `SELECT` y no cambia nada.

Esto, en el pipeline de GitHub Actions, es el flujo: lint y syntax-check en cada *pull request*, y `--check --diff` antes de aplicar.

## La prueba final

Un servidor nuevo y vacío, un solo comando, ILIAS en marcha: es el paso **A11** del [recorrido](README.md).
Con `ilias-01` ya hemos hecho el equivalente: de Ubuntu recién creado a ILIAS funcionando con una ejecución,
y la segunda con `changed=0`.
