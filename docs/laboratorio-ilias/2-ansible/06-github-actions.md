# A6. GitHub Actions: CI y despliegue

Repo: `bedoyacloud/ilias-ansible` (público). Dos workflows en `.github/workflows/`.

## A6.1 CI ([ci.yml](../../../.github/workflows/ci.yml))

En cada *push* a `main` y en cada *pull request*: instala Ansible y las colecciones, y ejecuta `--syntax-check` y `ansible-lint`.
No usa servidor ni secretos. Como `hosts.yml` y `vault.yml` están en `.gitignore`, el `syntax-check` usa `hosts.example.yml`.
Ver los resultados: pestaña **Actions** del repo, o `gh run list`.

## A6.2 Despliegue ([deploy.yml](../../../.github/workflows/deploy.yml))

Se lanza **a mano** (*Actions → Deploy → Run workflow*) y pregunta dos cosas:

| Entrada | Valores | Por defecto |
|---|---|---|
| `tags` | `all`, `plugins`, `ilias`, `nginx`, `php`, `mariadb`, `base`, `maintenance` | `all` |
| `dry_run` | sí / no | **sí**: solo `--check --diff`, no cambia nada |

Pasos del job:

1. Instala Ansible y las colecciones.
2. **Abre el puerto 22** del security group a la IP de este *runner* (solo a su IP, con `/32`).
3. Prepara la llave SSH, el inventario y los secretos desde *GitHub Secrets*.
4. `--check --diff`: enseña qué cambiaría.
5. Si `dry_run` es no: aplica y hace una **prueba de humo** (`login.php` debe dar 200).
6. **Cierra el puerto 22** siempre (`if: always()`), aunque el despliegue falle, y borra la llave y los secretos del *runner*.

**Qué se despliega:** el commit de `main`. Para actualizar un addon se cambia su `version` en `main.yml`, se sube por *pull request*
(el CI lo valida) y se lanza el deploy.

## A6.3 Secretos y variables

| Nombre | Tipo | Qué es | Estado |
|---|---|---|---|
| `DEPLOY_SSH_KEY` | secreto | Llave privada **solo para el pipeline** (distinta de la que usas para entrar tú), autorizada en `ilias-01` | hecho |
| `DEPLOY_KNOWN_HOSTS` | secreto | Huella del servidor: evita desactivar `host_key_checking` | hecho |
| `DEPLOY_HOST` | secreto | IP del servidor | hecho |
| `ILIAS_DB_PASSWORD`, `ILIAS_ROOT_PASSWORD` | secreto | Las mismas del Vault. En CI hacen su papel (no se sube ningún `vault.yml`) | hecho |
| `AWS_REGION` | variable | `eu-north-1` | hecho |
| `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` | secreto | Usuario IAM del pipeline | **pendiente** |
| `AWS_SG_ID` | variable | ID del security group `ilias-sg` (`sg-...`) | **pendiente** |

Las contraseñas deben seguir siendo las del servidor: si cambian, el despliegue las cambiaría también en MariaDB y en `ilias.json`.

## A6.4 Lo que falta (lo haces tú, 10 minutos)

**1. Un usuario IAM que solo pueda abrir y cerrar el puerto 22 de ese security group.**
En la consola de AWS: **IAM → Policies → Create policy → JSON**, y pega esto cambiando `REGION`, `CUENTA` y `sg-XXXX`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:AuthorizeSecurityGroupIngress",
        "ec2:RevokeSecurityGroupIngress"
      ],
      "Resource": "arn:aws:ec2:REGION:CUENTA:security-group/sg-XXXX"
    },
    {
      "Effect": "Allow",
      "Action": "ec2:CreateTags",
      "Resource": "arn:aws:ec2:REGION:CUENTA:security-group-rule/*"
    }
  ]
}
```

Luego **IAM → Users → Create user** (`github-actions-ilias`), asígnale esa política, y en *Security credentials → Create access key*
(caso de uso *Application running outside AWS*) copia las dos claves.

**2. Guardarlas en GitHub** (el valor se pide por teclado y no queda en el historial):

```bash
gh secret set AWS_ACCESS_KEY_ID -R bedoyacloud/ilias-ansible
gh secret set AWS_SECRET_ACCESS_KEY -R bedoyacloud/ilias-ansible
gh variable set AWS_SG_ID -R bedoyacloud/ilias-ansible --body sg-XXXX
```

**3. Probar.** Primero en seco:

```bash
gh workflow run deploy.yml -R bedoyacloud/ilias-ansible -f tags=all -f dry_run=true
gh run watch -R bedoyacloud/ilias-ansible
```

Debe terminar en verde con `changed=0`, y en el security group no debe quedar ninguna regla `github-actions-deploy`.

## A6.5 Limitaciones y cómo cambiaría con Hetzner

- **Aprobación obligatoria.** El entorno `production` exige un revisor (*Settings → Environments → production*): nadie despliega sin que el dueño apruebe. Además, `dry_run = sí` por defecto.
- **El paso del security group es de AWS.** En Hetzner dedicado, ese paso se cambia por la API del firewall de Hetzner, o por un *runner* propio
  dentro de la red. El resto del workflow no cambia.
- **Ansible solo ve el servidor por SSH**: lo que no esté en el playbook (por ejemplo, el usuario `deploy` o el firewall del proveedor) hay que prepararlo antes.

## A6.6 Seguridad de un repo público

Los secretos de GitHub son de solo escritura: ni el dueño puede volver a leerlos, ni en repos privados ni en públicos. El riesgo de hacer público el repo
no está en los secretos guardados, sino en **filtrarlos después** por un log o un workflow mal escrito. Lo que se ha revisado:

| Riesgo | Cómo se evita aquí |
|---|---|
| Un `echo` de un secreto en un log público | Los workflows solo imprimen la IP del *runner* y el código HTTP. Los secretos pasan por variables de entorno y a un fichero con permisos 0600 que se borra al terminar |
| Secretos que aparecen en la salida de Ansible (`--diff`, errores) | Toda tarea que toca una contraseña lleva `no_log: true` (usuario de MariaDB, `ilias.json`, hash de `root`, `UPDATE`). Nunca se usa `-v` |
| *Pull requests* de *forks* que roban secretos | El CI corre con `pull_request` (sin acceso a secretos para forks) y **no** se usa `pull_request_target`. El deploy solo se lanza con `workflow_dispatch`, que exige permiso de escritura en el repo |
| Que un colaborador externo ejecute workflows | *Settings → Actions → General → Fork pull request workflows*: exigir aprobación para todo colaborador externo |
| Que el token del workflow pueda escribir en el repo | Permisos del `GITHUB_TOKEN` por defecto en solo lectura, y `permissions: contents: read` en cada workflow |
| Que el pipeline haga más de lo previsto en AWS | El usuario IAM solo puede abrir y cerrar el puerto 22 de **un** security group |
| Llave SSH del pipeline robada | Es una llave **solo para el pipeline**, no la personal. Se revoca quitando su línea de `authorized_keys` |

Antes de publicar se buscó en todo el repo: IPs, correos, rutas del equipo local, nombres de llaves, claves privadas y cadenas con aspecto de contraseña. No quedó nada.
Los ficheros reales (`vault.yml`, `hosts.yml`, `.vault-pass`) están en `.gitignore` y nunca se han subido.
