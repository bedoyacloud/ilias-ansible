# Laboratorio ILIAS 11.5 en AWS EC2

Despliegue de ILIAS (LMS) en una EC2, primero **a mano** para entender cada pieza y después **con Ansible**
para hacerlo reproducible y desplegar también sus addons (plugins).

## Arquitectura

```
Navegador ──HTTP:80──> nginx ──socket──> PHP-FPM (pool ilias) ──socket──> MariaDB (solo localhost)
                         │
                         └── webroot: /var/www/ilias/public   (datos y config fuera del webroot)
```

## Decisiones de la PoC

| Tema | Decisión | En producción sería |
|---|---|---|
| Instancia | t3.small (2 GB) + 2 GB swap, 30 GB gp3 | t3.large o mayor, disco de datos aparte |
| SO | Ubuntu 24.04 LTS (PHP 8.3 nativo) | Igual |
| Acceso | Solo IP y HTTP, `cookie_secure = Off` | Dominio + HTTPS, `cookie_secure = On` |
| BD | `innodb_buffer_pool_size = 256M` | 1G o más |

## Estructura

```
docs/laboratorio-ilias/
├── 1-instalacion-manual/   los 10 pasos a mano (00 a 09) y el resumen para repasar
├── 2-ansible/              el mismo despliegue con Ansible, los plugins, el pipeline y un VPS con HTTPS (A0 a A8)
└── 3-monitorizacion/       Prometheus + Grafana con métricas de todos los servidores
ansible/                    el código de Ansible: inventario, variables, playbook y roles
```

## 1. Instalación manual (terminada)

- [x] 0. Preparar la EC2 — [00-preparar-ec2.md](docs/laboratorio-ilias/1-instalacion-manual/00-preparar-ec2.md)
- [x] 1. Sistema base — [01-sistema-base.md](docs/laboratorio-ilias/1-instalacion-manual/01-sistema-base.md)
- [x] 2. Paquetes — [02-paquetes.md](docs/laboratorio-ilias/1-instalacion-manual/02-paquetes.md)
- [x] 3. PHP y pool FPM — [03-php.md](docs/laboratorio-ilias/1-instalacion-manual/03-php.md)
- [x] 4. MariaDB — [04-mariadb.md](docs/laboratorio-ilias/1-instalacion-manual/04-mariadb.md)
- [x] 5. Código de ILIAS y dependencias — [05-codigo-ilias.md](docs/laboratorio-ilias/1-instalacion-manual/05-codigo-ilias.md)
- [x] 6. `setup.php install` — [06-instalacion-ilias.md](docs/laboratorio-ilias/1-instalacion-manual/06-instalacion-ilias.md)
- [x] 7. nginx — [07-nginx.md](docs/laboratorio-ilias/1-instalacion-manual/07-nginx.md)
- [x] 8. Cron — [08-cron.md](docs/laboratorio-ilias/1-instalacion-manual/08-cron.md)
- [x] 9. Comprobaciones y backup — [09-comprobaciones-y-backup.md](docs/laboratorio-ilias/1-instalacion-manual/09-comprobaciones-y-backup.md)
- [ ] 10. (Pendiente) Dominio y HTTPS

## 2. Ansible (ILIAS desplegado y verificado)

Recorrido completo en [2-ansible/README.md](docs/laboratorio-ilias/2-ansible/README.md).

- [x] A0. Conceptos y entorno — [00-conceptos-y-entorno.md](docs/laboratorio-ilias/2-ansible/00-conceptos-y-entorno.md)
- [x] A1. Proyecto y rol `base` — [01-proyecto-y-rol-base.md](docs/laboratorio-ilias/2-ansible/01-proyecto-y-rol-base.md)
- [x] A2. PHP, MariaDB y Vault — [02-php-mariadb-vault.md](docs/laboratorio-ilias/2-ansible/02-php-mariadb-vault.md)
- [x] A3. ILIAS, nginx y mantenimiento — [03-ilias-nginx-mantenimiento.md](docs/laboratorio-ilias/2-ansible/03-ilias-nginx-mantenimiento.md)
- [x] A4. Comprobar y lint — [04-comprobar-y-lint.md](docs/laboratorio-ilias/2-ansible/04-comprobar-y-lint.md)
- [x] A5. Plugins (addons de Surlabs; ejemplo GeoGebra) — [05-plugins.md](docs/laboratorio-ilias/2-ansible/05-plugins.md)
- [x] A6. GitHub Actions: CI y despliegue en 3 servidores (un addon distinto en cada uno), con plan visible y aprobación — [06-github-actions.md](docs/laboratorio-ilias/2-ansible/06-github-actions.md)
- [x] A7. Prueba desde cero en un servidor nuevo (un solo comando) — [07-prueba-desde-cero.md](docs/laboratorio-ilias/2-ansible/07-prueba-desde-cero.md)
- [x] A9. Backup y restauración, manual y automatizada, con simulacro de desastre — [09-backup-y-restauracion.md](docs/laboratorio-ilias/2-ansible/09-backup-y-restauracion.md)
- [x] A8. VPS compartido con HTTPS (otra versión, secretos propios) — [08-vps-compartido-https.md](docs/laboratorio-ilias/2-ansible/08-vps-compartido-https.md)
- [x] A9. Repo público sin datos personales ni secretos (ver la auditoría en [06-github-actions.md](docs/laboratorio-ilias/2-ansible/06-github-actions.md))

## 3. Monitorización (Prometheus + Grafana)

Guía: [3-monitorizacion/README.md](docs/laboratorio-ilias/3-monitorizacion/README.md).

- [x] Prometheus y Grafana en un VPS, con node_exporter en cada servidor
- [x] Métricas de los servidores de AWS (puerto 9100 abierto solo a la IP de Prometheus)
- [x] Grafana público por dominio, con usuario y contraseña, HTTPS y freno de intentos
