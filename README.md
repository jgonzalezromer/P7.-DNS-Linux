<h1>
<p align=center>
P7. DNS
</p>
</h1>
<h3>
<p align=center>
Juan Gabriel González Romero
</p>
</h3>

---
## Organización
Para comezar falaremos en como se divide este proyecto, onde teremos por unha parte o arquivo principal o cal chamaremos `docker-compose.yml`, este é un arquivo de configuración que define e executa aplicacions multi-contenedor de Docker, nel especificamos como se debe construir e executar os contenedores. 

Por outra parte teremos dous directorios principais que serviran como volumes como manda a [setup da imaxe](https://hub.docker.com/r/internetsystemsconsortium/bind9). Os cales serán `/etc/bind`, `/var/cache/bind` e `/var/lib/bind`, os cales podremos crear rapidamente usando o comando:
```
mkdir -p ./etc/bind ./var/cache/bind ./var/lib/bind
```
A rama de directorios de `/etc` serven para a configuración de BIND e a rama de `/var` serve para o espazo de traballo do funcionamento de BIND.

---
## Permisos
Antes de pasar a crear os documentos de configuración faremos o comando:
```
sudo chown -R 100:100 ./etc/bind/ ./var/cache/bind
```
O cal fai que os donos dos directorios e subdirectores sinalados cambien a o propietario e grupo 100.

Isto xa que asi nos aseguramos que BIND teña os permisos necesarios para funcionar dentro dun contenedor de Docker, o cal é clave para o funcionamento e a seguridade.

---
## docker-compose.yml
Neste docker-compose seguiremos as seguintes directrices:
```
Configurar un sistema cun servidor DNS e un cliente alpine que cumpla os seguintes requisitos.

Volumen por separado da configuración
Red propia interna para tódo-los contenedores
IP fixa no servidor
Configurar Forwarders
Cliente con ferramientas de rede
```
Para cumprir isto facemos o seguinte arquivo:
```
services:
  #Servizos que se executarán no contenedor
  DNS: #Como queremos chamar nos o servizo
    image: internetsystemsconsortium/bind9:9.18 #Imaxe que utilizaremos no contenedor
    container_name: DNS_P7 #Nome específico do contenedor
    ports:
      #Mapeo dos portos host:contenedor
      - 54:53/udp
      - 54:53/tcp
      - 127.0.0.1:953:953/tcp #Neste só pertmite conexións desde localhost
    volumes:
      #Volumes onde se montará o contenedor
      - ./etc/bind:/etc/bind
      - ./var/cache/bind:/var/cache/bind
      - ./var/lib/bind:/var/lib/bind
    restart: always #Esta opción indica que ó contenedor debe reiniciarse en ciertas condiciones
    networks:
      P7_network: #Rede que utilizará
        ipv4_address: 172.18.0.1 #IP que utilzará
  Cliente:
    image: alpine #Imaxe do cliente
    container_name: Cliente_P7 #Nome do container
    tty: true 
    stdin_open: true
    command: /bin/sh -c "apk update && apk add bind-tools" #Comando para que instale dig ao iniciar os contenedores
    networks:
      P7_network:  #Rede que utilizará
        ipv4_address: 172.18.0.2 #IP que utilizará
    dns:
      - 172.18.0.1 #IP do DNS
networks:
  P7_network: #Creamos unha rede tipo bridge
    driver: bridge
    ipam:
      config:
        - subnet: 172.18.0.0/16
          ip_range: 172.18.0.0/24
          gateway: 172.18.0.254
```

---
## named.conf
O named.conf o utilizaremos para chamar a outros archivos(como se fora unha función main), para que así quede todo maís ordenado:
```
include "/etc/bind/named.conf.options";
include "/etc/bind/named.conf.local";
```
> [!IMPORTANT]
> Para ver a estructura dos documentos e os arquivos en sí están neste mesmo repositorio
---
### named.conf.options
Este arquivo sirve para indicar as opcións do servidor DNS nel introduciremos o codigo:
```
options {
        directory "/var/cache/bind/"; #Directorio onde se almacenarán os datos de caché
        dnssec-validation no;
        forwarders {
                8.8.8.8; #DNS de Google
                1.1.1.1; #DNS de Cloudflare
         };
         forward only; #Indicamos que as consultas que non podan resolverse de una forma local serán reenviadas aos servidores indicados arriba
        listen-on { any; }; #O servidor escoitará todas as interface IPv4 dispoñibles no servidor
        listen-on-v6 { any; }; #O servidor escoitará todas as interface IPv6 dispoñibles no servidor
        allow-query{
                any;
        }; #Calquera dispositivo pode realizar consultas DNS
};
```
> [!WARNING]
> No comando `allow-query` non é recomendable por `any` nun entorno real
### named.conf.local
Este arquivo define a zona que terá o servidor DNS:

---
### named.conf.local
```
zone "practica7.int" {
        type master; #Indicamos que o servidor será o principal na zona
        file "/var/lib/bind/db.practica7.int"; Ruta ao ficheiro que contén os rexistros DNS na zona
        allow-query {
                any;
                };
        };  #Calquera dispositivo pode realizar consultas DNS
```
> [!WARNING]
> No comando `allow-query` non é recomendable por `any` nun entorno real

---
## db.practica7.int
Este arquivo almacenará a información de mapeo dos nomes das direccions IPs:
```
$TTL 86400 ; # Tiempo de vida (TTL) de los registros en segundos
@ IN SOA ns.practica7.in some.email.address. (
2024111401 ; Serial #número de versión del archivo de zona
3600 ; Refresh #tiempo que un secundario espera para actualizar la zona
1800 ; Retry #tiempo de espera en caso de fallo para reintentar
36000 ; Expire #tiempo después del cual la zona se considera no válida
86400 ; Minimum TTL #tiempo mínimo de caché para registros negativos
)
@ IN NS ns.practica7.int. # NS: Servidor de nombres de la zona
ns IN A 172.18.0.1 # A: Dirección IP del servidor de nombres principal
www IN A 172.18.0.3 # A: Dirección IP del servidor www.practica7.int.
alias IN CNAME www # CNAME: Alias de www, permite que alias.practica7.int apunte a www.practica7.int.
texto IN TXT "Rexistro TXT de exemplo" # TXT: Registro de texto para información adicional de la zona.
```

---
## Comprobación
Iniciaremos os contenedores co comando `docker compose up -d` e logo entrar no cliente con `docker exec -it Cliente_p7 /bin/sh`.
Usamos o comando: `dig @172.18.0.1 practica7.int`.

Este serviranos para que `dig` faga a consulta ao dominio `asircastelao.int` usando o servidor que lle indicamos, `172.18.0.1`. Unha resposta típica sería: 
```
; <<>> DiG 9.18.31 <<>> @172.18.0.1 practica7.int
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 31289
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 106b97c7fb094b0701000000673644b67358f8270c8d4905 (good)
;; QUESTION SECTION:
;practica7.int.			IN	A

;; AUTHORITY SECTION:
practica7.int.		86400	IN	SOA	ns.practica7.in.practica7.int. some.email.address. 2024111401 3600 1800 36000 86400

;; Query time: 0 msec
;; SERVER: 172.18.0.1#53(172.18.0.1) (UDP)
;; WHEN: Thu Nov 14 18:43:02 UTC 2024
;; MSG SIZE  rcvd: 153
```

---