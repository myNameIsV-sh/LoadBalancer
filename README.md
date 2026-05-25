# Configurando um Load Balancer utilizando Nginx
Um balanceador de carga ou *load balancer* é uma ferramenta/técnica utilizada para redirecionar o acesso de múltiplos clientes à um determinado recurso disponível no servidor, evitando que o este sofra com perca de desempenho ou inatividade por conta de um alto tráfego de dados ou por excesso de requisições. Ele atua como um intermediário recebendo todas as requisições dos clientes e redirecionando-as à um conjunto de nós (*nodes*) conectados ao balanceador de carga, por meio de algoritmos como o *Round-Robin* é possível distribuir a carga as requisições igualmente servidores nós.

Para configurarmos um *pool de servidores*, utilizamos o **contexto** `upstream` e dentro desse contexto adicionamos todos os servidores disponíveis que atuarão como réplicas da nossa aplicação. Podemos também substituir o algoritmo de balanceamento de carga que será utilizado, por padrão o Nginx utiliza o `round-robin`, não sendo necessário defini-lo dentro do contexto, além dele existem os algoritmos `least-connected` e `ip-hash`. O funcionamento desses algoritmos está fora do objetivo desse tutorial, então compreenda o funcionamento e entenda os seus *trade-offs*.

Uma boa prática é sempre manter um número **impar** de servidores para garantir algum nível de redundância para a nossa aplicação, é interessante começar com o número mínimo de **três** e ir aumentando de acordo com a sua necessidade. Um exemplo seria esse:
```nginx
    upstream myapp1 {
	    # ip-hash
        server srv1.example.com;
        server srv2.example.com;
        server srv3.example.com;
    }
```

```nginx
upstream react_backend {
    server 192.168.1.10:3000;
    server 192.168.1.11:3000;
    server 192.168.1.12:3000;
}

server {
    listen 80;
    server_name _;

    location / {
        root /var/www/react-app;
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://react_backend;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host $host;
        proxy_redirect off;
    }

    location ~ ^/app/ {
        proxy_pass http://react_backend;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host $host;
    }
}
```

### Referências
GEEKSFORGEEKS. Using Nginx as HTTP Load Balancer. Disponível em: https://www.geeksforgeeks.org/devops/using-nginx-as-http-load-balancer/. Acesso em: 24 maio 2026.

NGINX. Load balancing. Disponível em: https://nginx.org/en/docs/http/load_balancing.html. Acesso em: 20 maio 2026.

NGINX. HTTP load balancer. Disponível em: https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/. Acesso em: 20 maio 2026.
