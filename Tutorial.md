Blue-Green Deployment is a strategy to release new version of the app without downtime. The basic idea behind this technique involves using two identical production environments, named Blue and Green. At any time, only one of these environment is live and serving the production traffic. The other one is used to test newer version or for roll-back.

Let us assume that the current live production environment is Blue. When the new version is ready, we can deploy it to the non-production environment - Green. None of our users can see this new version as the live environment is still Blue. We can test the new version from the Green environment now. If this version is ready for release, we switch the production environment to Green and the users can now see the new release. Now the live version in at Green and staging can be done in Blue. If there is some error with the new release in Green, it is easy to roll-back to previous version by just switching the production environment back to Blue. Only thing to note here is that the switching is seamless.

In this article, we will build a blue-green deployment system with Dockers. We will create and control a cluster of nodes with Docker Swarm. We will build a Blue-Green deployment docker image that creates two environment, each running different versions of same test app. We will also see how to switch the live environment with out Docker image.

The docker image for Blue-Green deployment is hanzel/blue-green and its code can be found here. Also, the docker image for the test application is hanzel/nginx-html and its code can be found here.

Prerequisites
We will be using Docker Machine to create and manage remote hosts as a swarm. With Docker Machine, you can create hosts on your local machine or your cloud provider. Check this link to see the drivers supported by Docker Machine.

You need to have the following installed in you local computer:

Docker: version >= 1.10, to support Docker Compose File version 2 and Multi-Host networking.
Docker Machine: version >= 0.6
Docker Compose: version >= 1.6, to support Docker Compose file version 2
You can create the virtual hosts in you local system if you have VirtualBox installed. For this demonstration, I will be using DigitalOcean.

Creating the Swarm
The first thing we need to do is to create the Docker Swarm using Docker Machine and set it up. I have explained how to do this in my previous article, Load Balancing with Docker Swarm. Follow the steps from Initial Setup to The Swarm of that article to create and setup the Swarm.

Once the swarm is setup, you can see the hosts with docker-machine ls command. The output of this command must look something like this.

NAME     ACTIVE      DRIVER         STATE     URL                          SWARM             DOCKER
consul   -           digitalocean   Running   tcp://104.236.235.185:2376                     v1.10.1
master   * (swarm)   digitalocean   Running   tcp://159.203.119.37:2376    master (master)   v1.10.1
slave    -           digitalocean   Running   tcp://45.55.185.18:2376      master            v1.10.1 
Test App
To demonstrate blue-green deployment, we will deploy different versions of our test app. The docker image for this app is hanzel/nginx-html and the code for it can be found here. The docker image contains the nginx webserver that serves a static HTML page at port 80.

The HTML page contains the current version or tag of the docker image. So the image hanzel/nginx-html:1 serves the HTML page with Version 1, hanzel/nginx-html:2 serves the HTML page with Version 2 and hanzel/nginx-html:3 serves the HTML page with Version 3.

Consul Template
Now, we are going to build the image that does the blue-green deployment. It uses nginx webserver for load balancing and consul-template to manage nginx configuration dynamically. The docker image for this app is hanzel/blue-green and the code for it can be found here.

If the current live environment is blue, we can access that at port 80 of the container. The staging environment, green in this case, can be accessed from the port 8080 of the container. We will be able to switch the live environment anytime with just one command. If we switch the live environment to green, then we can access the live green environment at port 80 and staging blue environment at port 8080. This is how the image facilitates blue-green deployment.

We have used the registrator image to register our running docker images to consul. Now, consul-template will read these and create custom configuration of nginx. So, we need to create a template of nginx configuration. This file will be called default.ctmpl and it looks like this.

{{$blue  := env "BLUE_NAME"}}
{{$green := env "GREEN_NAME"}}
{{$live  := file "/var/live"}}
worker_processes  1;

events {
    worker_connections  1024;
}

http {
  upstream blue {
    least_conn;
    {{range service $blue}}
    server {{.Address}}:{{.Port}} max_fails=3 fail_timeout=60 weight=1;{{else}}
    server 127.0.0.1:55000;{{end}}
  }

  upstream green {
    least_conn;
    {{range service $green}}
    server {{.Address}}:{{.Port}} max_fails=3 fail_timeout=60 weight=1;{{else}}
    server 127.0.0.1:55000;{{end}}
  }

  server {
    listen 80 default;

    location / {
      {{if eq $live "blue"}}
      proxy_pass http://blue;
      {{else}}
      proxy_pass http://green;
      {{end}}
    }
  }

  server {
    listen 8080;

    location / {
      {{if eq $live "blue"}}
      proxy_pass http://green;
      {{else}}
      proxy_pass http://blue;
      {{end}}
    }
  }
}
First of all, we set three variables:

blue: The docker service name of the blue environment, taken from environment variable BLUE_NAME.
green: The docker service name of the green environment, taken from environment variable GREEN_NAME.
live: Current live environment, blue or green, taken from the file /var/live.
We will store the current live environment, blue or green, in the file /var/live. We cannot use environment variable for this because we cannot globally change the value of environment variable from inside a running docker image. So we write the current live environment, while switching, to the file and read its content from inside consul-template.

Inside the http block, we create an upstream block for blue and green. Inside each of this upstream block, we specify the load balancing configuration for each service. The least_conn line causes nginx is to route traffic to the least connected instance. We need to generate server configuration lines for each instance of the service currently running. This is done by the code blocks, {{range service $blue}}...{{end}} and {{range service $blue}}...{{end}}. The code between these directives are repeated for each instance of the service running with {{.Address}} replaced by the address and {{.Port}} replaced by its port of that instance. If there is no instance of any service, we have the default server 127.0.0.1:55000; line that causes an error.

Next we have the server block that is listening to the port 80. If the value of the live variable is blue, this is proxied to the blue app. If the value of live is green, this is proxied to green app. So, in essence, the port 80 will point to the live environment.

Similarly, we have a server block that listens to port 8080. This is proxied to the staging environment. So, if the value of live is blue, this points to the green app and vice-versa. In any case, the port 80 will give the live environment and port 8080 will give the staging environment.

Image Scripts
We need a bash script, that acts as the entry point to this docker image. The file start.sh looks like this.

#!/bin/sh
nginx -g 'daemon off;' &
echo -n $LIVE > /var/live
consul-template -consul=$CONSUL_URL -template="/templates/default.ctmpl:/etc/nginx/nginx.conf:nginx -s reload"  
The first line of the scipt starts up nginx. Now, we write the value of environment variable LIVE to the file /var/live. This environment variable contains the value blue or green, which is the initial live environment.

We then start up consul-template. This command need two parameter. The first one is -consul and it requires the url for consul. We pass an environment variable for this. The next one is called -template and it consists of three parts seperated by a colon. The first one is the path of the template file. The second is the path where the generated configuration file must be placed. The third is the command that must by run when new configuration is generated. Here, we need to reload nginx.

The consul-template listens for services and create new configuration file whenever a service starts or stops. The information about this is collected by the registrator services running in each node is our swarm and is stored in consul.

Now, we need another script to switch the live environment. The script will accept a parameter, either blue of green and change the current live environment to that value. The file switch looks like this.

#!/bin/sh

if [ $# -eq 0 ]
  then
    echo "No arguments supplied"
    exit 1
fi

if [ $1 = "blue" ]
  then
    echo -n "blue" > /var/live
  else
    echo -n "green" > /var/live
fi

consul-template -consul=$CONSUL_URL -template="/templates/default.ctmpl:/etc/nginx/nginx.conf:nginx -s reload" -retry 30s -once 
If there is no arguments passed to this scripts, it exits showing the error message, No arguments supplied. If the parameter is blue, it is written to the file /var/live. Else, the value green is written to that file. This is now the current live environment.

Finally, we run the consul-template command with the once parameter. This causes the consul-template to create new nginx configuration based on the new value in /var/live and reload nginx. This will switch the current live environment. As we have used the once parameter, the new configuration in made only once and consul-template will not listen for new services. For that, we have a consul-template running from our start.sh file.

Blue-Green Image
Save these three files, default.ctmpl, start.sh and switch, in folder named files. In its parent, we can have the Dockerfile and docker-compose.yml. The Dockerfile contains information on how to build this docker image and will look like this.

FROM nginx:alpine

RUN apk add --no-cache --virtual unzip
ADD https://releases.hashicorp.com/consul-template/0.14.0/consul-template_0.14.0_linux_amd64.zip /usr/bin/
RUN unzip /usr/bin/consul-template_0.14.0_linux_amd64.zip -d /usr/local/bin

COPY files/s* /bin/
RUN chmod +x /bin/switch /bin/start.sh
COPY files/default.ctmpl /templates/

ENV LIVE blue
ENV BLUE_NAME blue
ENV GREEN_NAME green

EXPOSE 80 8080
ENTRYPOINT ["/bin/start.sh"]
This dockerfile uses nginx:alpine as the base and installs unzip and consul-template into it. It then copies the start.sh, switch and default.ctmpl to required locations and make the scripts executable.

We also set the default values for following environment variables:

LIVE: The initial live environment. Set to blue.
BLUE_NAME: The docker service name of blue environment. Set to blue.
GREEN_NAME: The docker service name of green environment. Set to green.
We expose the port 80 and 8080. The start.sh file will be the entry point to this image.

Testing Blue-Green deployment
To test blue-green deployment, we will use the following docker-compose.yml file.

version: '2'

services:
  bg:
    image: hanzel/blue-green
    container_name: bg
    ports:
      - "80:80"
      - "8080:8080"
    environment:
      - constraint:node==master
      - CONSUL_URL=${KV_IP}:8500
      - BLUE_NAME=blue
      - GREEN_NAME=green
      - LIVE=blue
    depends_on:
      - green
      - blue
    networks:
      - blue-green

  blue:
    image: hanzel/nginx-html:1
    ports:
      - "80"
    environment:
      - SERVICE_80_NAME=blue
    networks:
      - blue-green

  green:
    image: hanzel/nginx-html:2
    ports:
      - "80"
    environment:
      - SERVICE_80_NAME=green
    networks:
      - blue-green

networks:
  blue-green:
    driver: overlay
We are using the version 2 of docker-compose file, with three services in an overlay network named blue-green. We have two versions of hanzel/nginx-html image running as blue and green services. We also have hanzel/blue-green image running for blue-green deployment.

The first service is the blue service, named blue. The image used is version 1 of hanzel/nginx-html. We have mapped the port 80 of the container to some port in the host. We have set the environment variable SERVICE_80_NAME to blue. This causes the registrator to register this service into consul named as blue. This is the initial live environment.

Similarly, we have the green service, named green. The image used here is version 2 of the hanzel/nginx-html. The environment variable SERVICE_80_NAME is set to green so that registrator will register it named as green. This is the initial statging environment.

Finally, we have the bg service with hanzel/blue-green image. You can also build the image we just made in the previous section for this service by replacing the line image: hanzel/blue-green with build: . and placing this file along with the Dockerfile we made in the previous section.

We map the ports 80 and 8080 of the container to that of the host. We also need to set the following environment variables.

constraint:node: The name of the node where this service should run. We want this service to always run on the master node.
CONSUL_URL: The url endpoint of consul. We have set it to ${KV_IP}:8500, where KV_IP is the environment variable we have set while making the swarm.
BLUE_NAME: The docker service name of the blue image. Set to blue.
GREEN_NAME: The docker service name of the green image. Set to green.
LIVE: The initial live environment, blue or green. Set to blue.
We can start the services with the following command.

docker-compose up -d
This will start up a single instance of each of these three services. We can scale the blue and green services to 3 instances each with the following command.

docker-compose scale blue=3 green=3
These will create two new instances for blue and green service. You can see the running services of docker-compose with the docker-compose ps command. The output of the command will look something like this.

Name          Command                State                               Ports
----------------------------------------------------------------------------------------------------------
bg            /bin/start.sh          Up      443/tcp, 45.55.185.18:80->80/tcp, 45.55.185.18:8080->8080/tcp
tmp_blue_1    nginx -g daemon off;   Up      443/tcp, 45.55.185.18:32777->80/tcp
tmp_blue_2    nginx -g daemon off;   Up      443/tcp, 159.203.119.37:32769->80/tcp
tmp_blue_3    nginx -g daemon off;   Up      443/tcp, 45.55.185.18:32778->80/tcp
tmp_green_1   nginx -g daemon off;   Up      443/tcp, 159.203.119.37:32768->80/tcp
tmp_green_2   nginx -g daemon off;   Up      443/tcp, 45.55.185.18:32779->80/tcp
tmp_green_3   nginx -g daemon off;   Up      443/tcp, 159.203.119.37:32770->80/tcp
We can see the live production environment from the url given by the command, docker-compose port bg 80. You will get some IP address like 45.55.185.18:80, which is the IP for the master node. Go to this url and we can see the live environment, currently blue, showing Version 1. You can see the staging environment, currently green, by going to port 8080 of the same IP. That will be 45.55.185.18:8080 in this case. This will show you Version 2.

Now, the users can see the version 1 of your app and only you can see version 2. You can test the new version and if you are satisfied, you can switch the live environment to green. To do this, use the following command.

docker exec bg switch green
Now, the live version is green and at port 80, you can see version 2 and at port 8080, you can see version 1. You can see the new nginx configuration with the command, docker exec bg cat /etc/nginx/nginx.conf. The output of this command will look like this.

worker_processes 1;

events {
  worker_connections  1024;
}

http {
  upstream blue {
    least_conn;

    server 10.132.12.95:32769 max_fails=3 fail_timeout=60 weight=1;
    server 10.132.35.39:32777 max_fails=3 fail_timeout=60 weight=1;
    server 10.132.35.39:32778 max_fails=3 fail_timeout=60 weight=1;
  }

  upstream green {
    least_conn;

    server 10.132.12.95:32768 max_fails=3 fail_timeout=60 weight=1;
    server 10.132.12.95:32770 max_fails=3 fail_timeout=60 weight=1;
    server 10.132.35.39:32779 max_fails=3 fail_timeout=60 weight=1;
  }

  server {
    listen 80 default;

    location / {
      proxy_pass http://green;
    }
  }

  server {
    listen 8080;

    location / {
      proxy_pass http://blue;
    }
  }
}
You can always check the current live environment using the command, docker exec bg cat /var/live. Now, blue is the staging environment and we can check version 3 there. So in the blue service of the docker-compose.yml file, change the line image: hanzel/nginx-html:1 to image: hanzel/nginx-html:3. To update the blue service, run the following command.

docker-compose up -d blue
All three instances for blue services will be upgraded from version 1 to version 3 now. The staging environment at port 8080 of the bg service will now show Version 3. You can check this version and if it is okay for production, switch the live environment to blue with the following command.

docker exec bg switch blue
Now, live environment at port 80 will show Version 3 and the staging environment at port 8080 will show Version 2. You can repeat this process for newer versions.

Conclusion
In this article, we have seen how to build a blue-green deployment system with Dockers to release new version of the app without downtime. We have made a docker image to implement this blue-green deployment and tested it on an app deployed with Docker Swarm.

Once you are done, the services can be stopped and the hosts removed with the following commands.

docker-compose down
docker-machine stop consul master slave
docker-machine rm consul master slave
