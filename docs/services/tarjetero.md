# tarjetero

## Summary

`tarjetero` es un servicio propio que envía recordatorios de pago de tarjeta de
crédito y lleva el saldo del ciclo. Corre en `atlas` en Docker y se sirve
internamente en `tarjeta.home.arpa` a través del proxy de `rpi-01`.

Es el segundo servicio no multimedia en `atlas`, después de Vaultwarden, y se
colocó ahí por la misma razón: Docker ya estaba corriendo, el nodo tiene
capacidad de sobra, y el servicio es solo interno. Ponerlo en `rpi-01` habría ido
en contra de la prioridad de descargar el nodo de borde.

## Role

- recordatorios de corte y de fecha límite de pago, por `ntfy`
- registro de gastos y pagos del ciclo
- ingesta automática de los avisos de consumo que manda el banco

## Service Stack

| Componente | Detalle |
|---|---|
| Runtime | Node 24 en Alpine, un solo contenedor |
| Base de datos | SQLite en `/srv/tarjetero/data/tarjetero.db`, modo WAL |
| Puerto | `8090`, restringido por `ufw` a `192.168.1.30` |
| Compose | `/srv/tarjetero/app/docker-compose.yml` |
| Configuración | `/srv/tarjetero/app/.env`, modo 600 |

Sigue el patrón ya establecido en el nodo: un stack de compose por servicio bajo
`/srv`.

## Access Model

Interno únicamente. `tarjeta.home.arpa` resuelve por el rewrite comodín
`*.home.arpa → 192.168.1.30` que ya existe en ambas instancias de AdGuard, así
que **dar de alta el servicio no requirió ningún cambio de DNS**.

TLS lo termina `rpi-01` con el leaf del CA interno, que cubre `*.home.arpa`. El
nombre es de una sola etiqueta, que es lo único que cubre un comodín.

El vhost redirige `80` a `443`: son datos financieros y no se sirven en claro ni
dentro de la LAN.

## Firewall

### `ufw` no restringe un puerto publicado por Docker

Esto se descubrió durante el despliegue y merece quedar escrito, porque el
síntoma es un firewall que *parece* correcto.

Se añadió la regla que el criterio de scoping de `vault` pedía:

```
8090/tcp   ALLOW  192.168.1.30        (solo el proxy inverso)
```

`ufw status` la muestra. Y sin embargo, desde la workstation:

```
curl http://192.168.1.22:8090/healthz  ->  {"ok":true}
```

**Docker hace DNAT en `PREROUTING` y el tráfico continúa por la cadena
`FORWARD`, mientras que las reglas de `ufw` viven en `INPUT`.** El puerto acaba
respondiendo a toda la LAN aunque el firewall diga lo contrario.

Lo desconcertante es que sobre IPv6 el comportamiento era el opuesto: ahí quien
escucha es `docker-proxy`, el tráfico sí pasa por `INPUT`, y `ufw` lo bloqueaba
correctamente. El mismo puerto, filtrado en una familia y abierto en la otra.

Dos medidas:

1. **Publicar en la IP de la LAN y no en `0.0.0.0`**, lo que elimina el listener
   `[::]` y con él la superficie IPv6.
2. **Filtrar en la cadena `DOCKER-USER`**, que es la que Docker inserta al
   principio de `FORWARD` y nunca sobrescribe. Las reglas están en
   `deploy/ufw-docker-tarjetero.rules` y se aplican desde `/etc/ufw/after.rules`.

La lección generaliza a cualquier servicio en Docker de este homelab, y encaja
con la que ya está registrada en [`docs/network.md`](../network.md): **una regla
que existe no es lo mismo que una puerta cerrada; lo único que cuenta es medir
el alcance real desde fuera.**

Verificación correcta, que no es leer `ufw status`:

```
desde rpi-01:          curl -m 5 http://192.168.1.22:8090/healthz  ->  responde
desde la workstation:  el mismo curl                               ->  cuelga
```

Conviene repetirla después de un `docker restart` o de reiniciar el demonio:
Docker recrea sus cadenas.

## Notificaciones

Los avisos salen por `ntfy`, el mismo canal que ya usan las alertas de respaldo.
La conexión es saliente, así que no abre ninguna superficie nueva.

**El detalle del mensaje es configurable y por defecto va en `low`**, que omite
montos y comercios. Importa porque si el `ntfy` en uso es el público `ntfy.sh`,
el texto del aviso cruza un tercero. Con un `ntfy` autoalojado y alcanzable
tendría sentido subirlo a `full`; con el público, no.

Por la misma razón, **los SMS del banco no se enrutan por `ntfy`**: el reenviador
del celular entrega directo al servicio, y por eso sólo funciona desde la LAN o
la VPN.

## Consecuencia conocida del modelo interno

Los avisos de pago llevan un botón "Ya pagué" que apunta a
`https://tarjeta.home.arpa/api/cycles/<id>/paid`. Ese nombre sólo resuelve dentro,
así que el botón funciona en casa o por VPN y no desde una red ajena. El aviso sí
llega a cualquier parte, porque `ntfy` es saliente.

Con la migración a Tailscale (2026-08-29) esta limitación quedó prácticamente
resuelta: el celular lleva el tailnet consigo, el split DNS resuelve
`tarjeta.home.arpa` desde fuera, y el vhost admite el rango `100.64.0.0/10`,
así que el botón funciona desde cualquier red donde Tailscale esté activo. La
alternativa de publicar *sólo* ese endpoint por el túnel de Cloudflare queda
anotada por si algún día se quiere que funcione sin VPN; la URL ya va firmada
con un HMAC acotado a un ciclo y a una operación, no con un secreto
reutilizable.

## Backups

`tarjetero-backup.timer` corre a las 02:10 y deja un volcado comprimido en
`/srv/nas/backups/atlas/tarjetero/`, de donde el barrido de `vault` a las 03:30 lo
mete en el repositorio de restic.

El horario se eligió para caer entre el volcado de Vaultwarden (02:00) y el de
`rpi-01` (02:20), y bien antes del barrido.

El script usa `sqlite3 .backup` y no `cp`, porque la base está en WAL y copiarla
en caliente produce un archivo que parece bueno y está truncado. Corre
`PRAGMA integrity_check` **antes** de publicar el archivo, y rota los viejos
**después** de escribir el nuevo, de modo que un fallo de hoy no se lleva por
delante los respaldos que sí sirven.

Falla ruidosamente: si el staging no está montado, si la base no existe, o si la
verificación no pasa, sale con código distinto de cero y puede mandar un `ntfy`.

## TLS y la versión de nginx

El vhost declara `listen 443 ssl http2;`, la sintaxis anterior a nginx 1.25.1.
`rpi-01` corre 1.24.0, el de Ubuntu 24.04, donde la forma moderna `http2 on;`
aborta el arranque con `unknown directive` y **deja la configuración en un estado
que no recarga**. Es la misma declaración que ya usa `pass.home.arpa`.

El leaf del CA interno cubrió el nombre sin reemitirse, porque `*.home.arpa`
abarca cualquier etiqueta única y `tarjeta` lo es.

## Current State

Desplegado y verificado el 2026-08-16.

Lo comprobado midiendo, no leyendo configuración:

| Comprobación | Resultado |
|---|---|
| `https://tarjeta.home.arpa` con TLS validado contra el CA interno | 200, sin `-k` |
| Redirección de `80` a `443` | 301 |
| Assets de la PWA (`manifest`, `sw.js`, icono) | 200 |
| Sesión a través del proxy con cookie `Secure` | funciona |
| Contraseña incorrecta | 401 |
| `:8090` directo desde `rpi-01` | responde |
| `:8090` directo desde la workstation | bloqueado |
| Respaldo a `vault` | `tarjetero-20260816.db.gz` publicado |
| Aviso de prueba por `ntfy` | entregado |

Una tarjeta dada de alta: corte el 5, pago el 10, `America/Lima`, `PEN`.

La ingesta por IMAP está activa contra Gmail, acotada por remitente al banco.
**No modifica el buzón**: no marca como leído, no mueve nada. El estado de qué
se procesó vive en `ingest_raw.source_ref`, con índice único. Verificado: 14.515
correos sin leer antes y después de la primera pasada.

**La ventana entre corte y pago es de cinco días**, más corta que la de una
tarjeta típica. Los recordatorios por defecto no cabían en ella y hubo que
recortarlos: la ventana de "falta capturar el saldo" termina como muy tarde el
día anterior al vencimiento, y se omite el aviso de pago que caería antes del
propio corte. Queda como cuatro avisos por ciclo: día 2, 6, 8-9 y 10.

## Planned Changes

- activar la ingesta por IMAP: el conector está escrito y desactivado a falta de
  una contraseña de aplicación de Gmail; el parser específico del banco necesita
  correos reales de ejemplo
- importación de CSV del estado de cuenta para conciliar
- exponer `/metrics` en formato Prometheus para que `monitor` lo scrapee
