**P7-DNS_Linux - Guía de Configuración de un Servidor DNS con Docker**

Para configurar correctamente un servidor DNS utilizando Docker, seguimos una serie de pasos que incluyen la creación de un archivo `docker-compose.yml`, configuración de los archivos de DNS y zonas, y validación mediante herramientas como `dig`. A continuación se detallan todos los pasos necesarios.

### **1. Creación del archivo `docker-compose.yml`**

El archivo `docker-compose.yml` es esencial para definir los servicios necesarios para ejecutar el servidor DNS y el cliente que hará las consultas DNS.

```yaml
services:
  bind9:
    container_name: Luismiserver
    image: internetsystemsconsortium/bind9:9.18
    platform: linux/amd64
    ports:
      - 55:53/tcp
      - 55:53/udp
    networks:
      alex_subnet:
        ipv4_address: 172.30.8.1
    volumes:
      - ./conf:/etc/bind
      - ./zonas:/var/lib/bind
    restart: always

  cliente:
    container_name: cliente2
    image: alpine
    platform: linux/amd64
    tty: true
    stdin_open: true
    dns:
      - 172.30.8.1  # IP del servidor DNS
    networks:
      luismi_subnet:
        ipv4_address: 172.30.8.2

networks:
  luismi_subnet:
    driver: bridge
    ipam:
      config:
        - subnet: 172.30.0.0/16
          ip_range: 172.30.8.0/24
          gateway: 172.30.8.254
```

En este archivo:

- **`bind9`**: Es el servidor DNS que utiliza la imagen de `bind9` en su versión 9.18. Los puertos 53 TCP y UDP se exponen en el contenedor, redirigiéndolos a los puertos 55 para evitar conflictos con otros servicios en la máquina host. Se asigna una IP fija `172.30.8.1` al servidor DNS.
- **`cliente`**: Es un contenedor basado en la imagen `alpine`. Configuramos la IP `172.30.8.2` y la dirección IP del servidor DNS `172.30.8.1` para que el cliente utilice este servidor para realizar las consultas.

La red `luismi_subnet` está configurada con el driver `bridge` y una subred `172.30.0.0/16`. El rango de IPs es limitado a `172.30.8.0/24`, y el gateway es `172.30.8.254`.

### **2. Archivos de Configuración de BIND**

Dentro de la carpeta `conf`, creamos varios archivos de configuración que son necesarios para que BIND funcione correctamente.

#### **2.1. Archivo `named.conf.local`**

Este archivo configura las zonas de DNS. Aquí se especifica el archivo de zona para el dominio `asircastelao.int`.

```bash
zone "asircastelao.int" {
    type master;
    file "/etc/bind/db.asircastelao.com";   # Archivo con los registros DNS de la zona
};
```

#### **2.2. Archivo `named.conf.options`**

Este archivo contiene las opciones globales de BIND, como las opciones de reenvío y la validación de DNSSEC.

```bash
options {
    directory "/var/cache/bind";
    recursion yes;                      # Permitir la resolución recursiva
    allow-query { any; };               # Permitir consultas desde cualquier IP
    dnssec-validation no;               # Permitir realizar actualizaciones sin problemas
    forwarders {
        8.8.8.8;                        # Google DNS
        1.1.1.1;                        # Cloudflare DNS
    };
    listen-on { any; };                 # Escuchar en todas las interfaces
    listen-on-v6 { any; };
};
```

#### **2.3. Archivo `named.conf`**

Este archivo enlaza las configuraciones anteriores.

```bash
include "/etc/bind/named.conf.options";
include "/etc/bind/named.conf.local";
```

### **3. Archivo de Zona**

El archivo de zona `db.asircastelao.int` contiene los registros DNS para el dominio `asircastelao.int`. 

#### **3.1. Contenido del archivo `db.asircastelao.int`**

```bash
$TTL    604800              # TTL (Time To Live) predeterminado en segundos

@       IN      SOA     ns.asircastelao.int. admin.asircastelao.int. (
                        2         # Serial: número de serie del archivo de zona.

                        604800    # Refresh: intervalo de actualización en segundos 

                        86400     # Retry: tiempo de reintento en segundos (24 horas).

                        2419200   # Expire: tiempo de expiración en segundos 

                        604800 )  # Negative Cache TTL: tiempo en segundos para almacenar en caché las respuestas negativas

@       IN      NS      ns.asircastelao.int.       # Define el servidor de nombres (NS) para la zona asircastelao.int.
ns       IN      A       172.30.8.1                 # IP del servidor de nombres

test    IN      A       172.30.8.4                 # Subdominio test.asircastelao.int apuntando a la IP 172.30.8.4
alias   IN      CNAME   web.asircastelao.com.       # Alias que apunta a web.asircastelao.com

texto   IN      TXT     "Este es un registro de texto para asircastelao.com"   # Registro de texto
```

### **4. Comprobación**

Una vez que todos los archivos estén configurados y el servidor esté listo, podemos iniciar los contenedores y realizar una prueba para verificar que todo esté funcionando correctamente.

#### **4.1. Iniciar el servidor y los contenedores**

Primero, levanta los contenedores con el siguiente comando:

```bash
docker compose up -d
```

Esto ejecutará los contenedores en segundo plano.

#### **4.2. Acceder al contenedor cliente**

Una vez que los contenedores estén en funcionamiento, entra en el contenedor cliente con:

```bash
docker exec -it cliente2 /bin/sh
```

#### **4.3. Instalar herramientas de DNS**

Dentro del contenedor cliente, instala las herramientas de BIND necesarias para realizar consultas DNS:

```bash
apk update
apk add --update bind-tools
```

#### **4.4. Realizar una consulta con `dig`**

Ya con `dig` instalado, puedes realizar una consulta al servidor DNS:

```bash
dig @172.30.8.1 ejemplo.asircastelao.int
```

Si todo está configurado correctamente, deberías recibir una respuesta similar a la siguiente:

```bash
; <<>> DiG 9.18.27 <<>> @172.30.8.1 ejemplo.asircastelao.int
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 59946
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 444f62cfab6bd7aa010000006733ad5dd371b9b09805231e (good)
;; QUESTION SECTION:
;ejemplo.asircastelao.int.	IN	A

;; AUTHORITY SECTION:
.			900	IN	SOA	a.root-servers.net. nstld.verisign-grs.com. 2024111201 1800 900 604800 86400

;; Query time: 31 msec
;; SERVER: 172.30.8.1#53(172.30.8.1) (UDP)
;; WHEN: Tue Nov 12 19:32:45 UTC 2024
;; MSG SIZE  rcvd: 157
```

El código `NXDOMAIN` significa que el dominio no está registrado, lo que es normal si aún no se han agregado registros A para el subdominio `ejemplo.asircastelao.int`. A medida que agregues más registros a la zona, las consultas a estos dominios deben devolver respuestas correctas.

### **Conclusión**

Con estos pasos, hemos configurado un servidor DNS en Docker usando BIND9. También hemos probado la configuración utilizando un cliente en Alpine Linux y hemos confirmado que las consultas DNS se resuelven correctamente.