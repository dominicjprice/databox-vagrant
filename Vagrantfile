# -*- mode: ruby -*-
# vi: set ft=ruby :

require "json"
require "./vagrant-config.rb"

Constants = {
    "databox_application_server" => {
        "name" => "databox-app-server",
        "mongodb" => {
            "name" => "databox-app-server-mongodb",
            "host" => "databox-app-server-mongodb",
            "port" => "27017",
            "user" => "user",
            "pass" => "pass",
            "db" => "databox-app-server"
        }
    },
    "databox_container_manager" => {
    	"registryUrl" => "registry.docker:5000",
    	"storeUrl" => "http://databox-app-server.docker",
    	"serverPort" => 8080
    }
}

class ::Hash
  def deep_merge(second)
    merger = proc { |key, v1, v2| Hash === v1 \
        && Hash === v2 ? v1.merge(v2, &merger) : v2 }
    self.merge(second, &merger)
  end
end

Vagrant.configure(2) do |vagrant|

  Config = Constants.deep_merge(Configuration)
   
  vagrant.vm.box = "ubuntu/wily64"
    
  vagrant.vm.network "private_network", ip: Config["IP"]
    
  vagrant.vm.provider "virtualbox" do |v|
    v.memory = Config["memory"]
  end
    
  vagrant.vm.provision "shell", inline: <<-SHELL
    
    export DEBIAN_FRONTEND=noninteractive
    curl -sL https://deb.nodesource.com/setup_6.x | sudo -E bash -
    apt-get update
    apt-get install -y git build-essential nodejs
    
    INSTALLDIR="/opt/databox-app-server"
    IP="#{Config["IP"]}"
    
    
    if [ ! -e "$INSTALLDIR" ]; then
      git clone -o origin -b master \
          https://github.com/me-box/databox-app-server.git $INSTALLDIR
    else
      cd $INSTALLDIR && git pull origin master
    fi
    
    cat << EOM > $INSTALLDIR/config.json
#{JSON.generate(Config["databox_application_server"])}
EOM
    
    sed -i -e "s/datashop\\.amar\\.io\\/user/$IP\\/app-server\\/user/" \
        -e "s/datashop\\.amar\\.io/$IP/" $INSTALLDIR/email.ls
        
    CERTDIR=/opt/certs
    mkdir -p $CERTDIR
    mkdir -p /etc/docker/certs.d
    ln -s $CERTDIR /etc/docker/certs.d/registry.docker:5000 
    openssl req -newkey rsa:4096 -nodes -sha256 -keyout \
        $CERTDIR/client.key -x509 -days 365 -out $CERTDIR/client.cert -subj \
        "/C=GB/ST=Notts/L=Nottingham/O=University of Nottingham/OU=Horizon DER\
            /CN=registry.docker"
    cp -f $CERTDIR/client.cert $CERTDIR/ca.crt
    exit 0
    
  SHELL
  
  vagrant.vm.provision "docker", 
      images: [ "tutum/mongodb", "gliderlabs/resolvable:master", 
          "registry:2", "nginx:latest" ]
          
  vagrant.vm.provision "shell", inline: <<-SHELL
  
    for container in "registry" "resolvable" "frontend-proxy" \
        "#{Config["databox_application_server"]["name"]}" \
        "#{Config["databox_application_server"]["mongodb"]["name"]}"; do 
      docker stop $container 
  	  docker wait $container
      docker rm $container
    done
    
    exit 0
  
  SHELL
  
  vagrant.vm.provision "docker" do |d|

    d.run "gliderlabs/resolvable:master",
        auto_assign_name: false,
        args: "--name resolvable \
            -v /var/run/docker.sock:/tmp/docker.sock \
            -v /etc/resolv.conf:/tmp/resolv.conf"

    d.run "registry:2",
        auto_assign_name: false,
        args: "--name registry -p 5000:5000 \
            -v /opt/certs:/certs \
            -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/client.cert \
            -e REGISTRY_HTTP_TLS_KEY=/certs/client.key"
            
    das_conf = Config["databox_application_server"]
    
    d.run "tutum/mongodb",
        auto_assign_name: false,
        args: "--name #{das_conf["mongodb"]["name"]} \
            -e MONGODB_USER=\"#{das_conf["mongodb"]["user"]}\" \
            -e MONGODB_PASS=\"#{das_conf["mongodb"]["pass"]}\" \
            -e MONGODB_DATABASE=\"#{das_conf["mongodb"]["db"]}\""
                        
    d.build_image "/opt/databox-app-server", args: "-t #{das_conf["name"]}"
        
    d.run das_conf["name"],
        auto_assign_name: false,
        args: "--name #{das_conf["name"]} -e PORT=80 --link \
            #{das_conf["mongodb"]["name"]}:#{das_conf["mongodb"]["name"]}"

  end
  
  vagrant.vm.provision "shell", inline: <<-SHELL
            
    if [ ! -e "/opt/frontend-proxy" ]; then
      mkdir /opt/frontend-proxy
    fi
    
    cat << EOM > /opt/frontend-proxy/default.conf
server {
    
  listen       80;
  server_name  localhost;

  location /app-server/ {
      proxy_pass http://#{Config["databox_application_server"]["name"]}.docker/;
      proxy_http_version 1.1;
      proxy_set_header Host \\$host;
  }

  location /socket.io/ {
    proxy_pass http://#{Config["IP"]}:8080/socket.io/;
    proxy_http_version 1.1;
    proxy_set_header Upgrade \\$http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host \\$host;
  }

  location / {
    proxy_pass http://#{Config["IP"]}:8080/;
    proxy_http_version 1.1;
    proxy_set_header Host \\$host;
  }

  error_page 500 502 503 504 /50x.html;
  location = /50x.html {
    root /usr/share/nginx/html;
  }

}
EOM
    
    cat << EOM > /opt/frontend-proxy/Dockerfile
FROM nginx:latest
ADD default.conf /etc/nginx/conf.d/default.conf
EOM
    
  SHELL
  
  vagrant.vm.provision "docker" do |d|
        
    d.build_image "/opt/frontend-proxy", args: "-t frontend-proxy"
        
    app_server_name = Config["databox_application_server"]["name"]
        
    d.run "frontend-proxy", 
        auto_assign_name: false,
        args: "--name frontend-proxy \
            --link #{app_server_name}:#{app_server_name} \
            -p 80:80"

  end
  
  vagrant.vm.provision "shell", inline: <<-SHELL
  
  	from_registry="registry.upintheclouds.org"
  	to_registry="registry.docker:5000"
    for image in "databox-arbiter" "databox-directory" \
        "databox-notifications"; do
      docker pull ${from_registry}/${image}:latest
      docker tag -f ${from_registry}/${image}:latest \
          ${to_registry}/${image}:latest
      docker push ${to_registry}/${image}:latest
    done
  
    INSTALLDIR="/opt/databox-container-manager"
    
    if [ ! -e "$INSTALLDIR" ]; then
      git clone -o origin -b master \
          https://github.com/ktg/databox-container-manager.git $INSTALLDIR
    else
      cd $INSTALLDIR && git pull origin master
    fi
    
    cat << EOM > $INSTALLDIR/src/config.json
#{JSON.generate(Config["databox_container_manager"])}
EOM

	sed -i "s/var server = require('\\.\\/server.js');/var server = require('\\.\\/server.js');\\nprocess.env.NODE_TLS_REJECT_UNAUTHORIZED = \\"0\\";/" $INSTALLDIR/src/main.js

	killall npm
    cd $INSTALLDIR
    npm install -g riot pug
    npm install --production
    nohup npm start 0<&- &>/var/log/databox-container-manager &
  
  SHELL
  
end
