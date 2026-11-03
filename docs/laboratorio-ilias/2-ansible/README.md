# Montaje con Ansible: el mismo despliegue, automatizado

Objetivo: repetir con Ansible lo que hicimos a mano en [1-instalacion-manual](../1-instalacion-manual/),
y entender qué hace cada pieza. El código vive en `ansible/` (en la raíz del repo); aquí están las guías.

## Cómo trabajamos

- Un servidor **nuevo y vacío** (`ilias-01`), para que se vea que Ansible construye todo desde cero. El de la instalación manual se queda como referencia.
- Hay **un rol por responsabilidad** y cada uno equivale a un paso manual. Cada guía explica el rol, el concepto que enseña y cómo comprobarlo.
- La regla de oro: después de cada cambio, ejecutas el playbook **dos veces**. La segunda debe dar `changed=0`. Eso es la idempotencia.

## Recorrido

| Guía | Qué automatiza | Conceptos | Paso manual |
|---|---|---|---|
| [A0. Conceptos y entorno](00-conceptos-y-entorno.md) | Inventario, conexión | Inventario, módulos, *ad-hoc*, idempotencia | 0 |
| [A1. Proyecto y rol `base`](01-proyecto-y-rol-base.md) | apt, swap, UFW, fail2ban | Playbook, roles, tareas, `become`, `creates`, `when` | 1 |
| [A2. PHP, MariaDB y Vault](02-php-mariadb-vault.md) | PHP-FPM, base de datos y usuario, secretos | Plantillas Jinja2, *handlers*, `flush_handlers`, Vault | 2, 3, 4 |
| [A3. ILIAS, nginx y mantenimiento](03-ilias-nginx-mantenimiento.md) | Código, `setup.php`, nginx, cron, backups | `git`, `become_user`, instalar solo si hace falta, `nginx -t` | 5 a 9 |
| [A4. Comprobar y lint](04-comprobar-y-lint.md) | — | `--check --diff`, tags, `ansible-lint` | — |
| [A5. Plugins](05-plugins.md) | Addons de ILIAS (GeoGebra) | Datos en listas, bucles, estado real desde la BD | — |

## Siguiente

| Etapa | Qué | Para qué |
|---|---|---|
| **A6** | [GitHub Actions](06-github-actions.md): CI (lint, syntax-check) y deploy manual | Pipeline |
| **A7** | [Prueba desde cero](07-prueba-desde-cero.md) en un servidor nuevo | Demostrar la reproducibilidad |
| **A8** | Publicar en GitHub: limpiar datos propios, licencia, README de inicio rápido | Repo reutilizable |

## Estructura del código

```
ansible/
├── ansible.cfg
├── requirements.yml
├── inventory/
│   ├── hosts.example.yml        (hosts.yml es el real y no se sube)
│   └── group_vars/all/{main.yml,vault.example.yml}
├── playbooks/site.yml
└── roles/{base,php,mariadb,ilias,nginx,ilias_plugins,ilias_maintenance}
```

## Qué no hacemos (todavía)

HTTPS, monitorización y CI. Se añaden después, cuando el despliegue base sea reproducible.
