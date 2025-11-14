# Load Balancer com Nginx + 5 Nós (Round Robin)

Este projeto implementa um Load Balancer utilizando Nginx, configurado com 5 nós e utilizando o algoritmo Round Robin.
O balanceador distribui as requisições entre os contêineres que hospedam a aplicação ReactJs. Toda a configuração é realizada utilizando comandos básicos do Docker, sem o uso de Docker Compose.

---

## Estrutura do Repositório
```bash
[NOME-DA-APLICACAO]-loadbalancer/
│── build/              # Aplicação React compilada
│── nginx.conf          # Arquivo global de configuração do Nginx (Load Balancer)
│── default.conf        # Load Balancer (upstream)
│── README.md           # Documentação do projeto
```

---

## Pré-requisitos
Antes de iniciar:
* Docker instalado
* Aplicação React já compilada
```bash
cd minha-aplicacao
npm run build
```
* Copiar a pasta build para dentro do repositório

---

## Configurações do Nginx
### 1. Arquivo nginx.conf
Usado como configuração global do serviço Nginx.
```bash

user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$http_x_real_ip - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;
}
```

### 2. Arquivo default.conf
Usado para configurar o Load Balancer + Round Robin + IP real do cliente.
```bash
#log_format upstreamlog '$server_name to: $upstream_addr [$request] '
#	'$remote_addr [$time_local] ' 
#	'upstream_response_time $upstream_response_time '
#	'msec $msec request_time $request_time' ;

upstream nodes{
	server 172.17.0.2;
	server 172.17.0.3;
	server 172.17.0.4;
	server 172.17.0.5;
	server 172.17.0.6; 
}

server {
	
	server_name healthbase;

	access_log /var/log/nginx/access.log;
	
	location /{
		proxy_pass http://nodes;		
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header Host $host;
		proxy_set_header X-Forwarded-Proto $scheme;
	}
}
```

---

## Subindo os Contêineres Docker
### 1. Criar os 5 nós (servidores React estáticos)
Cada nó serve a pasta build/ com Nginx interno:
```bash
docker run -itd --name node1 -v ~/Documents/GCSI-2025.2/Atividade2/healthbase-loadbalancing/dist:/usr/share/nginx/html nginx:1.29.3-alpine
docker run -itd --name node2 -v ~/Documents/GCSI-2025.2/Atividade2/healthbase-loadbalancing/dist:/usr/share/nginx/html nginx:1.29.3-alpine
docker run -itd --name node3 -v ~/Documents/GCSI-2025.2/Atividade2/healthbase-loadbalancing/dist:/usr/share/nginx/html nginx:1.29.3-alpine
docker run -itd --name node4 -v ~/Documents/GCSI-2025.2/Atividade2/healthbase-loadbalancing/dist:/usr/share/nginx/html nginx:1.29.3-alpine
docker run -itd --name node5 -v ~/Documents/GCSI-2025.2/Atividade2/healthbase-loadbalancing/dist:/usr/share/nginx/html nginx:1.29.3-alpine
```
### 2. Criar container do Load Balancer
Aqui montamos os arquivos nginx.conf e default.conf vindos do host:
```bash
docker run -itd --name loadbalancer -p 81:80 \
  -v ~/Documents/GCSI-2025.2/Atividade2/healthbase-loadbalancing/nginx.conf:/etc/nginx/nginx.conf \
  -v ~/Documents/GCSI-2025.2/Atividade2/healthbase-loadbalancing/default.conf:/etc/nginx/conf.d/default.conf \
  nginx:1.29.3-alpine
```
## Testando o Balanceamento
Execute:
```bash
curl http://localhost
```
Para verificar o Round Robin, consulte os logs:
```bash
docker logs -f loadbalancer
```
Você verá as requisições sendo distribuídas entre os 5 nós.

## Visualizando os IPs reais da requisição
```bash
172.17.0.1 - - [12/Oct/2024:09:21:14 +0000] "GET / HTTP/1.1" 200 532 "-" "Mozilla..." realip=172.17.0.1 forwardedfor=-
```
