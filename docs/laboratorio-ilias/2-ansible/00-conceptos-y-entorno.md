# A0. Conceptos y entorno

Objetivo: entender qué es Ansible, montar el proyecto, y comprobar con comandos sueltos (*ad-hoc*) los dos conceptos
que sostienen todo lo demás: **módulos** e **idempotencia**. Aún no escribimos ningún playbook.

## A0.1 Qué es Ansible

- **Sin agente:** no instalas nada en el servidor. Ansible entra por SSH, igual que tú, ejecuta una pequeña pieza de Python y se va.
- **Declarativo:** no dices "ejecuta `apt install nginx`", dices "nginx debe estar instalado". Si ya lo está, no hace nada.
- **Idempotente:** puedes ejecutarlo cien veces y el resultado es el mismo. La primera vez cambia cosas; las siguientes, nada.

Esto es lo que nos faltaba al hacerlo a mano: tus comandos del paso 1 al 9 no se pueden repetir sin pensar
(`ufw enable` o un `CREATE DATABASE` fallan o duplican si ya existen).

## A0.2 Vocabulario

| Término | Qué es | Ejemplo de nuestra instalación manual |
|---|---|---|
| **Inventario** | Lista de servidores y sus grupos | Nuestro `inventario.ini` |
| **Módulo** | Una unidad de trabajo ya hecha: `apt`, `copy`, `template`, `service`, `ufw`... | `apt install nginx` → módulo `apt` |
| **Tarea** (*task*) | Una llamada a un módulo con sus parámetros | "Instalar nginx" |
| **Play** | Un grupo de tareas aplicado a unos servidores | "Todo lo del grupo `ilias`" |
| **Playbook** | Un fichero YAML con uno o más plays | `site.yml` |
| **Variable** | Un valor reutilizable | `ilias_version: v11.5` |
| **Plantilla** (*template*) | Un fichero con huecos (Jinja2) que se rellenan con variables | `99-ilias.ini`, `ilias.json` |
| **Handler** | Una acción que solo se ejecuta si algo cambió | Recargar nginx solo si cambió su configuración |
| **Rol** | Un conjunto de tareas, plantillas y variables reutilizable | `nginx_ilias`, `mariadb` |
| **Vault** | Cifrado de secretos dentro del repo | La contraseña de la BD |
| **Facts** | Datos que Ansible descubre del servidor (SO, RAM, IPs) | Que es Ubuntu 24.04 |

## A0.3 El servidor de Ansible

Trabajamos sobre un servidor **nuevo y vacío**, para ver cómo Ansible lo construye todo.
Repite el [paso 0 de la instalación manual](../1-instalacion-manual/00-preparar-ec2.md) con estos cambios:

| | Valor |
|---|---|
| Nombre | `ilias-ansible` |
| AMI | Ubuntu Server **24.04** LTS |
| Tipo | `t3.small` |
| Disco | 30 GiB gp3 |
| Par de claves | `mi-llave` (la misma) |
| Security group | `ilias-sg` (el mismo: 22 y 80 desde tu IP) |

Cuando la instancia esté en *Running*, guarda su IP pública en una variable de la terminal del Mac.
La usaremos en varios comandos de esta etapa:

```bash
IP_NUEVA=x.x.x.x        # cambia x.x.x.x por la IP pública de ilias-ansible
echo "$IP_NUEVA"
```

La variable solo existe en esa terminal. Si abres otra, vuelve a definirla.

**No instales nada en el servidor.** Entra una vez por SSH solo para aceptar la huella (responde `yes`):

```bash
ssh -i ~/.ssh/mi-llave.pem ubuntu@"$IP_NUEVA"
exit
```

Si el servidor de la instalación manual está encendido, puedes detenerlo: dos instancias a la vez gastan el doble.

## A0.4 El proyecto

Todo lo de esta sección se hace **en tu Mac**, en la misma terminal donde definiste `IP_NUEVA`. Al terminar tendrás:

```
ansible/
├── ansible.cfg              configuración de Ansible para este proyecto
├── inventory/
│   ├── hosts.yml            tu IP y tu llave (no se sube a GitHub)
│   └── hosts.example.yml    la plantilla con valores de ejemplo (sí se sube)
└── playbooks/               vacía hasta A1
```

Los ficheros se crean con un *heredoc*: `cat > fichero <<'EOF'` escribe en `fichero` todo lo que pegues
hasta la línea `EOF`. Copia **el bloque entero**, desde `cat` hasta `EOF`, y pégalo en la terminal.
Si lo prefieres, puedes crear los mismos ficheros desde VS Code con el mismo contenido.

### Paso 1. Crear las carpetas y entrar en `ansible/`

```bash
cd ruta/a/ilias-ansible
mkdir -p ansible/inventory ansible/playbooks
cd ansible
pwd                      # debe terminar en /taller-ansible/ansible
```

A partir de aquí, **todos los comandos de Ansible se ejecutan desde `ansible/`**. El motivo está en el paso 2.

### Paso 2. Crear `ansible.cfg`

```bash
cat > ansible.cfg <<'EOF'
[defaults]
inventory = inventory/hosts.yml
host_key_checking = True
interpreter_python = auto_silent
callback_result_format = yaml
retry_files_enabled = False

[ssh_connection]
pipelining = True
EOF
```

Qué hace cada línea:

- `inventory`: así no hace falta `-i inventory/hosts.yml` en cada comando.
- `host_key_checking = True`: verifica la huella del servidor (la seguridad de SSH). **No lo desactives.**
- `interpreter_python = auto_silent`: busca Python 3 en el servidor sin mostrar avisos.
- `callback_result_format = yaml`: salidas en YAML, más fáciles de leer que JSON. Ojo con el nombre:
  si escribes `result_format`, Ansible lo ignora **sin avisar**.
- `retry_files_enabled = False`: no crea ficheros `.retry` cuando un playbook falla.
- `pipelining = True`: menos conexiones SSH por tarea, así va más rápido.

Comprueba que Ansible lo está usando:

```bash
ansible --version | head -2
```

Esperado: `config file = .../taller-ansible/ansible/ansible.cfg`. Si dice `config file = None`, hay dos causas posibles:
no estás en `ansible/` (compruébalo con `pwd`), o el fichero no se creó (compruébalo con `ls`: debe aparecer `ansible.cfg`).

Ansible busca su configuración en este orden y usa **solo la primera** que encuentra:

1. La variable de entorno `ANSIBLE_CONFIG`.
2. `ansible.cfg` en el **directorio actual**.
3. `~/.ansible.cfg`.
4. `/etc/ansible/ansible.cfg`.

Por eso hay que trabajar desde `ansible/`: si no, Ansible no encuentra el `ansible.cfg` ni el inventario.

### Paso 3. Crear `inventory/hosts.yml`

Antes usábamos formato INI (`inventario.ini`). Pasamos a YAML porque muestra los grupos y las variables con más claridad.

Este bloque usa la variable `IP_NUEVA` de A0.3. Aquí `EOF` va **sin comillas**, a diferencia del paso 2,
para que la terminal sustituya `$IP_NUEVA` por la IP:

```bash
cat > inventory/hosts.yml <<EOF
all:
  children:
    ilias:
      hosts:
        ilias-01:
          ansible_host: $IP_NUEVA
      vars:
        ansible_user: ubuntu
        ansible_ssh_private_key_file: ~/.ssh/mi-llave.pem
EOF
cat inventory/hosts.yml
```

Comprueba que la IP quedó escrita. Hay dos formas: mirar el fichero, o preguntarle a Ansible qué variables ve para el host.

```bash
grep ansible_host inventory/hosts.yml
ansible-inventory --host ilias-01
```

Esperado (con tu IP):

```
          ansible_host: 203.0.113.10
{
    "ansible_host": "203.0.113.10",
    "ansible_ssh_private_key_file": "~/.ssh/mi-llave.pem",
    "ansible_user": "ubuntu"
}
```

La segunda es la mejor prueba: confirma que Ansible lee el inventario y que el host recibe las variables del grupo.

- **Si `ansible_host` sale vacío**, la variable no estaba definida en esta terminal. Defínela como en A0.3 y repite el bloque.
- **El `~` de la llave sale tal cual, sin cambiar por tu ruta.** Es correcto: Ansible lo resuelve al conectar.

- `ilias-01` es un **nombre** que tú eliges; `ansible_host` es la IP real.
- Está dentro del grupo `ilias`, que usarán todos los playbooks.
- `ansible_user` y la llave están en `vars` del grupo, así que se aplican a todos sus servidores.

**Por qué `ilias-01` y no `ilias-aws`.** Los playbooks apuntarán al **grupo** (`hosts: ilias`), nunca al nombre del host,
así que el nombre no aparece en el código. Aun así conviene que sea genérico:

- **No ata el repo a un proveedor.** Lo mismo funciona en AWS, en otra nube o en una máquina virtual local con Ubuntu 24.04.
- **Escala.** Un segundo servidor sería `ilias-02`, sin renombrar nada.
- **Es la convención habitual:** función más número. El proveedor o el entorno (`poc`, `prod`) se expresan con grupos
  o con inventarios separados, no con el nombre del host.

### Paso 4. Crear `inventory/hosts.example.yml`

`hosts.yml` tiene tu IP y la ruta de tu llave: está en `.gitignore` y **no se sube** a GitHub.
Lo que se sube es una copia con valores de ejemplo, para que otra persona sepa qué rellenar.
`sed` la genera a partir de `hosts.yml`, cambiando esas dos líneas:

```bash
sed -e 's/ansible_host: .*/ansible_host: IP_DEL_SERVIDOR/' \
    -e 's#ansible_ssh_private_key_file: .*#ansible_ssh_private_key_file: ~/.ssh/TU_LLAVE.pem#' \
    inventory/hosts.yml > inventory/hosts.example.yml
cat inventory/hosts.example.yml
```

Debe mostrar `IP_DEL_SERVIDOR` y `TU_LLAVE.pem`, no tus datos.

Quien clone el repo hará `cp inventory/hosts.example.yml inventory/hosts.yml` y pondrá sus datos.
Este patrón, **el real ignorado y el `.example` versionado**, lo repetiremos con los secretos en A4.

### Paso 5. Comprobar lo creado

```bash
find . -type f | sort
```

Esperado:

```
./ansible.cfg
./inventory/hosts.example.yml
./inventory/hosts.yml
```

`playbooks/` no aparece porque todavía está vacía.

## A0.5 Comprobar el inventario y la conexión

Siempre desde `ansible/`:

```bash
ansible-inventory --graph              # ¿ve el grupo y el host?
ansible ilias -m ping                  # ¿entra por SSH?
```

Esperado: `ilias-01 | SUCCESS` con `"ping": "pong"`. Si falla, es lo mismo que en el taller: IP, llave, `chmod 400`,
o el security group.

## A0.6 Comandos *ad-hoc*: módulos en acción

Un comando *ad-hoc* es un módulo suelto, sin playbook: `ansible <grupo> -m <módulo> -a "<argumentos>"`.

**1. Descubrir el servidor (facts):**

```bash
ansible ilias -m setup -a "filter=ansible_distribution*"
ansible ilias -m setup -a "filter=ansible_memtotal_mb"
```

Ansible ya sabe que es Ubuntu 24.04 y cuánta RAM tiene: lo que decidimos a mano en la instalación, aquí es un dato leído.

**2. Un comando simple:**

```bash
ansible ilias -m command -a "uname -a"
```

**3. La idempotencia, en dos ejecuciones.** Instala `htop` con el módulo `apt`. Necesita `--become` (equivale a `sudo`):

```bash
ansible ilias -m apt -a "name=htop state=present update_cache=yes" --become
ansible ilias -m apt -a "name=htop state=present" --become
```

- Si la primera ya sale `"changed": false`, es que la imagen de Ubuntu traía `htop`. Bórralo con
  `ansible ilias -m apt -a "name=htop state=absent" --become` y repite las dos ejecuciones.
- La primera sale en **amarillo**, con `"changed": true`: lo instaló.
- La segunda sale en **verde**, con `"changed": false`: ya estaba, no hizo nada.

Eso es la idempotencia. `state=present` significa "debe existir", no "instálalo".

**4. Modo prueba (`--check`)**: dice qué haría, sin hacerlo:

```bash
ansible ilias -m apt -a "name=tree state=present" --become --check
```

Dice `changed: true` pero `tree` **no** se instala. Verifícalo:

```bash
ansible ilias -m command -a "which tree"
```

Debe fallar (`rc=1`): confirma que `--check` no tocó nada.

**5. Limpiar** lo que instalamos de prueba:

```bash
ansible ilias -m apt -a "name=htop state=absent" --become
```

## A0.7 Qué has aprendido

| Colores de la salida | Significado |
|---|---|
| Verde (`ok`) | Ya estaba como lo pedías: no se tocó |
| Amarillo (`changed`) | Se modificó algo |
| Rojo (`failed`) | Error |
| Cian (`skipping`) | Tarea omitida por una condición |

## Comprobación final de A0

Pégame:
1. Las dos primeras líneas de `ansible --version` (la de `config file`).
2. La salida de `ansible-inventory --graph`.
3. La salida del `ping`.
4. Las dos ejecuciones de `htop`: la primera `changed: true` y la segunda `changed: false`.
5. La comprobación de `which tree`.

Y una pregunta para ti, que no hace falta responder aquí: ¿qué crees que pasaría si ejecutaras dos veces
un `command: apt install htop`, en lugar de usar el módulo `apt`? Lo vemos en A1.
