# A1. Un playbook, un rol y el sistema base

Equivale al [paso 1 manual](../1-instalacion-manual/01-sistema-base.md). Aquí aparecen los conceptos que sostienen todo lo demás.

## A1.1 La estructura

```
ansible/
├── ansible.cfg                       dónde está el inventario y los roles
├── requirements.yml                  colecciones que necesita el proyecto
├── inventory/
│   ├── hosts.yml                     tu servidor (no se sube a GitHub)
│   └── group_vars/all/
│       ├── main.yml                  variables comunes (sí se sube)
│       └── vault.yml                 secretos cifrados (no se sube)
├── playbooks/site.yml                qué roles se aplican y en qué orden
└── roles/<nombre>/{tasks,handlers,templates}/
```

- **Playbook** (`site.yml`): "a los servidores del grupo `ilias`, aplica estos roles en este orden". Es corto a propósito.
- **Rol**: una carpeta con todo lo de una responsabilidad (tareas, plantillas, handlers). Es la unidad reutilizable.
  El playbook solo los ordena. Por eso, para otro servidor, se cambia el inventario y no los roles.
- **Variables**: `group_vars/all/main.yml`. Todo lo que cambia de un entorno a otro (versión, dominio, tamaños) está ahí.

Los roles de este proyecto, y el paso manual que automatizan:

| Rol | Paso manual |
|---|---|
| `base` | 1. Sistema base |
| `php` | 2 y 3. Paquetes de PHP y pool FPM |
| `mariadb` | 4. MariaDB |
| `ilias` | 5 y 6. Código e instalación |
| `nginx` | 7. nginx |
| `ilias_maintenance` | 8 y 9. Cron, logrotate y backups |

Para ILIAS esto se ejecuta con una orden:

```bash
cd ansible
export ANSIBLE_VAULT_PASSWORD_FILE=$PWD/../.vault-pass     # ver A2.3
ansible-playbook playbooks/site.yml
```

## A1.2 El rol `base` ([roles/base/tasks/main.yml](../../../ansible/roles/base/tasks/main.yml))

Cada tarea es una llamada a un módulo. Los que usa:

| Tarea | Módulo | Lo que hace |
|---|---|---|
| Actualizar el sistema | `ansible.builtin.apt` con `upgrade: dist` | `apt full-upgrade`, y `cache_valid_time` evita actualizar el índice si es reciente |
| Zona horaria | `community.general.timezone` | UTC |
| Swap | `command` con `creates`, `file`, `mount`, `sysctl` | Crea, protege (0600), activa y persiste el swap |
| Firewall | `community.general.ufw` | Entrada denegada por defecto; permite SSH y 80 **antes** de activarlo |
| fail2ban, unattended-upgrades | `apt` + `service` | Instalados, arrancados y habilitados |

Dos ideas importantes:

**1. Los comandos "de una sola vez" se hacen idempotentes con `creates` o `when`.**
`fallocate -l 2G /swapfile` no es idempotente por sí solo. Con `args: creates: /swapfile`, Ansible lo salta si el fichero ya existe.
`mkswap` y `swapon` solo se ejecutan si el hecho (*fact*) `ansible_facts.swaptotal_mb` dice que no hay swap.
Si existe un módulo para lo que quieres (`apt`, `ufw`, `sysctl`, `mount`), úsalo: ya es idempotente.

**2. El orden del firewall.** La tarea que permite SSH va antes de la que activa UFW. Es el mismo cuidado del paso manual:
al revés, te quedas fuera del servidor. Las reglas salen de la variable `ssh_allowed_ports`, que se recorre con `loop`.

## A1.3 Ejecutar y comprobar

```bash
ansible-playbook playbooks/site.yml --tags base      # solo este rol
ansible-playbook playbooks/site.yml --tags base      # otra vez: changed=0
```

La primera ejecución da `changed=N`; la segunda, `changed=0`. Si la segunda cambia algo, la tarea no es idempotente: hay que corregirla.
`--tags` ejecuta solo los roles etiquetados en `site.yml`.
