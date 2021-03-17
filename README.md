# Pix Report CSP 

Nous avons besoin de récolter les reports fait par les Content Security Policy - CSP.
Pour cela, nous avons créé un nginx, permettant de logger tous les reports qui lui sont envoyés. 

## Comment ça marche ? 

Les CSP mis en place sur d'autres applications reportent vers ce serveur nginx à l'aide de la directive : `report-uri`. 

### Le format des logs 

Nous loggons directement, le request body au type JSON grâce au `escape=none`.

```conf
log_format CSP escape=none $request_body;
```
Le premier serveur nginx : 
1. Ecoute sur le port passé en environnement. 
2. Logue seulement les requêtes type CSP sur l'uri `/`.
3. Envoie la requête sur un second serveur, car le fonctionnement de [nginx fait en sorte de ne logger que les requêtes traitées](https://docs.nginx.com/nginx/admin-guide/monitoring/logging/#setting-up-the-access-log).

```erb
server {
  listen <%= ENV['PORT'] %>;
  set $logme 0;
  access_log logs/access.log CSP if=$logme;
  
  client_max_body_size 4k;
  
  location = / {
    limit_except POST { 
      deny all; 
    }
    if ($http_content_type != "application/csp-report") {
      return 403;
    } 
    set $logme 1;`
    proxy_pass http://127.0.0.1:81/;
  }
}
```


### Les sécurités mise en place pour éviter d'être spammé par des fausses requêtes 

1. Le serveur n'accepte pas de requêtes de plus de 4k. 
```conf
client_max_body_size 4k;
```

2. Le serveur n'accepte que les requêtes POST sur l'uri `/` et avec le `Content-type: application/csp-report`

3. Le serveur ne loggue que les requêtes qui répondent à nos critères grâce à: 
   - la variable `set $logme 0;`
   - la condition `access_log logs/access.log CSP if=$logme;`

## Comment tester en local ? 

1. Cloner le buildpack nginx de Scalingo : 

```shell
git@github.com:Scalingo/nginx-buildpack.git
```

2. Copier le fichier `servers.conf.erb` de ce repository dans le clone du buildpack.

3. Aller sur le répertoire du buildpack. 

4. Lancer un container avec l'image nginx et les fichiers de configuration nginx nécessaires : 

```shell
PORT=80 HAS_SERVER_CONF=true erb ./config/nginx.conf.erb > nginx.conf \
& PORT=80 erb servers.conf.erb > servers.conf \
&& docker run --rm -ti -v $PWD/nginx.conf:/etc/nginx/nginx.conf:ro -v $PWD/servers.conf:/etc/nginx/servers.conf:ro  --tmpfs /etc/nginx/logs -p 9000:80 nginx nginx
```

5. Dans un second terminal, tester différentes requêtes : 

```shell
http -v --form :9000/ x=1
http -v POST :9000/ Content-type:application/csp-report
```

6. Voir les logs : 

```shell
docker exec $(docker ps -lq) cat /etc/nginx/logs/access.log
```

