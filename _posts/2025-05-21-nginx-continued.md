---
title: Nginx from scratch part 2
categories: ["System Design"]
tags: [web, Devops, proxy, microservices, "load balancer"]
math: true
---
## Cacheing with Nginx
- Nginx supports multiple ways to cache static files and responses

- ### client side cacheing
  - In client side cacheing nginx doesn't store anything, Instead nginx sets headers telling he client browser to cache static files like images, css, js for a period of time
  ```nginx
    server {
      listen 80;
      server_name example.com;

      location / {
          root /var/www/html;
          index index.html;
      }

      # Cache static assets in the browser for 30 days
      location ~* \.(jpg|jpeg|png|gif|ico|css|js|woff2?|ttf|svg|eot)$ {
          root /var/www/html;
          access_log off; # we generally use it on static data to stop logging about the accesed files so that we can focus on dynamic routes or api
          expires 30d;
          add_header Cache-Control "public";
      }
  }
  ```

  - Cache-Control can take the following Directives
    - public: Response can be cached by any cache (browser, CDN, proxy).
    - private: Response can only be cached by the browser, not shared caches.
    - no-cache: Response must be revalidated with the server before using the cache.
    - no-store : Response must not be cached anywhere, ever.
    - max-age=SECONDS: How long (in seconds) the response is considered fresh.
    - s-maxage=SECONDS: Same as `max-age`, but for shared caches (like CDNs or proxies).
    - must-revalidate : Forces revalidation when the cached content expires.
    - proxy-revalidate: Like `must-revalidate`, but for shared proxies.
    - immutable: Tells browsers the content won’t change during its freshness period.

- ### server side Cacheing
  - If the files are from an upstream like a CDN or an SSR application we can inform nginx to cache those content aswell

  ```nginx
  http {
    proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=static_cache:10m inactive=24h max_size=1g;

    server {
        listen 80;

        location {
            proxy_pass http://backend_server;
            proxy_cache static_cache;
            proxy_cache_valid 200 30d;
            proxy_cache_use_stale error timeout updating;
            add_header X-Cache-Status $upstream_cache_status;
        }
    }

    upstream backend_server {
        server 127.0.0.1:8080;
    }
}

  ```

## nginx as a Media server
  - nginx can be used for serving audio / video content with byte range support and optional streaming 
  - The user doesn't have to download the entire video file as the video player client can request the necessary bytes from the media server
  - for example

  ```nginx
server {
  listen 80;

  location /media/ {
    root /var/www;
    types {
      video/mp4 mp4; # these are the mime types that are supposed to be streamed
        video/webm webm;
      audio/mpeg mp3;
    }
    mp4; # this line enables byte range streaming
  }
}
```

## nginx as an API geteway
- nginx is like a central server that lies between clients and servers and routes the traffic appropriately it manages things like rate limiting, authentication, cacheing, modifying headers, load balancing, CORS etc

```nginx
server {
    listen 80;
    server_name api.example.com;

    location /users/ {
        proxy_pass http://user_service;
    }

    location /orders/ {
        proxy_pass http://order_service;
    }

    location /auth/ {
        proxy_pass http://auth_service;
    }

    limit_req zone=api_limit burst=10; # limit 10 requests
}
```

## nginx for handling dynamic content
- This is common, like i said most people know nginx because of nginx, nginx server html files server dynamically by the php-fpm (FastCGI) server, this is a small example on how to sever php files using nginx
```nginx
server {
    listen 80;

    root /var/www/html;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```


## etc
well nginx is a very simple and very powerful software, there are a lot of load balancers out there but nginx will always be special for me, you might wanna try experimenting with nginx, sure you might get a lot of 50x and 40x errors but learning from them is whats important
