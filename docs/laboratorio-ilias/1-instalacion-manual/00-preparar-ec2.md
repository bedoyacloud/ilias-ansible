# 0. Preparar la EC2

## Por qué esta instancia

| | Mínimo ILIAS 11 | PoC | Motivo |
|---|---|---|---|
| SO | — | Ubuntu 24.04 LTS | La guía de ILIAS usa 24.04. 26.04 trae PHP 8.5 y ILIAS 11 solo soporta 8.3/8.4 |
| CPU / RAM | 2 / 4 GB | t3.small: 2 / 2 GB + 2 GB swap | `composer` y `npm` necesitan memoria al instalar; con 1 GB fallan |
| Disco | 25 GB | 30 GB gp3 | Código + `vendor` + `node_modules` + datos |

## Pasos en la consola de AWS

Trabaja en la misma región donde está el par de claves `mi-llave`.

### 0.1 Security group `ilias-sg`

EC2 → Grupos de seguridad → Crear. Reglas de **entrada**:

| Tipo | Puerto | Origen | Por qué |
|---|---|---|---|
| SSH | 22 | **Mi IP** | Solo tú puedes intentar entrar por SSH |
| HTTP | 80 | Anywhere IPv4 | La web de ILIAS |

- No abras el 3306: MariaDB solo escucha en localhost.
- El 443 se añade cuando haya dominio y HTTPS.
- Si cambia la IP de tu casa, SSH dará *timed out*: edita la regla y vuelve a elegir "Mi IP".

### 0.2 Lanzar la instancia

EC2 → Instancias → Lanzar instancias:

- Nombre: `ilias-poc`
- AMI: **Ubuntu Server 24.04 LTS**, arquitectura 64 bits (x86)
- Tipo: `t3.small`
- Par de claves: `mi-llave`
- Red → Seleccionar grupo de seguridad existente → `ilias-sg`
- Almacenamiento: **30 GiB gp3**

Espera a que diga **Running** y **2/2 comprobaciones superadas**.

### 0.3 Elastic IP

Sin ella, la IP pública cambia cada vez que detienes e inicias la instancia.

EC2 → Direcciones IP elásticas → Asignar → selecciónala → Acciones → Asociar → instancia `ilias-poc`.

Si algún día borras la instancia, libera también la Elastic IP: si queda sin asociar, AWS la sigue cobrando.

## Comprobar

Desde tu Mac, cambiando `IP_ELASTICA`:

```bash
ssh -i ~/.ssh/mi-llave.pem ubuntu@IP_ELASTICA
```

La primera vez responde `yes`. Ya dentro:

```bash
lsb_release -ds    # Ubuntu 24.04.x LTS
nproc              # 2
free -h            # Mem total ~1.9Gi
df -h /            # Size ~29G
```

## Al terminar el día

EC2 → Instancias → `ilias-poc` → Estado de la instancia → **Detener** (no Terminar). La Elastic IP se mantiene.
