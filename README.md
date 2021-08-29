# wb28-introduccion-a-CICD

```
apt-get update
apt-get install default-jdk git nginx ufw python-certbot-nginx

wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
apt-get update
apt-get install jenkins
``` 
Creamos un proxypass en nginx para jenkins al 8080:
```
upstream jenkins {
  keepalive 32; # keepalive connections
  server 127.0.0.1:8080; # jenkins ip and port
}

# Required for Jenkins websocket agents
map $http_upgrade $connection_upgrade {
  default upgrade;
  '' close;
}

server {
    server_name jenkins.devops-alumno08.com;

    ignore_invalid_headers off;

  location ~ "^/static/[0-9a-fA-F]{8}\/(.*)$" {
    # rewrite all static files into requests to the root
    # E.g /static/12345678/css/something.css will become /css/something.css
    rewrite "^/static/[0-9a-fA-F]{8}\/(.*)" /$1 last;
  }

   location /userContent {
    # have nginx handle all the static requests to userContent folder
    # note : This is the $JENKINS_HOME dir
    root /var/lib/jenkins/;
    if (!-f $request_filename){
      # this file does not exist, might be a directory or a /**view** url
      rewrite (.*) /$1 last;
      break;
    }
    sendfile on;
    }

    location / {
      sendfile off;
      proxy_pass         http://jenkins;
      proxy_redirect     default;
      proxy_http_version 1.1;

      # Required for Jenkins websocket agents
      proxy_set_header   Connection        $connection_upgrade;
      proxy_set_header   Upgrade           $http_upgrade;

      proxy_set_header   Host              $host;
      proxy_set_header   X-Real-IP         $remote_addr;
      proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
      proxy_set_header   X-Forwarded-Proto $scheme;
      proxy_max_temp_file_size 0;

      #this is the maximum upload size
      client_max_body_size       10m;
      client_body_buffer_size    128k;

      proxy_connect_timeout      90;
      proxy_send_timeout         90;
      proxy_read_timeout         90;
      proxy_buffering            off;
      proxy_request_buffering    off; # Required for HTTP CLI commands
      proxy_set_header Connection ""; # Clear for keepalive
    }

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/jenkins.devops-alumno08.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/jenkins.devops-alumno08.com/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}
server {
    if ($host = jenkins.devops-alumno08.com) {
        return 301 https://$host$request_uri;
    } # managed by Certbot

    listen 80;
    server_name jenkins.devops-alumno08.com;
    return 404; # managed by Certbot

}
```

Instalamos jenkins y creamos el usuario enrique con el pass Wb28_2021

> Cuando lo tengas todo , añade a los pasos de build de django la ejecución del deploy. Jenkins deberá hacer ssh al servidor nuevo, cambiar a la carpeta donde está el repo, hacer git pull, correr migraciones si las hubiese, cambiar los permisos a www-data y finalmente hacer un reload del uwsgi si todo ha ido bien.
```
#!/bin/bash -x

####################################
###### ENVIRONMENT VARIABLES #######
####################################
export PROJECT_NAME="/var/django/djangotest"

ssh-agent bash
ssh-add 
ssh-keyscan 65.21.186.195 >> ~/.ssh/known_hosts &&
  
ssh -A root@65.21.186.195 "cd ${PROJECT_NAME} && git checkout $BRANCH && git pull"  &&
  
ssh -A root@65.21.186.195 "cd ${PROJECT_NAME} && python3 manage.py makemigrations && python3 manage.py migrate"  &&
ssh -A root@65.21.186.195 "chown -R www-data: ${PROJECT_NAME}" && 
 
ssh -A root@65.21.186.195 "service uwsgi reload"
```

## Gitlab plugin
Usamos github en lugar de gitlab. Nos vamos a la configuración del proyecto de github en el apartado de webhooks y creamos un nuevo webhook para la rama master. El web hook es el siguiente:
http://jenkins.devops-alumno08.com:8080/github-webhook/

En jenkins hay que habilitar la opción de GitHub hook trigger for GITScm polling

Al hacer commit a master se lanza automáticamente (tras un periodo de gracia de 3 segundos) el build.

## Builds avanzadas
Usamos github en lugar de gitlab. Esta es la config final:
```
#!/bin/bash -x

function parse_environment_and_branch() {

if [ "${GIT_BRANCH}" == "origin/master" ]
then
export SERVER_IP=65.21.186.195
export BRANCH=master
export ENVIRONMENT=production
fi

if [ "${GIT_BRANCH}" == "origin/staging" ]
then
export SERVER_IP=65.21.186.195
export BRANCH=staging
export ENVIRONMENT=staging
fi

if [ "${GIT_BRANCH}" == "origin/develop" ] 
then
export SERVER_IP=65.21.186.195
export BRANCH=develop
export ENVIRONMENT=develop
fi

}

####################################
###### ENVIRONMENT VARIABLES #######
####################################
export SECRET_KEY="blablablablab"
export PROJECT_NAME="/var/django/djangotest"

####################################
############### BUILD ##############
####################################
parse_environment_and_branch

ssh-agent bash
ssh-add 

{

if [ -z $ENVIRONMENT ] ; then echo "No hay entorno definido" && exit 1 ; fi

  #python3 manage.py test &&

  ssh-keyscan ${SERVER_IP} >> ~/.ssh/known_hosts &&
  
  ssh -A root@${SERVER_IP} "cd ${PROJECT_NAME} && git checkout $BRANCH && git pull"  &&
  
  #ssh -A root@${SERVER_IP} "cd ${PROJECT_NAME} && pip3 install -r requirements.txt"  &&

  ssh -A root@${SERVER_IP} "cd ${PROJECT_NAME} && python3 manage.py makemigrations && python3 manage.py migrate"  &&
 
  ssh -A root@${SERVER_IP} "service uwsgi reload"

} || {

echo "Algo ha fallado en la ejecucion"
exit 1

}
```
