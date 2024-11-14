## Practica 7 - Johan Erazo

### Configura un sistema cun servidor DNS e un cliente alpine que cumpla os seguintes requisitos.

Necesitamos primeiro crear unha carpeta para comezar a traballar. Dentro dela faremos os arquivos pertinentes, que serían:

1. **`compose.yml`**: O arquivo principal de configuración de Docker Compose.
2. **Carpeta de zonas**: Onde estará o dominio, que terá o nome a escoller.
3. **Carpeta `conf`**: Aquí incluiránse os arquivos de configuración:
   - **`named.conf`**: Arquivo principal de configuración de BIND.
   - **`named.conf.options`**: Definir as opcións de configuración, como os *forwarders* e a validación DNSSEC.
   - **`named.conf.local`**: Configuración específica local, como as zonas e outras personalizacións.

Todos estes arquivos créanse na mesma carpeta de `conf`.

##### - Volumen por separado da configuración

Os volumes púxenos separados no arquivo compose.yml da seguinte maneira:

version: '3'

services:
  practica7:
    container_name: 7practica
    image: internetsystemsconsortium/bind9:9.18
    platform: linux/amd64
    ports:
      - "54:53/udp"
      - "54:53/tcp"
    networks:
      bind9_subnet:
        ipv4_address: 192.168.23.1
    volumes:
      - ./conf:/etc/bind
      - ./zonas:/var/lib/bind
    restart: always
  cliente:
    container_name: cliente2
    image: alpine:latest
    platform: linux/amd64
    tty: true
    stdin_open: true
    # configuramos para que el cliente use nuestro dns
    dns:
      - 192.168.23.1
    networks:
      bind9_subnet:
        ipv4_address: 192.168.23.2
    
networks:
  bind9_subnet:
    driver: bridge
    ipam:
      config:
        - subnet: 192.168.0.0/16


##### - Red propia interna para tódo-los contenedores

Nas *networks* creo a rede `bind9_subnet` de tipo *bridge*, que é a que permite a comunicación entre os contedores.

Aquí usei a `ipv4_address` para que os contedores teñan a IP fixa.
    networks:
      bind9_subnet:
        ipv4_address: 192.168.23.2

Na subrede especifico un rango de IPs que será usado exclusivamente para os contedores.networks:
  bind9_subnet:
    driver: bridge
    ipam:
      config:
        - subnet: 192.168.0.0/16

##### - ip fixa no servidor

Para a IP fixa do servidor usei a seguinte:
    networks:
      bind9_subnet:
        ipv4_address: 192.168.23.1 

##### - Configurar Forwarders

Na configuración de *Forwarders* vaise ao arquivo `named.conf.options`.

options {
	directory "/var/cache/bind";
	recursion yes;
	allow-query { any; };
	dnssec-validation no;
	forwarders {
	 	8.8.8.8;
		8.8.4.4;
	 };
	listen-on { any; };
	listen-on-v6 { any; };
};

Aquí puxen os DNS de Google nos *forwarders* para resolver calquera dominio que o servidor BIND9 non sexa capaz de resolver.

Engadín `dnssec-validation no;` para poder usar a imaxe alpine, xa que se o activo, o DNSSEC non me deixa realizar *updates* ou *add*.

##### - Crear Zona propia
#####   - Rexistros a configurar: NS, A, CNAME, TXT, SOA

Na carpeta `zonas/db.erazo.castelao.int` engadín todo o requirido.

$TTL 38400	; 10 hours 40 minutes
@		IN SOA	ns.erazo.castelao.int. root.erazo.castelao.int. (
				23         ; serial
				690000     ; refresh (3 hours)
				36000       ; retry (1 hour)
				604800     ; expire (1 week)
				38400      ; minimum (10 hours 40 minutes)
				)
@		IN NS	ns.erazo.castelao.int.
ns		IN A		192.168.23.1
test		IN A		192.168.23.4
alias	IN CNAME	web.erazo.castelao.com.
info	IN TXT		"Server DNS erazo.castelao.int"

Poñín a zona `erazo.castelao.int` que inclúe:

- O NS con `ns.erazo.castelao.int`.
- O rexistro A, onde apunto a `ns` e `web` á IP `192.168.23.1`.
- O CNAME puxen como alias `web.erazo.castelao.com`.
- O TXT puxen o texto informativo.
- O SOA puxen a autoridade e o root.

##### - Cliente con ferramientas de rede

Para comezar a usar o cliente na terminal dentro da carpeta de traballo, arrancamos o comando `docker compose up -d`. Así, o contedor executase en segundo plano e agora accedemos ao cliente con `docker exec -it cliente2 sh`. Con isto, entramos no cliente coa imaxe alpine.

Agora temos que instalar as ferramentas para poder facer as comprobacións, que sería `apk update`, despois `apk add bind-tools`.

Con isto, poderemos realizar as comprobacións con: `dig @192.168.23.1 web.erazo.catelao.com`. Debería saírnos toda a información.