# A6. GitHub Actions: CI y despliegue en varios servidores

Dos workflows en `.github/workflows/`, y una acción compartida en `.github/actions/prepare/`.

## A6.1 CI ([ci.yml](../../../.github/workflows/ci.yml))

En cada *push* a `main` y en cada *pull request*: instala Ansible y las colecciones, y ejecuta `--syntax-check` y `ansible-lint`.
No usa servidores ni secretos. Como `hosts.yml` y `vault.yml` no están en el repo, el `syntax-check` usa `hosts.example.yml`.

## A6.2 Despliegue ([deploy.yml](../../../.github/workflows/deploy.yml))

Se lanza **a mano** (*Actions → Deploy → Run workflow*) y pregunta tres cosas:

| Entrada | Valores | Por defecto |
|---|---|---|
| `target` | `all`, `ilias-01`, `ilias-fresh`, `ilias-contabo` | `all` |
| `tags` | `plugins`, `all`, `ilias`, `nginx`, `php`, `mariadb`, `base`, `maintenance`, `metrics`, `monitoring` | `plugins` |
| `dry_run` | sí / no | **sí**: solo el plan, no cambia nada |

```
Elegir servidores ─> Plan (un job por servidor, sin aprobación) ─> Aplicar (un job por servidor, con aprobación)
```

1. **Elegir servidores:** convierte `target` en una matriz. Cada servidor lleva la marca `aws`, que indica si su firewall (*security group*) hay que abrir.
2. **Plan:** por cada servidor ejecuta `--check --diff` y publica el resultado en el *Summary* de la ejecución. No pide aprobación.
3. **Aplicar:** solo si `dry_run` es no, y **después de ver el plan**. El entorno `production` exige la aprobación de un revisor; una sola aprobación cubre a todos los servidores.
4. En cada servidor de AWS, abre el puerto 22 solo a la IP del *runner*, espera a que **responda de verdad**, ejecuta, y lo cierra siempre (`if: always()`).
   En un servidor con el SSH ya accesible (el VPS) no se toca ningún firewall.
5. **Prueba de humo:** desde el propio servidor, por SSH, comprueba que ILIAS responde 200 y lista los plugins registrados.

**Qué se despliega:** el commit de `main`. Los addons de cada servidor están en [`inventory/host_vars/<servidor>/plugins.yml`](../../../ansible/inventory/host_vars/),
versionados. Cambiar el addon de un servidor es cambiar ese fichero, abrir un *pull request* (el CI lo valida), fusionarlo y lanzar el deploy.

## A6.3 Secretos y variables

Los valores se guardan en *GitHub Secrets* (solo escritura: ni el dueño puede volver a leerlos). Hacen el papel que en local hacen `hosts.yml` y el Vault.

| Nombre | Tipo | Qué contiene |
|---|---|---|
| `DEPLOY_SSH_KEY` | secreto | Llave privada **solo para el pipeline**, autorizada en cada servidor |
| `DEPLOY_KNOWN_HOSTS` | secreto | Huellas SSH de los servidores (no se desactiva `host_key_checking`) |
| `INVENTORY_YML` | secreto | El `hosts.yml` completo: IPs y variables de cada servidor |
| `VAULT_FILES_B64` | secreto | Los `vault.yml` descifrados (contraseñas de cada servidor), en un tar.gz en base64 |
| `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` | secreto | Usuario IAM del pipeline (ver A6.4) |
| `AWS_REGION`, `AWS_SG_ID` | variable | Región y *security group* donde abrir el puerto 22 |

Para generarlos (el valor no se muestra):

```bash
gh secret set DEPLOY_SSH_KEY -R USUARIO/REPO < llave_del_pipeline
gh secret set INVENTORY_YML  -R USUARIO/REPO < hosts_para_ci.yml          # con ansible_ssh_private_key_file: ~/.ssh/deploy_key
tar -czf - -C directorio_con_vaults_descifrados inventory | base64 | tr -d '\n' | gh secret set VAULT_FILES_B64 -R USUARIO/REPO
```

## A6.4 Usuario IAM del pipeline (solo servidores de AWS)

Una política que solo puede abrir y cerrar reglas **de un security group**. En **IAM → Policies → Create policy → JSON**, cambiando `REGION`, `CUENTA` y `sg-XXXX`:

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
      "Resource": [
        "arn:aws:ec2:REGION:CUENTA:security-group/sg-XXXX",
        "arn:aws:ec2:REGION:CUENTA:security-group-rule/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": "ec2:CreateTags",
      "Resource": "arn:aws:ec2:REGION:CUENTA:security-group-rule/*"
    }
  ]
}
```

AWS evalúa las acciones de abrir y cerrar también sobre el recurso *regla* (`security-group-rule/*`): sin esa segunda línea falla con `UnauthorizedOperation`.
Luego **IAM → Users → Create user**, se le asigna la política y se crea una *access key* (*Application running outside AWS*).

## A6.5 Resultado: un addon distinto en cada servidor

Desplegado por el pipeline, con una sola aprobación para los tres:

| Servidor | Addon del `plugins.yml` | Tipo | Resultado |
|---|---|---|---|
| `ilias-01` (AWS) | JSXGraph (`release_11`) | componente de página | instalado y activo |
| `ilias-fresh` (AWS) | LiveVoting (`dev_11`) | objeto de repositorio, con dependencias de composer | instalado y activo |
| `ilias-contabo` (VPS) | GeoGebra (`release_11`) | componente de página | instalado y activo |

Los tres tenían GeoGebra de pruebas anteriores (se quitó del VPS antes para que fuera una instalación real); el rol añade plugins, no los desinstala.
Una segunda ejecución del mismo despliegue dio **`changed=0` en los tres** y 200 en la prueba de humo, y el registro completo no contiene IPs, dominios ni claves.

## A6.6 Problemas reales encontrados

| Síntoma | Causa | Solución |
|---|---|---|
| El registro de una ejecución mostraba las IPs y el dominio | GitHub enmascara las **líneas completas** de un secreto multilínea, no sus valores sueltos, y la prueba de humo imprimía la URL | La acción `prepare` lee las IPs y dominios del inventario y los registra con `::add-mask::`; la prueba de humo ya no imprime la URL. La ejecución con la fuga se eliminó |
| `Connection timed out` en un servidor de AWS, justo tras abrir la regla | Un cambio en un security group tarda unos segundos en aplicarse (las IPs de los dos *runners* eran distintas: no era un choque) | Un paso espera a que el puerto 22 responda (hasta 60 s) y `reconnection_retries = 3` en `ansible.cfg` |
| `ilias-fresh:` aparece como `***` en el registro | El nombre del servidor coincide con una línea del secreto `INVENTORY_YML`, que GitHub enmascara | Es solo estético. No se imprime sin necesidad |
| El plan marca menos cambios de los que habrá | En `--check`, las tareas que dependen de otras (instalar tras clonar) se omiten, porque lo anterior no se ejecuta de verdad | Es una limitación de `--check`: el plan indica *qué* cambia, no *todo* lo que hará. El *Summary* se lee como una pista, no como un inventario completo |
| `\s` no funcionaba en el `sed` del enmascarado | macOS y Linux interpretan `\s` distinto | Clases POSIX (`[[:space:]]`), que valen en ambos |

## A6.6b Seguridad de un repo público

Los secretos de GitHub son de solo escritura: ni el dueño puede volver a leerlos, ni en repos privados ni en públicos. El riesgo de un repo público
no está en los secretos guardados sino en **filtrarlos después**, por un registro o un workflow mal escrito.

| Riesgo | Cómo se evita aquí |
|---|---|
| Un secreto impreso en un registro público | Los workflows no imprimen secretos. Las IPs y dominios del inventario se enmascaran con `add-mask`. Se **escanea el registro completo** de cada despliegue en busca de IPs, dominios y claves |
| Secretos en la salida de Ansible (`--diff`, errores) | Toda tarea que toca una contraseña lleva `no_log: true`. Nunca se usa `-v` |
| *Pull requests* de *forks* que roban secretos | El CI usa `pull_request` (sin acceso a secretos para forks); **no** se usa `pull_request_target`. El deploy solo se lanza con `workflow_dispatch`, que exige permiso de escritura |
| Colaboradores externos ejecutando workflows | *Settings → Actions → General*: exigir aprobación para todo colaborador externo |
| Que el token del workflow pueda escribir | Permisos del `GITHUB_TOKEN` en solo lectura y `permissions: contents: read` en cada workflow |
| Que el pipeline haga más de lo previsto en AWS | El usuario IAM solo puede abrir y cerrar reglas de **un** security group |
| Llave SSH del pipeline robada | Es una llave solo para el pipeline, no la personal; se revoca quitando su línea de `authorized_keys` |
| Desplegar sin revisar | El entorno `production` exige un revisor; el plan se ve **antes** de aprobar |

**Un punto débil a mejorar:** la llave del pipeline entra en cada servidor con un usuario que tiene `sudo` sin contraseña. En producción convendría un usuario
`deploy` con permisos acotados (solo los comandos que necesita Ansible), sobre todo en un servidor compartido.

## A6.7 Con Hetzner

El paso de AWS (abrir y cerrar el security group) es el único específico de proveedor. En un servidor dedicado de Hetzner se sustituye por la API de su firewall
o por un *runner* propio dentro de la red; si el SSH ya está accesible desde internet (con solo llave y fail2ban), no hace falta ninguno, como en el VPS de este laboratorio.
El resto del workflow no cambia.
