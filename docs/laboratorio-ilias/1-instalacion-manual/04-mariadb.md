# 4. MariaDB

Objetivo: configurar MariaDB como pide ILIAS (utf8mb3), crear la base de datos y un usuario con permisos mínimos.
Todo se ejecuta dentro del servidor.

## 4.1 Configuración (`60-ilias.cnf`)

```bash
sudo tee /etc/mysql/mariadb.conf.d/60-ilias.cnf > /dev/null <<'EOF'
[mysqld]
bind-address = 127.0.0.1
character-set-server = utf8mb3
collation-server = utf8mb3_general_ci
innodb_buffer_pool_size = 256M
join_buffer_size = 256K
table_open_cache = 1000
sql_mode = NO_ENGINE_SUBSTITUTION
skip-name-resolve
EOF
sudo systemctl restart mariadb
```

| Valor | Por qué |
|---|---|
| `bind-address = 127.0.0.1` | MariaDB solo acepta conexiones desde el propio servidor. Nadie de fuera llega al puerto 3306 |
| `utf8mb3` | **ILIAS no soporta `utf8mb4`.** Es el requisito más fácil de olvidar |
| `innodb_buffer_pool_size = 256M` | Caché de datos en RAM. La guía usa 1G, pero aquí compartimos 2 GB con PHP y nginx |
| `sql_mode = NO_ENGINE_SUBSTITUTION` | Modo SQL que ILIAS espera |
| `skip-name-resolve` | No hace búsquedas DNS por cada conexión: más rápido y sin dependencia de DNS |

El prefijo `60-` hace que este fichero se lea después de los de Debian y los pise.

## 4.2 Comprobar que MariaDB aceptó la configuración

```bash
sudo mariadb -e "SHOW VARIABLES WHERE Variable_name IN ('character_set_server','collation_server','innodb_buffer_pool_size','bind_address');"
```

Esperado: `utf8mb3`, `utf8mb3_general_ci`, `268435456` (256 MB) y `127.0.0.1`.

`sudo mariadb` entra como root de MariaDB sin contraseña. Funciona porque el usuario root de la base de datos
se autentica por socket Unix, es decir, solo si eres el root del sistema.

## 4.3 Base de datos y usuario

Genera una contraseña larga y aleatoria, y guárdala en un gestor de contraseñas. La usaremos en el paso 5:

```bash
openssl rand -base64 24 | tr -d '/+=' | cut -c1-32
```

Entra en MariaDB y pega los comandos (cambia `LA_CONTRASEÑA` por la que generaste):

```bash
sudo mariadb
```

```sql
CREATE DATABASE ilias CHARACTER SET utf8mb3 COLLATE utf8mb3_general_ci;
CREATE USER 'ilias'@'localhost' IDENTIFIED BY 'LA_CONTRASEÑA';
GRANT ALL PRIVILEGES ON ilias.* TO 'ilias'@'localhost';
GRANT LOCK TABLES ON *.* TO 'ilias'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

- El usuario `ilias` solo puede conectar desde `localhost` y solo tiene permisos sobre la base `ilias`: el principio de mínimo privilegio.
- `LOCK TABLES` a nivel global lo pide ILIAS para sus operaciones internas.
- No uses el usuario root de MariaDB desde la aplicación.

## 4.4 Comprobar

Prueba que el usuario nuevo puede entrar con su contraseña (te la pide):

```bash
mariadb -u ilias -p -e "SELECT DATABASE(), @@character_set_database; SHOW DATABASES;"
```

Esperado: ves la base `ilias` con `utf8mb3`, y el usuario no ve otras bases de datos de usuario.

Y comprueba que el puerto no está expuesto fuera:

```bash
sudo ss -ltnp | grep 3306
```

Esperado: `127.0.0.1:3306`. Si saliera `0.0.0.0:3306`, `bind-address` no se aplicó.

## 4.5 Seguridad inicial de MariaDB

En Ubuntu 24.04, `mariadb-server` ya viene sin usuarios anónimos y con root por socket. Compruébalo:

```bash
sudo mariadb -e "SELECT user, host, plugin FROM mysql.global_priv;" 2>/dev/null || \
sudo mariadb -e "SELECT user, host FROM mysql.user;"
```

Esperado: `root`, `mariadb.sys` y `mysql` (cuentas internas de MariaDB), más `ilias`, todos en `localhost`.
Lo importante es que no haya ninguna fila con usuario vacío (usuario anónimo) ni hosts como `%`.
