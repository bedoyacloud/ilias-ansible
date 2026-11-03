# 8. Cron de ILIAS

Objetivo: que ILIAS ejecute sus tareas programadas cada 5 minutos. Sin esto, la web "funciona", pero no se envían
correos, no se limpian sesiones ni temporales, no se actualizan indicadores y fallan notificaciones y sincronizaciones.
En producción es obligatorio.

## Cómo funciona

```
cron del sistema ─ cada 5 min ─> php cli/cron.php run-jobs cron-user pocilias
                                       │
                                       └─ entra en ILIAS como "cron-user" y ejecuta los jobs que tocan
```

ILIAS necesita un **usuario propio** para esto: así sus acciones quedan registradas a su nombre y no al de una persona.

## 8.1 Crear el usuario `cron-user` (en el navegador)

1. Entra como `root` en `http://IP_DEL_SERVIDOR`.
2. **Administración → Usuarios y roles → Usuarios → Añadir usuario**
   (en inglés: *Administration → Users and Roles → Users → Add New User*).
3. Rellena:
   - **Login:** `cron-user`
   - **Contraseña:** una larga y aleatoria. Nadie la va a escribir nunca: guárdala en tu gestor de contraseñas.
   - **Nombre y apellido** (ej. `Cron` / `Job`) y un **correo** (puede ser el tuyo).
4. En la pestaña de roles, asigna el rol global **Administrator**.
5. Guarda.

## 8.2 Probarlo a mano

Ejecuta la tarea una vez, como `www-data` (el mismo usuario que usará el cron):

```bash
cd /var/www/ilias
sudo -u www-data php cli/cron.php run-jobs cron-user pocilias
```

- `cron-user`: el login que acabas de crear. `pocilias`: el `client_id` de `ilias.json`.
- Debe terminar sin errores. Si dice que no encuentra el usuario o que no tiene permisos, revisa el login y el rol del 8.1.
- Si falla en este paso, **no sigas**: pégame la salida.

## 8.3 Programarlo

Ubuntu lee los ficheros de `/etc/cron.d/`. A diferencia del crontab personal, aquí hay una columna extra con el usuario que ejecuta:

```bash
sudo tee /etc/cron.d/ilias > /dev/null <<'EOF'
# ILIAS: tareas programadas cada 5 minutos
*/5 * * * * www-data /usr/bin/php /var/www/ilias/cli/cron.php run-jobs cron-user pocilias >> /var/www/logs/cron.log 2>&1
EOF
sudo chmod 644 /etc/cron.d/ilias
```

- `*/5 * * * *`: cada 5 minutos.
- `www-data`: el usuario que ejecuta el comando. Nunca root.
- `>> /var/www/logs/cron.log 2>&1`: guarda la salida en un log (la guía original la tira a `/dev/null`). Para aprender, y para depurar, es mejor verla.
- El fichero debe pertenecer a root y no ser escribible por otros: si no, cron lo ignora.

Comprueba que cron lo ha cogido:

```bash
systemctl is-active cron
sudo grep -i ilias /var/log/syslog | tail -3 || sudo journalctl -u cron --since "10 min ago" | tail -5
```

## 8.4 Rotar el log

El log de cron crece cada 5 minutos. Una regla de `logrotate` lo comprime y borra los antiguos:

```bash
sudo tee /etc/logrotate.d/ilias > /dev/null <<'EOF'
/var/www/logs/*.log {
    weekly
    rotate 8
    compress
    delaycompress
    missingok
    notifempty
    create 640 www-data www-data
}
EOF
sudo logrotate -d /etc/logrotate.d/ilias 2>&1 | tail -8
```

`-d` es un ensayo: muestra lo que haría sin tocar nada.

## 8.5 Comprobar que funciona (espera unos 10 minutos)

```bash
sudo tail -5 /var/www/logs/cron.log        # puede estar vacío si no hay nada que mostrar; que exista es buena señal
ls -l /var/www/logs/
```

Y en el navegador, como `root`:
**Administración → Sistema y mantenimiento → Cron jobs** (*Administration → System Settings and Maintenance → Cron Jobs*).
Arriba aparece **"Last Start of the Cron Job"**: debe tener una hora de hace menos de 5 o 10 minutos y refrescarse sola.

## Comprobación final del paso 8

```bash
cat /etc/cron.d/ilias
ls -l /etc/cron.d/ilias                    # -rw-r--r-- root root
sudo -u www-data php /var/www/ilias/cli/cron.php run-jobs cron-user pocilias; echo "salida: $?"   # 0
```
