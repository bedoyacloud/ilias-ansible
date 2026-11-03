# Monitorización: Prometheus + Grafana

Un único Grafana que muestra las métricas (CPU, memoria, disco, red, carga) de **todos** los servidores del laboratorio,
estén en AWS o en un VPS de otro proveedor. Se despliega con los mismos playbooks que ILIAS.

## 1. Arquitectura

```
                       servidor de monitorización (VPS)
                   ┌──────────────────────────────────────┐
  tu Mac ──túnel──>│ Grafana 127.0.0.1:3000               │
     SSH           │      │                               │
                   │      ▼                               │
                   │ Prometheus 127.0.0.1:9090 ───────────┼──> node_exporter :9100  (servidor 1, AWS)
                   │      │                               │
                   │      └─ node_exporter 127.0.0.1:9100 │──> node_exporter :9100  (servidor 2, AWS)
                   └──────────────────────────────────────┘
```

- **node_exporter** corre en cada servidor y publica sus métricas en `:9100`. Es solo un lector: no guarda nada.
- **Prometheus** *consulta* (*pull*) cada 30 s a todos los node_exporter y guarda las series 15 días.
- **Grafana** dibuja lo que Prometheus tiene. Ya trae el dashboard *Node Exporter Full* (id 1860 de grafana.com).

## 2. Decisiones

| Decisión | Por qué |
|---|---|
| Prometheus **consulta** a los servidores (modelo *pull*) en vez de que ellos envíen datos | Es el modelo normal de Prometheus: un solo sitio sabe qué servidores existen (el inventario), y los servidores no necesitan credenciales para enviar |
| Grafana y Prometheus escuchan solo en `127.0.0.1` | No se abre ningún puerto público nuevo en un servidor que ya está en producción. Se entra con un túnel SSH |
| `network_mode: host` en los contenedores | Prometheus alcanza `127.0.0.1:9100` y las IPs externas sin configurar redes de Docker |
| node_exporter del repositorio de Ubuntu (`prometheus-node-exporter`) | Se actualiza con el resto del sistema y no hay binarios que descargar |
| node_exporter solo en `127.0.0.1` por defecto | Sus métricas no llevan autenticación. En los servidores remotos se abre **solo a la IP de Prometheus** |
| Imágenes con la versión fijada y dashboard con la revisión fijada | El despliegue es reproducible: no cambia solo porque salga una versión nueva |
| La contraseña de Grafana va en el Vault y en un fichero `0600` | No aparece en el repo ni en la salida de Ansible (`no_log`) |

## 3. Qué se añadió al código

| Elemento | Qué es |
|---|---|
| Rol [`node_exporter`](../../../ansible/roles/node_exporter/tasks/main.yml) | Instala el paquete, fija la dirección de escucha y, si hace falta, permite el puerto 9100 en UFW **solo desde las IPs indicadas** |
| Rol [`monitoring`](../../../ansible/roles/monitoring/tasks/main.yml) | Prometheus y Grafana con Docker Compose, el datasource y el dashboard ya provisionados |
| Segundo *play* de [`site.yml`](../../../ansible/playbooks/site.yml) | Aplica `monitoring` solo al grupo `monitoring` del inventario (opcional) |
| Etiqueta `metrics` / `monitoring` | `--tags metrics` instala node_exporter; `--tags monitoring` levanta Prometheus y Grafana |
| Variables | `node_exporter_listen`, `node_exporter_allow_from`, `node_exporter_port`, `monitoring_*` |

Los **objetivos de Prometheus salen del inventario**: la plantilla recorre el grupo `ilias`, así que añadir un servidor al inventario
lo añade a las métricas sin tocar nada más.

## 4. Configuración (inventario)

En `hosts.yml` (que no se sube). El servidor de monitorización va en el grupo `monitoring`, y los servidores remotos abren
node_exporter solo a su IP:

```yaml
ilias:
  hosts:
    ilias-01:                                   # servidor remoto
      ansible_host: IP_DEL_SERVIDOR_1
      node_exporter_listen: "0.0.0.0:9100"
      node_exporter_allow_from: ["IP_DEL_SERVIDOR_DE_MONITORIZACION"]
    ilias-vps:                                  # el servidor de monitorización
      ansible_host: IP_DEL_SERVIDOR_DE_MONITORIZACION
      # node_exporter por defecto: 127.0.0.1 (lo consulta Prometheus desde el mismo servidor)
monitoring:
  hosts:
    ilias-vps:
```

Y la contraseña del administrador, cifrada, en `inventory/host_vars/ilias-vps/vault.yml`:

```yaml
vault_grafana_admin_password: UNA_CONTRASENA_LARGA
```

> **Firewall de la nube.** Poner `node_exporter_allow_from` configura UFW **dentro** del servidor. Si el servidor está en AWS, el
> *security group* debe permitir también el puerto 9100 desde la IP de Prometheus (regla de entrada TCP 9100, origen `IP/32`).
> Sin esa regla, Prometheus marca el servidor como `DOWN` con `context deadline exceeded`.

## 5. Desplegar

```bash
cd ansible
export ANSIBLE_VAULT_PASSWORD_FILE=$PWD/../.vault-pass
ansible-playbook playbooks/site.yml --tags metrics --limit ilias           # node_exporter en todos
ansible-playbook playbooks/site.yml --tags monitoring --limit monitoring   # Prometheus + Grafana
ansible-playbook playbooks/site.yml --tags monitoring --limit monitoring   # 2.ª vez: changed=0
```

## 6. Entrar en Grafana

Grafana solo escucha en `localhost` del servidor, así que se abre un túnel SSH desde tu Mac:

```bash
ssh -L 3000:127.0.0.1:3000 TU_USUARIO@IP_DEL_SERVIDOR_DE_MONITORIZACION
```

Con el túnel abierto, en el navegador: **http://localhost:3000**. Usuario `admin` y la contraseña del Vault:

```bash
ansible-vault view inventory/host_vars/ilias-vps/vault.yml
```

Dashboard: **Dashboards → Servidores → Node Exporter Full**, y el selector *Host* para elegir el servidor.

## 7. Comprobar

Desde el servidor de monitorización:

```bash
# ¿Qué servidores consulta Prometheus y cómo están? (health = up/down)
curl -s localhost:9090/api/v1/targets | python3 -c "import sys,json; [print(t['labels']['instance'], t['health']) for t in json.load(sys.stdin)['data']['activeTargets']]"

# La misma consulta a través de Grafana (comprueba datasource + credenciales)
curl -s -u admin:CONTRASENA 'localhost:3000/api/datasources/proxy/uid/prometheus/api/v1/query?query=up'
```

En el propio Prometheus, `up{job="node"}` vale `1` por servidor si todo va bien.

## 8. Resultados de la validación

| Comprobación | Resultado |
|---|---|
| Prometheus y Grafana, 1.ª ejecución | `failed=0`; ver el apartado 9 (un fallo corregido en el camino) |
| Contenedores | `prometheus` y `grafana` en marcha, junto al contenedor que ya existía |
| Puertos de Prometheus y Grafana | Solo `127.0.0.1:9090` y `127.0.0.1:3000` |
| Grafana | `/api/health` 200, base de datos `ok` |
| Login con la contraseña del Vault / con `admin/admin` | `200` / `401` |
| Datasource y dashboard | Provisionados: *Prometheus* y *Node Exporter Full* |
| Consulta Grafana → Prometheus | `200`, con datos reales |
| Servidor de monitorización (consulta local) | `up=1` |
| Servidores remotos de AWS | `up=0` hasta abrir el puerto 9100 en su *security group* (ver apartado 4) |

## 9. Problemas reales encontrados

| Síntoma | Causa | Solución |
|---|---|---|
| `'node_exporter_port' is undefined` | El valor era un `defaults` del rol `node_exporter`, y `monitoring` corre en **otro play** donde ese rol no se carga | Las variables que comparten dos roles van en `group_vars`, no en los `defaults` de uno de ellos |
| Servidores remotos `DOWN` con `context deadline exceeded` | El *security group* de AWS filtra el 9100 aunque UFW lo permita: hay dos firewalls | Abrir el 9100 en el security group solo para la IP de Prometheus. Un *timeout* indica un firewall; una conexión rechazada indicaría que nada escucha |
| Todos los objetivos `UNKNOWN` justo tras desplegar | Prometheus aún no había hecho la primera consulta | Esperar unos 30 s |
| Grafana sin responder 20 s después de arrancar | Está creando su base de datos | Esperar a que `/api/health` dé 200 |
| Cambiar `vault_grafana_admin_password` no cambia la contraseña | Grafana solo lee `GF_SECURITY_ADMIN_PASSWORD` al crear su base de datos | `docker exec grafana grafana cli admin reset-admin-password NUEVA` |

## 10. Qué falta (y por qué)

- **Alertas.** Hoy solo se *ven* las métricas. El siguiente paso son reglas de alerta (disco al 85 %, servidor caído) con Alertmanager o con las alertas de Grafana.
- **Métricas de ILIAS.** node_exporter mide el servidor, no la aplicación. Para PHP-FPM, nginx o MariaDB harían falta sus exporters.
- **Acceso por dominio.** El túnel SSH es lo más seguro para empezar. Con un subdominio y HTTPS (por ejemplo vía Cloudflare), el acceso sería más cómodo, pero expone un login a internet.
- **Servidores sin IP fija.** Las IPs públicas de AWS cambian si la instancia se detiene y se arranca. En ese caso hay que usar una IP elástica o una VPN (WireGuard, Tailscale) entre los servidores.
