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

Com essa compreensão inicial sobre o tema, já somos capazes de implementar um balanceador de carga simples com identificação de endereços IP. No entanto, em cenários reais e mais complexos, como aplicações web modernas desenvolvidas em *frameworks* como React, é necessário adicionar configurações mais robustas que garantam o funcionamento correto da aplicação distribuída. Na próxima seção veremos a implementação de um balanceador de carga para uma aplicação React.]

No tutorial anterior sobre [Redirecionamento em Aplicações React](https://github.com/myNameIsV-sh/NginxAndTests/blob/main/redirecionamento_react.md), exploramos como o Nginx utiliza a diretiva `try_files` para resolver problemas de roteamento no lado do cliente com React Router. Dessa vez não será diferente, iremos manter essa diretiva, mas agora iremos adicionar **cinco** nós para o balanceamento de carga, manteremos também a identificação dos IPs por meio do `proxy_set_header X-Real-IP` visto anteriormente.

## Balanceando uma aplicação React
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

Agora vamos criar o arquivo `nginx.conf` e adicionar uma inclusão (ou referência) ao `default.conf`, basta adicionar a linha `include /etc/nginx/conf.d/*.conf` no final do arquivo.
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
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    sendfile on;
    tcp_nopush on;
    keepalive_timeout 65;
    gzip on;

    include /etc/nginx/conf.d/*.conf;
}


### Referências
GEEKSFORGEEKS. Using Nginx as HTTP Load Balancer. Disponível em: https://www.geeksforgeeks.org/devops/using-nginx-as-http-load-balancer/. Acesso em: 24 maio 2026.

NGINX. Load balancing. Disponível em: https://nginx.org/en/docs/http/load_balancing.html. Acesso em: 20 maio 2026.

NGINX. HTTP load balancer. Disponível em: https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/. Acesso em: 20 maio 2026.

NGINX. Reverse proxy. Disponível em: https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/. Acesso em: 25 maio 2026.
