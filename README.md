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
  DNS:
    image: internetsystemsconsortium/bind9:9.18
    container_name: DNS_P7
    ports:
      - 54:53/udp
      - 54:53/tcp
      - 127.0.0.1:953:953/tcp
    volumes:
      - ./etc/bind:/etc/bind
      - ./var/cache/bind:/var/cache/bind
      - ./var/lib/bind:/var/lib/bind
    restart: always
    networks:
      P7_network:
        ipv4_address: 172.18.0.1
  Cliente:
    image: alpine
    container_name: Cliente_P7
    tty: true
    stdin_open: true
    command: /bin/sh -c "apk update && apk add bind-tools"
    networks:
      P7_network:
        ipv4_address: 172.18.0.2
    dns:
      - 172.18.0.1
networks:
  P7_network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.18.0.0/16
          ip_range: 172.18.0.0/24
          gateway: 172.18.0.254
```

---
## named.conf
```
include "/etc/bind/named.conf.options";
include "/etc/bind/named.conf.local";
```

---
### named.conf.options
```
options {
        directory "/var/cache/bind/";
        dnssec-validation no;
        forwarders {
                8.8.8.8;
                1.1.1.1;
         };
         forward only;
        listen-on { any; };
        listen-on-v6 { any; };
        allow-query{
                any;
        };
};
```

---
### named.conf.local
```
zone "practica7.int" {
        type master;
        file "/var/lib/bind/db.practica7.int";
        allow-query {
                any;
                };
        };
```

---
## db.practica7.int
```
$TTL 86400 ;
@ IN SOA ns.practica7.in some.email.address. (
2024111401 ; Serial
3600 ; Refresh
1800 ; Retry
36000 ; Expire
86400 ; Minimum TTL
)
@ IN NS ns.practica7.int.
ns IN A 172.18.0.1
www IN A 172.18.0.3
alias IN CNAME www
texto IN TXT "Rexistro TXT de exemplo"
```

---
## Comprobación
`dig @172.18.0.1 practica7.int'
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