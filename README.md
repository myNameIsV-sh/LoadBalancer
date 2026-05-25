# Configurando um Load Balancer utilizando Nginx
Um balanceador de carga ou *load balancer* é uma ferramenta/técnica utilizada para redirecionar o acesso de múltiplos clientes à um determinado recurso disponível no servidor, evitando que o este sofra com perca de desempenho ou inatividade por conta de um alto tráfego de dados ou por excesso de requisições. Ele atua como um intermediário recebendo todas as requisições dos clientes e redirecionando-as à um conjunto de nós (*nodes*) conectados ao balanceador de carga, por meio de algoritmos como o *Round-Robin* é possível distribuir a carga as requisições igualmente servidores nós.

Para configurarmos um grupo, conjunto ou *pool de servidores* (chame do que quiser), utilizamos o **contexto** `upstream` e dentro desse contexto adicionamos todos os servidores disponíveis que atuarão como réplicas da nossa aplicação. Podemos também substituir o algoritmo de balanceamento de carga que será utilizado, por padrão o Nginx utiliza o `round-robin`, não sendo necessário defini-lo dentro do contexto, além dele existem os algoritmos `least_connected` e `ip_hash`. O funcionamento desses algoritmos está fora do objetivo desse tutorial, então compreenda o funcionamento e entenda os seus *trade-offs*.

Uma boa prática é sempre manter um número **impar** de servidores para garantir algum nível de redundância para a nossa aplicação, é interessante começar com o número mínimo de **três** e ir aumentando de acordo com a sua necessidade. Suponhamos que queremos disponibilizar três nós utilizando o algoritmo de hash para redirecionar o tráfego do usuário baseado no seu endereço IP (`ip_hash`), basta escrever dentro do contexto `http`:
```nginx
upstream myapp1 {
    ip_hash;
    server localhost:3001;
    server localhost:3002;
    server localhost:3003;
}
```
Definido o conjunto de nós que serão utilizados para redistribuir a carga, adicionamos logo abaixo uma diretiva chamada `proxy_pass`, essa diretiva permite *"encaminhar requisições para outros servidores"* .

Após definirmos os conjuntos de nós, é preciso encaminhar essas conexões para o `upstream` definido, nesse caso, utiliza-se a diretiva `proxy_pass`.
```nginx
server {
	location / {
		proxy_pass http://myapp1;
	}
}
```
Por enquanto a nossa configuração estaria assim:
```nginx
upstream myapp1 {
    ip_hash;
    server localhost:3001;
    server localhost:3002;
    server localhost:3003;
}

server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://myapp1;
    }
}
```
Podemos ir além, adicionando cabeçalhos de identificação para sermos capazes de determinar qual cliente realizou a requisição; utilizamos a diretiva `proxy_set_header`. Com o cabeçalho `X-Real-IP`, é possível rastrear o endereço IP real do cliente que originou a requisição. O símbolo `$` representa uma variável do Nginx, neste caso `$remote_addr` retorna o endereço IP do cliente remoto, como é demonstrado no exemplo abaixo:
```nginx
upstream myapp1 {
    ip_hash;
    server localhost:3001;
    server localhost:3002;
    server localhost:3003;
}

server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://myapp1;
		# X-Real-IP: 192.168.1.100
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```
Todas essas configurações são feitas no arquivo de configuração do Nginx. Em instalações tradicionais, este arquivo é localizado em `/etc/nginx/conf.d/` e deve ser nomeado com a extensão `.conf` (ex.: `load-balancer.conf`). Após criar ou editar o arquivo, valide a sintaxe com `nginx -t` e recarregue as configurações utilizando o comando `nginx -s reload` ou `systemctl reload nginx`.

>[!NOTE] 
Em ambientes com Docker, o arquivo de configuração é copiado para dentro do contêiner durante a construção da imagem. Caso prefira utilizar um servidor Nginx em uma instalação tradicional, salve o arquivo em `/etc/nginx/sites-available/`, crie um link simbólico em `/etc/nginx/sites-enabled/` com o comando `ln -s /etc/nginx/sites-available/load-balancer /etc/nginx/sites-enabled/load-balancer`, valide a sintaxe com `nginx -t` e recarregue o serviço com `systemctl reload nginx`.

Com essa compreensão inicial sobre o tema, já somos capazes de implementar um balanceador de carga simples com identificação de endereços IP. No entanto, em cenários reais e mais complexos, como aplicações web modernas desenvolvidas em *frameworks* como React, é necessário adicionar configurações mais robustas que garantam o funcionamento correto da aplicação distribuída. Na próxima seção veremos a implementação de um balanceador de carga para uma aplicação React.

## Balanceando uma aplicação React
No tutorial anterior sobre [Redirecionamento em Aplicações React](https://github.com/myNameIsV-sh/NginxAndTests/blob/main/redirecionamento_react.md), exploramos como o Nginx utiliza a diretiva `try_files` para resolver problemas de roteamento no lado do cliente com React Router. Dessa vez não será diferente, iremos manter essa diretiva, mas agora iremos adicionar **cinco** nós para o balanceamento de carga, manteremos também a identificação dos IPs por meio do `proxy_set_header X-Real-IP` visto anteriormente.

Primeiramente, iremos realizar o *build* da nossa aplicação React por meio do `npm run build`, para esse tutorial, irei utilizar o projeto React feito pelo o meu professor [Rhavy Maia](https://github.com/rhavymaia):
```bash
git clone https://github.com/rhavymaia/minhaappweb20252
cd minhaappweb20252
```
Em seguida  instalamos todas as dependências e realizamos o *build* da aplicação:
```bash
npm install && npm run build
```
Após a execução, você terá um diretório `dist/` (ou `build/`, dependendo da configuração do projeto) contendo todos os arquivos estáticos compilados da sua aplicação React. Este diretório será copiado para dentro dos contêineres Docker dos nós que executarão a aplicação. Você verá algo similar à isso:
```text
v@hp-256r-g9:~/Projetos/LoadBalancer/minhaappweb20252$ ls -lh
total 176K
drwxrwxr-x   3 v v 4,0K mai 25 09:01 dist
-rw-rw-r--   1 v v  763 mai 25 09:01 eslint.config.js
-rw-rw-r--   1 v v  374 mai 25 09:01 index.html
-rw-rw-r--   1 v v  430 mai 25 09:01 Jenkinsfile
drwxrwxr-x 182 v v 4,0K mai 25 09:01 node_modules
-rw-rw-r--   1 v v  944 mai 25 09:01 package.json
-rw-rw-r--   1 v v 137K mai 25 09:01 package-lock.json
-rw-rw-r--   1 v v 1,2K mai 25 09:01 README.md
drwxrwxr-x   7 v v 4,0K mai 25 09:01 src
-rw-rw-r--   1 v v  161 mai 25 09:01 vite.config.js
```
Com o projeto pronto, agora podemos focar nos arquivos de configuração do Nginx, fora do contêiner, vamos criar um arquivo `default.conf` para configurar o servidor com o `upstream`
```nginx
upstream react_app {
    server localhost:3001;
    server localhost:3002;
    server localhost:3003;
    server localhost:3004;
    server localhost:3005;
}

server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://react_app;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```
>[!NOTE]
>Além do `X-Real-IP`, existem outros cabeçalhos úteis para requisições de proxy: `X-Forwarded-For $proxy_add_x_forwarded_for` mantém um histórico de todos os IPs pelos quais a requisição passou, `X-Forwarded-Proto $scheme` preserva o protocolo original (HTTP ou HTTPS) da requisição, e `Host $host` garante que o cabeçalho `Host` original seja encaminhado aos nós backend. Estes podem ser adicionados conforme necessário em cenários mais complexos.

Agora vamos criar o arquivo `nginx.conf` e adicionar uma inclusão (ou referência) ao `default.conf`, basta adicionar a linha `include /etc/nginx/conf.d/*.conf` no final do arquivo. Na configuração do `log` é interessante adicionarmos a variável `$upstream_addr` para verificarmos os endereços dos nós que receberam a requisição.
```nginx
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"'
                    'to: $upstream_addr';

    access_log /var/log/nginx/access.log main;

    sendfile on;
    tcp_nopush on;
    keepalive_timeout 65;
    gzip on;

    include /etc/nginx/conf.d/*.conf;
}
```
Com os arquivos de configuração prontos, agora podemos ir para a criação dos contêineres. Vamos criar um contêiner principal que será o balanceador de carga; nada mais justo que chamá-lo de `load-balancer`, esse contêiner terá a porta 80 mapeada e também terá os arquivos de configuração que escrevemos há pouco também mapeados; é interessante manter esses arquivos de configuração restrito à apenas leitura com o parâmetro `:ro`,  dessa forma o comando fica assim:
```
docker run -d --name load-balancer -p 80:80 \ -v /caminho/para/nginx.conf:/etc/nginx/nginx.conf:ro \ -v /caminho/para/default.conf:/etc/nginx/conf.d/default.conf:ro \ nginx:latest
```
Agora iremos criar os nossos nós, o processo de criação é semelhante, mas agora iremos enumerar as portas sequencialmente para que possamos identificar com mais facilidade e serão esses nós que vão servir o conteúdo presente em `dist/`.
```text
docker run -d --name react-node-1 -p 3001:80 \
  -v /caminho/para/dist:/usr/share/nginx/html:ro \
  nginx:latest

docker run -d --name react-node-2 -p 3002:80 \
  -v /caminho/para/dist:/usr/share/nginx/html:ro \
  nginx:latest

docker run -d --name react-node-3 -p 3003:80 \
  -v /caminho/para/dist:/usr/share/nginx/html:ro \
  nginx:latest

docker run -d --name react-node-4 -p 3004:80 \
  -v /caminho/para/dist:/usr/share/nginx/html:ro \
  nginx:latest

docker run -d --name react-node-5 -p 3005:80 \
  -v /caminho/para/dist:/usr/share/nginx/html:ro \
  nginx:latest
```
Ao final do procedimento, você terá os seguintes contêineres:
```
v@hp-256r-g9:~/Projetos/Faculdade/LoadBalancer/test$ docker ps -a
CONTAINER ID   IMAGE           COMMAND                  CREATED          STATUS          PORTS                                     NAMES
e7730ff67214   nginx:latest    "/docker-entrypoint.…"   2 seconds ago    Up 1 second     0.0.0.0:3005->80/tcp, [::]:3005->80/tcp   react-node-5
ad48c483bf47   nginx:latest    "/docker-entrypoint.…"   5 seconds ago    Up 5 seconds    0.0.0.0:3004->80/tcp, [::]:3004->80/tcp   react-node-4
a30019b7d8e6   nginx:latest    "/docker-entrypoint.…"   9 seconds ago    Up 8 seconds    0.0.0.0:3003->80/tcp, [::]:3003->80/tcp   react-node-3
884c3f526d48   nginx:latest    "/docker-entrypoint.…"   12 seconds ago   Up 12 seconds   0.0.0.0:3002->80/tcp, [::]:3002->80/tcp   react-node-2
e5d8f0ff2828   nginx:latest    "/docker-entrypoint.…"   15 seconds ago   Up 15 seconds   0.0.0.0:3001->80/tcp, [::]:3001->80/tcp   react-node-1
b3a28900d2f8   nginx:latest    "/docker-entrypoint.…"   23 seconds ago   Up 23 seconds   0.0.0.0:80->80/tcp, [::]:80->80/tcp       load-balancer
```
Com tudo pronto, agora podemos realizar os nossos testes, iremos utilizar o `curl` para isso:
```text
v@hp-256r-g9:~$ curl http://localhost:8080
<!doctype html>
<html lang="pt-br">

<head>
  <meta charset="UTF-8" />
  <link rel="icon" type="image/svg+xml" href="/vite.svg" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Instituições de Ensino - Censo Escolar</title>
  <script type="module" crossorigin src="/assets/index-BgEsQYJp.js"></script>
  <link rel="stylesheet" crossorigin href="/assets/index-DcCezYBB.css">
</head>

<body>
  <div id="root"></div>
</body>

</html>
```
Em um outro terminal, vamos utilizar o comando `docker logs` para monitorar todas as requisições feitas ao `load-balancer`, como resultado teremos essa saída aqui:
```text
v@hp-256r-g9:~$ docker logs -f load-balancer
172.21.0.1 - - [25/May/2026:13:42:19 +0000] "GET / HTTP/1.1" 200 470 "-" "curl/8.14.1" "-" to: 172.21.0.3:80
172.21.0.1 - - [25/May/2026:13:42:19 +0000] "GET / HTTP/1.1" 200 470 "-" "curl/8.14.1" "-" to: 172.21.0.4:80
172.21.0.1 - - [25/May/2026:13:42:19 +0000] "GET / HTTP/1.1" 200 470 "-" "curl/8.14.1" "-" to: 172.21.0.5:80
172.21.0.1 - - [25/May/2026:13:42:19 +0000] "GET / HTTP/1.1" 200 470 "-" "curl/8.14.1" "-" to: 172.21.0.6:80
172.21.0.1 - - [25/May/2026:13:42:42 +0000] "GET / HTTP/1.1" 200 470 "-" "curl/8.14.1" "-" to: 172.21.0.7:80
172.21.0.1 - - [25/May/2026:13:42:43 +0000] "GET / HTTP/1.1" 200 470 "-" "curl/8.14.1" "-" to: 172.21.0.3:80
172.21.0.1 - - [25/May/2026:13:42:43 +0000] "GET / HTTP/1.1" 200 470 "-" "curl/8.14.1" "-" to: 172.21.0.4:80
172.21.0.1 - - [25/May/2026:13:42:43 +0000] "GET / HTTP/1.1" 200 470 "-" "curl/8.14.1" "-" to: 172.21.0.5:80
172.21.0.1 - - [25/May/2026:13:42:43 +0000] "GET / HTTP/1.1" 200 470 "-" "curl/8.14.1" "-" to: 172.21.0.6:80
172.21.0.1 - - [25/May/2026:13:51:40 +0000] "GET / HTTP/1.1" 200 470 "-" "curl/8.14.1" "-" to: 172.21.0.7:80
172.21.0.1 - - [25/May/2026:13:52:12 +0000] "GET / HTTP/1.1" 200 470 "-" "curl/8.14.1" "-" to: 172.21.0.3:80
```
É possível perceber que as os endereços após o `to:` variam de `172.21.0.3` à `172.21.0.7`, exatamente a quantidade de nós que definimos anteriormente.

### Referências
GEEKSFORGEEKS. Using Nginx as HTTP Load Balancer. Disponível em: https://www.geeksforgeeks.org/devops/using-nginx-as-http-load-balancer/. Acesso em: 24 maio 2026.

NGINX. Load balancing. Disponível em: https://nginx.org/en/docs/http/load_balancing.html. Acesso em: 20 maio 2026.

NGINX. HTTP load balancer. Disponível em: https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/. Acesso em: 20 maio 2026.

NGINX. Reverse proxy. Disponível em: https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/. Acesso em: 25 maio 2026.
