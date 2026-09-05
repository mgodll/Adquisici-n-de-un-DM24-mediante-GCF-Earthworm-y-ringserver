# Manual de instalación desde cero

## Adquisición de un DM24 mediante GCF, Earthworm y ringserver

Este procedimiento instala en un contenedor Docker un adquisidor para la estación DM24 DIAM y entrega sus datos al `ringserver` existente. Está basado en el escenario validado en `tororoi`.

## 1. Resultado esperado

```text
DM24 172.16.18.87:1568 (GCF/MSS)
              │
              ▼
     gcf2ew, Earthworm 32 bits
              │ TYPE_TRACEBUF2
              ▼
       DIAM_RING, clave 3100
              │
              ▼
        ew2ringserver
              │ miniSEED/DataLink
              ▼
 ringserver 127.0.0.1:16000
              │ SeedLink
              ▼
       172.16.10.2:18000
```

Streams finales:

```text
CM.DIAM.00.HHE
CM.DIAM.00.HHN
CM.DIAM.00.HHZ
```

## 2. Consideraciones previas

- Servidor: Debian 11 o posterior, arquitectura `amd64`.
- Docker debe estar instalado y funcionando.
- El DM24 debe ser alcanzable desde el servidor por `172.16.18.87:1568/TCP`.
- El `ringserver` existente debe escuchar DataLink en `16000` y SeedLink en `18000`.
- Se usa un contenedor `linux/386` porque la biblioteca `libgcf.a` incluida con `gcf2ew` contiene objetos i386 de 32 bits.
- Antes de uso comercial debe verificarse la licencia de la biblioteca GCF suministrada con Earthworm/ISTI.
- Los códigos `CM`, `DIAM`, `00`, `HHE`, `HHN` y `HHZ` deben coincidir con los metadatos oficiales de la estación.

## 3. Verificaciones iniciales

### Sistema operativo y arquitectura

```bash
cat /etc/os-release
uname -m
sudo docker version
```

### Ruta y puerto del DM24

```bash
ip route get 172.16.18.87
nc -vz -w 5 172.16.18.87 1568
```

### Puertos de ringserver

```bash
sudo ss -ltnp | grep -E ':16000|:18000'
```

Se esperan listeners en ambos puertos. Compruebe la configuración:

```bash
sudo grep -En '^[[:space:]]*(RingDirectory|RingSize|DataLinkPort|SeedLinkPort|ServerID)' \
  /usr/local/Apps/ringserver/ring.conf
```

Valores esperados en este servidor:

```text
DataLinkPort 16000
SeedLinkPort 18000
```

No modifique ni reinicie el ringserver existente para montar el contenedor.

## 4. Crear el proyecto

```bash
mkdir -p /home/julian/diam-earthworm-docker/params
mkdir -p /home/julian/earthworm-diam-container-logs
cd /home/julian/diam-earthworm-docker
```

La estructura final será:

```text
diam-earthworm-docker/
├── Dockerfile
├── entrypoint.sh
└── params/
    ├── gcf2ew.d
    ├── ew2ringserver.d
    └── startstop_unix.d
```

## 5. Crear el Dockerfile

Cree `/home/julian/diam-earthworm-docker/Dockerfile` con este contenido:

```dockerfile
FROM --platform=linux/386 debian:bookworm

ARG DEBIAN_FRONTEND=noninteractive
ARG EARTHWORM_TAG=v8.0b17

RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      build-essential ca-certificates curl file gfortran \
      libtirpc-dev netbase procps xxd \
 && rm -rf /var/lib/apt/lists/*

WORKDIR /opt/earthworm

RUN curl -fsSL \
      "https://gitlab.com/seismic-software/earthworm/-/archive/${EARTHWORM_TAG}/earthworm-${EARTHWORM_TAG}.tar.gz" \
      -o /tmp/earthworm.tar.gz \
 && tar -xzf /tmp/earthworm.tar.gz \
 && mv "earthworm-${EARTHWORM_TAG}" earthworm_8.0 \
 && rm /tmp/earthworm.tar.gz

ENV EW_INSTALL_HOME=/opt/earthworm \
    EW_INSTALL_VERSION=earthworm_8.0 \
    EW_RUN_DIR=/opt/earthworm/run_diam \
    EW_INSTALL_INSTALLATION=INST_UNKNOWN \
    EW_INSTALL_BITS=32

RUN mkdir -p /opt/earthworm/run_diam/params \
             /opt/earthworm/run_diam/log \
             /opt/earthworm/run_diam/data \
 && bash -lc '. /opt/earthworm/earthworm_8.0/environment/ew_linux.bash; cd /opt/earthworm/earthworm_8.0/src; make unix' \
 && cp /opt/earthworm/earthworm_8.0/environment/earthworm.d \
       /opt/earthworm/run_diam/params/earthworm.d \
 && cp /opt/earthworm/earthworm_8.0/environment/earthworm_global.d \
       /opt/earthworm/run_diam/params/earthworm_global.d

# gcf2ew es código heredado. Su biblioteca GCF solamente está disponible aquí en 32 bits.
# Se eliminan dos Werror para permitir compilar declaraciones antiguas con GCC moderno.
RUN sed -i 's/-Werror=implicit-function-declaration//g; s/-Werror=int-conversion//g' \
      /opt/earthworm/earthworm_8.0/environment/ew_linux.bash \
 && bash -lc '. /opt/earthworm/earthworm_8.0/environment/ew_linux.bash; cd /opt/earthworm/earthworm_8.0/src/data_sources/gcf2ew; make -f makefile.unix' \
 && test -x /opt/earthworm/earthworm_8.0/bin/gcf2ew \
 && file /opt/earthworm/earthworm_8.0/bin/gcf2ew

COPY params/ /opt/earthworm/run_diam/params/
COPY entrypoint.sh /usr/local/bin/diam-entrypoint

RUN printf '%s\n' \
      'Ring     DIAM_RING                   3100' \
      'Module   MOD_GCF2EW_DIAM             250' \
      'Module   MOD_EW2RINGSERVER_DIAM      251' \
      'Message  TYPE_GCFSOH_PACKET          120' \
      >> /opt/earthworm/run_diam/params/earthworm.d \
 && chmod 0755 /usr/local/bin/diam-entrypoint

ENTRYPOINT ["/usr/local/bin/diam-entrypoint"]
```

Antes de usar los números 250, 251, 3100 y 120 en otra instalación Earthworm, compruebe que no estén duplicados en `earthworm.d`.

## 6. Crear el punto de entrada

Cree `entrypoint.sh`:

```bash
#!/bin/bash
set -e

export EW_INSTALL_HOME=/opt/earthworm
export EW_INSTALL_VERSION=earthworm_8.0
export EW_RUN_DIR=/opt/earthworm/run_diam
export EW_INSTALL_INSTALLATION=INST_UNKNOWN
export EW_INSTALL_BITS=32

source /opt/earthworm/earthworm_8.0/environment/ew_linux.bash
cd /opt/earthworm/run_diam/params
exec startstop
```

Asígnele permisos:

```bash
chmod 0755 entrypoint.sh
```

## 7. Configurar gcf2ew

Cree `params/gcf2ew.d`:

```text
MyModuleId     MOD_GCF2EW_DIAM
RingName       DIAM_RING
HeartbeatInt   10
LogFile        1
TimeoutNoSend  0
SaveSOH2LOG    1
InjectSOH      0

HostAddress    172.16.18.87
PortNumber     1568

#                System  Stream  Station Channel Network Location
InfoSCNL         OVSM    DIAME2  DIAM    HHE     CM      00
InfoSCNL         OVSM    DIAMN2  DIAM    HHN     CM      00
InfoSCNL         OVSM    DIAMZ2  DIAM    HHZ     CM      00
```

`TimeoutNoSend 0` evita que `gcf2ew` termine durante pausas o entregas por ráfagas del DM24. Esto no sustituye la supervisión externa.

`InfoSCNL` funciona simultáneamente como selector y mapeo. Solamente pasan los streams definidos. El orden es:

```text
InfoSCNL SystemID StreamID Station Channel Network Location
```

## 8. Configurar ew2ringserver

Cree `params/ew2ringserver.d`:

```text
MyModuleId       MOD_EW2RINGSERVER_DIAM
RingName         DIAM_RING
RSAddress        127.0.0.1:16000
HeartBeatInt     30
LogFile          1

GetMsgLogo       INST_WILDCARD MOD_GCF2EW_DIAM TYPE_TRACEBUF2
MaxMsgSize       4096
QueueSize        1000
ReconnectInterval 10
Int32Encoding    STEIM2

Send_scnl        DIAM HHE CM 00
Send_scnl        DIAM HHN CM 00
Send_scnl        DIAM HHZ CM 00
```

El puerto `16000` es DataLink para escritura. Los clientes SeedLink consultan el puerto `18000`.

## 9. Configurar startstop

Cree `params/startstop_unix.d`:

```text
Ring           DIAM_RING 16384

MyModuleId     MOD_STARTSTOP
HeartbeatInt   30
MyClassName    OTHER
MyPriority     0
LogFile        1
KillDelay      10
HardKillDelay  5

Process        "gcf2ew gcf2ew.d"
Class/Priority OTHER 0

Process        "ew2ringserver ew2ringserver.d"
Class/Priority OTHER 0
```

## 10. Revisar archivos antes de compilar

```bash
cd /home/julian/diam-earthworm-docker
find . -maxdepth 2 -type f -print
grep -nE 'HostAddress|PortNumber|InfoSCNL|Send_scnl|RSAddress' \
  params/gcf2ew.d params/ew2ringserver.d
```

Debe verse:

```text
172.16.18.87
1568
127.0.0.1:16000
DIAM HHE CM 00
DIAM HHN CM 00
DIAM HHZ CM 00
```

## 11. Construir la imagen

```bash
cd /home/julian/diam-earthworm-docker
sudo docker build --platform linux/386 \
  -t earthworm-diam:gcf-test . 2>&1 | \
  tee /home/julian/earthworm-diam-docker-build.log
```

La compilación completa puede tardar varios minutos y producir advertencias de código heredado.

Validar la imagen y el ejecutable:

```bash
sudo docker image inspect earthworm-diam:gcf-test \
  --format 'arquitectura={{.Architecture}} sistema={{.Os}} tamaño={{.Size}}'

sudo docker run --rm --platform linux/386 \
  --entrypoint /usr/bin/file earthworm-diam:gcf-test \
  /opt/earthworm/earthworm_8.0/bin/gcf2ew
```

Se espera arquitectura `386` y un ejecutable `ELF 32-bit ... Intel 80386`.

## 12. Prueba controlada interactiva

Antes de crear el servicio permanente, compruebe que no exista otro contenedor adquiriendo DIAM:

```bash
sudo docker ps -a --filter name=earthworm-diam
sudo ss -tnp | grep '172.16.18.87:1568' || true
```

Ejecute:

```bash
sudo docker run --rm -it --init \
  --platform linux/386 \
  --name earthworm-diam-test \
  --network host \
  -v /home/julian/earthworm-diam-container-logs:/opt/earthworm/run_diam/log \
  earthworm-diam:gcf-test
```

Mensajes esperados:

```text
gcf2ew: MSS Connect with 172.16.18.87 1568
connected to: DataLink ...
capabilities: ... WRITE
gcf2ew Alive
ew2ringserver Alive
```

En otra terminal:

```bash
sudo docker top earthworm-diam-test
sudo ss -tnp | grep -E '172.16.18.87:1568|127.0.0.1:16000'
```

Comprobar paquetes durante 90 segundos:

```bash
sudo docker exec earthworm-diam-test bash -lc \
  'source /opt/earthworm/earthworm_8.0/environment/ew_linux.bash; timeout 90 sniffring DIAM_RING INST_WILDCARD MOD_GCF2EW_DIAM TYPE_TRACEBUF2'
```

Comprobar encabezados:

```bash
sudo docker exec earthworm-diam-test bash -lc \
  'source /opt/earthworm/earthworm_8.0/environment/ew_linux.bash; timeout 90 sniffwave DIAM_RING wild wild CM wild n noflush verbose'
```

Detenga la prueba escribiendo `quit` en la consola interactiva. Confirme que terminó:

```bash
sudo docker ps --filter name=earthworm-diam-test
```

## 13. Crear el servicio permanente

```bash
sudo docker run -d \
  --init \
  --platform linux/386 \
  --name earthworm-diam \
  --restart unless-stopped \
  --network host \
  -v /home/julian/earthworm-diam-container-logs:/opt/earthworm/run_diam/log \
  earthworm-diam:gcf-test
```

Validar:

```bash
sudo docker ps --filter name=earthworm-diam
sudo docker top earthworm-diam
sudo docker logs --tail 100 earthworm-diam
sudo docker inspect earthworm-diam \
  --format 'estado={{.State.Status}} reinicios={{.RestartCount}} inicio={{.State.StartedAt}}'
```

El mensaje siguiente es normal en modo no interactivo:

```text
Interactive() thread exiting: fgets returned NULL
```

`Queue is empty, waiting for producer` indica una pausa sin nuevos paquetes, no necesariamente una desconexión.

## 14. Verificar la publicación SeedLink

Debian Bookworm no proporciona `slinktool` como paquete estándar en este escenario. Se puede compilar temporalmente dentro de otro contenedor sin instalarlo en el host:

```bash
sudo docker run --rm --network host debian:bookworm-slim bash -lc 'apt-get update >/dev/null && apt-get install -y --no-install-recommends build-essential ca-certificates curl >/dev/null && curl -fsSL https://github.com/EarthScope/slinktool/archive/refs/tags/v4.5.0.tar.gz | tar -xz && cd slinktool-4.5.0 && make >/dev/null && ./slinktool -Q 127.0.0.1:18000' 2>&1 | grep -i DIAM
```

Resultado esperado:

```text
CM DIAM  00 HHE D fecha_inicial - fecha_final
CM DIAM  00 HHN D fecha_inicial - fecha_final
CM DIAM  00 HHZ D fecha_inicial - fecha_final
```

Repita la consulta varios minutos después. Los tiempos finales deben avanzar.

## 15. Operación básica

### Parar

```bash
sudo docker stop -t 30 earthworm-diam
```

### Arrancar

```bash
sudo docker start earthworm-diam
```

### Reiniciar

```bash
sudo docker restart -t 30 earthworm-diam
```

### Revisar logs

```bash
sudo docker logs --since 30m earthworm-diam
tail -100 /home/julian/earthworm-diam-container-logs/gcf2ew*.log
tail -100 /home/julian/earthworm-diam-container-logs/ew2ringserver*.log
```

## 16. Actualizar configuraciones

Los archivos de configuración están incorporados en la imagen mediante `COPY`. Después de modificar `params/*.d`, reconstruya y recree el contenedor:

```bash
cd /home/julian/diam-earthworm-docker
sudo docker build --platform linux/386 -t earthworm-diam:gcf-test .
sudo docker stop -t 30 earthworm-diam
sudo docker rm earthworm-diam
sudo docker run -d --init --platform linux/386 \
  --name earthworm-diam --restart unless-stopped --network host \
  -v /home/julian/earthworm-diam-container-logs:/opt/earthworm/run_diam/log \
  earthworm-diam:gcf-test
```

## 17. Reversión

Para retirar el nuevo adquisidor sin afectar ringserver:

```bash
sudo docker stop -t 30 earthworm-diam
```

Esto deja intactos:

- el ringserver existente;
- su búfer histórico;
- la imagen Docker;
- los logs persistentes;
- los archivos del proyecto.

Para volver a activarlo:

```bash
sudo docker start earthworm-diam
```

## 18. Problemas comunes

### `Unknown version of scream protocol`

El puerto entrega GCF/MSS y no Scream. Use `gcf2ew`, no `scream2ew`.

### `sock_open(): can't get tcp protocol entry`

La imagen carece de `/etc/protocols`. Instale `netbase` dentro de la imagen y reconstruya.

### `/bin/ps: not found`

Instale `procps` dentro de la imagen.

### `gcf2ew` no se genera con `make unix`

Compile explícitamente:

```bash
cd /opt/earthworm/earthworm_8.0/src/data_sources/gcf2ew
make -f makefile.unix
```

### Errores de declaración implícita al compilar gcf2ew

El módulo es heredado. En esta instalación se retiraron únicamente:

```text
-Werror=implicit-function-declaration
-Werror=int-conversion
```

Las advertencias restantes deben conservarse en el log de compilación.

### gcf2ew queda `Zombie`

```bash
sudo docker restart -t 30 earthworm-diam
```

Luego revise causa, procesos, conexiones y logs. La política Docker solamente supervisa directamente el proceso principal; se recomienda añadir posteriormente un `HEALTHCHECK`.

### Pérdida de bloques

```bash
grep -E 'missed|lost|Total Blocks' \
  /home/julian/earthworm-diam-container-logs/gcf2ew*.log | tail -50
```

Una pérdida sostenida puede indicar problemas de red, entrega por ráfagas, búfer del DM24 o competencia con otro cliente/NAM.

## 19. Criterios de aceptación

La instalación se considera correcta cuando:

1. `earthworm-diam` está `running`.
2. `startstop`, `gcf2ew` y `ew2ringserver` están activos.
3. Las conexiones a `172.16.18.87:1568` y `127.0.0.1:16000` están `ESTAB`.
4. `sniffring` recibe `TYPE_TRACEBUF2`.
5. `sniffwave` muestra `DIAM.HHE.CM.00`, `DIAM.HHN.CM.00` y `DIAM.HHZ.CM.00`.
6. `slinktool -Q` muestra los tres streams en `127.0.0.1:18000`.
7. Los tiempos finales avanzan.
8. No hay pérdidas sostenidas ni reinicios inesperados.

## 20. Referencias

- Código fuente de Earthworm: <https://gitlab.com/seismic-software/earthworm>
- Comandos de gcf2ew: <https://folkworm.ceri.memphis.edu/ew-doc/cmd/cmd_gcf2ew.html>
- Comandos de ew2ringserver: <https://folkworm.ceri.memphis.edu/ew-doc/cmd/ew2ringserver_cmd.html>
- Cliente SeedLink slinktool: <https://github.com/EarthScope/slinktool>

