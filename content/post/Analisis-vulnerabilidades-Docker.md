+++
categories = ["docker"]
date = "2018-07-11T20:41:27+02:00"
title = "Análisis de vulnerabilidades en Docker: CoreOS Clair y Clair-scanner"

+++

## Análisis de vulnerabilidades en Docker

Prueba de concepto sobre el análisis de vulnerabilidades en imágenes docker

![Docker](/blog/images/docker-secure-or-not.jpg)

En este primer post sobre seguridad en contenedores voy a crear un tutorial básico de cómo analizar una imagen docker usando [CoreOS Clair](https://github.com/coreos/clair) y [Clair-scanner](https://github.com/arminc/clair-scanner). En artículos futuros lo integraremos en un entorno de CI/CD.

El entorno está creado por contenedores para no necesitar instalaciones/dependencias y se compone de:
+ Base de datos PostgreSQL para almacenar la información de las vulnerabilidades (CVE).
+ Servicio Clair que analiza las capas que componen la imagen docker.
+ Cliente Clair-scanner que manda las tareas de análisis a Clair y devuelve el resultado.

La limitación de esta herramienta es que trabaja a nivel de paquetería, lo que instalados a mano como los tarballs no será comprobado.

Pasos a seguir para levantar el escenario:

```
git clone https://github.com/k4mmin/docker-clair-scanner.git docker-clair-scanner
cd $_
```

Usar la última release de [Clair](https://github.com/coreos/clair/releases)(consultar el enlace para modificar al valor correcto) y añadir la contraseña de la base de datos (por defecto 'password').

```
export POSTGRES_PASSWORD=""
sed "s/clair:v2.0.1/clair:v2.0.3/" -i docker-compose.yml && \
sed "s/password=password/password=${POSTGRES_PASSWORD:-password}/" -i clair_config/config.yaml && \
sed "s/POSTGRES_PASSWORD: password/POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-password}/" -i docker-compose.yml
```

Levantar la base de datos y poblarla.

```
docker-compose up -d postgres

curl -L https://gist.githubusercontent.com/BenHall/34ae4e6129d81f871e353c63b6a869a7/raw/5818fba954b0b00352d07771fabab6b9daba5510/clair.sql -o clair_config/clair.sql && \
docker run -it -v $(pwd)/clair_config:/sql/ --network "docker_scan" --link clair_postgres:clair_postgres \
postgres:latest bash -c "PGPASSWORD=${POSTGRES_PASSWORD:-password} psql -h clair_postgres -U postgres < /sql/clair.sql"
```

Levantar el servidor Clair

```
docker-compose up -d clair
```

Crear la imagen de clair-scanner

```
make build
```

Ejecutar clair-scanner con la id de una o varias imágenes docker para obtener el resultado

```
source scan.sh 'image_id'
```

Ejemplo con las imágenes centos:6 y debian:stretch
```
source scan.sh 70b5d81549ec
2018/07/10 18:51:26 [INFO] ▶ Start clair-scanner
2018/07/10 18:51:31 [INFO] ▶ Server listening on port 9279
2018/07/10 18:51:31 [INFO] ▶ Analyzing 3cdae3dd0537787ca73f7603e5b823f886bff78679a28e53df412cc18f011c23
2018/07/10 18:51:32 [INFO] ▶ Image [70b5d81549ec] contains NO unapproved vulnerabilities
```

```
source scan.sh 8626492fecd3
[!] scanning debian_stretch
2018/07/11 17:37:28 [INFO] ▶ Start clair-scanner
2018/07/11 17:37:29 [INFO] ▶ Server listening on port 9279
2018/07/11 17:37:29 [INFO] ▶ Analyzing e7faacaf1878064b6a1a48940d41f684aef4f57c98f8c213c85e3bbb1829356d
2018/07/11 17:37:29 [WARN] ▶ Image [8626492fecd3] contains 6 total vulnerabilities
2018/07/11 17:37:29 [ERRO] ▶ Image [8626492fecd3] contains 6 unapproved vulnerabilities
+------------+---------------------------+--------------+-----------------+------------------------------------------------------------+
| STATUS     | CVE SEVERITY              | PACKAGE NAME | PACKAGE VERSION | CVE DESCRIPTION                                            |
+------------+---------------------------+--------------+-----------------+------------------------------------------------------------+
| Unapproved | High CVE-2017-12424       | shadow       | 1:4.4-4.1       | In shadow before 4.5, the newusers tool could be           |
|            |                           |              |                 | made to manipulate internal data structures in ways        |
|            |                           |              |                 | unintended by the authors. Malformed input may lead        |
|            |                           |              |                 | to crashes (with a buffer overflow or other memory         |
|            |                           |              |                 | corruption) or other unspecified behaviors. This           |
|            |                           |              |                 | crosses a privilege boundary in, for example, certain      |
|            |                           |              |                 | web-hosting environments in which a Control Panel allows   |
|            |                           |              |                 | an unprivileged user account to create subaccounts.        |
|            |                           |              |                 | https://security-tracker.debian.org/tracker/CVE-2017-12424 |
+------------+---------------------------+--------------+-----------------+------------------------------------------------------------+
| Unapproved | Medium CVE-2016-10228     | glibc        | 2.24-11+deb9u3  | The iconv program in the GNU C Library (aka glibc or       |
|            |                           |              |                 | libc6) 2.25 and earlier, when invoked with the -c option,  |
|            |                           |              |                 | enters an infinite loop when processing invalid multi-byte |
|            |                           |              |                 | input sequences, leading to a denial of service.           |
|            |                           |              |                 | https://security-tracker.debian.org/tracker/CVE-2016-10228 |
+------------+---------------------------+--------------+-----------------+------------------------------------------------------------+
| Unapproved | Medium CVE-2017-12132     | glibc        | 2.24-11+deb9u3  | The DNS stub resolver in the GNU C Library (aka            |
|            |                           |              |                 | glibc or libc6) before version 2.26, when EDNS             |
|            |                           |              |                 | support is enabled, will solicit large UDP responses       |
|            |                           |              |                 | from name servers, potentially simplifying off-path        |
|            |                           |              |                 | DNS spoofing attacks due to IP fragmentation.              |
|            |                           |              |                 | https://security-tracker.debian.org/tracker/CVE-2017-12132 |
+------------+---------------------------+--------------+-----------------+------------------------------------------------------------+
| Unapproved | Low CVE-2017-15670        | glibc        | 2.24-11+deb9u3  | The GNU C Library (aka glibc or libc6) before 2.27         |
|            |                           |              |                 | contains an off-by-one error leading to a heap-based       |
|            |                           |              |                 | buffer overflow in the glob function in glob.c,            |
|            |                           |              |                 | related to the processing of home directories              |
|            |                           |              |                 | using the ~ operator followed by a long string.            |
|            |                           |              |                 | https://security-tracker.debian.org/tracker/CVE-2017-15670 |
+------------+---------------------------+--------------+-----------------+------------------------------------------------------------+
| Unapproved | Negligible CVE-2017-7246  | pcre3        | 2:8.39-3        | Stack-based buffer overflow in the pcre32_copy_substring   |
|            |                           |              |                 | function in pcre_get.c in libpcre1 in PCRE 8.40            |
|            |                           |              |                 | allows remote attackers to cause a denial of               |
|            |                           |              |                 | service (WRITE of size 268) or possibly have               |
|            |                           |              |                 | unspecified other impact via a crafted file.               |
|            |                           |              |                 | https://security-tracker.debian.org/tracker/CVE-2017-7246  |
+------------+---------------------------+--------------+-----------------+------------------------------------------------------------+
| Unapproved | Negligible CVE-2017-11164 | pcre3        | 2:8.39-3        | In PCRE 8.41, the OP_KETRMAX feature in the match function |
|            |                           |              |                 | in pcre_exec.c allows stack exhaustion (uncontrolled       |
|            |                           |              |                 | recursion) when processing a crafted regular expression.   |
|            |                           |              |                 | https://security-tracker.debian.org/tracker/CVE-2017-11164 |
+------------+---------------------------+--------------+-----------------+------------------------------------------------------------+
```

Esta herramienta puede ser muy útil para ser integrada en el desarrollo de nuestros contenedores y para conocer qué imágenes base pueden resultar más seguras.

#### Otras herramientas relacionadas

https://thenewstack.io/assessing-the-state-current-container-security/
