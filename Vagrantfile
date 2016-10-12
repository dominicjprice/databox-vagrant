# -*- mode: ruby -*-
# vi: set ft=ruby :

require "json"
require "./vagrant-config.rb"

CONSTANTS = {
    "databox_application_server" => {
        "name" => "databox-app-server",
        "mongodb" => {
            "host" => "databox-app-server-mongodb",
            "port" => "27017",
            "user" => "user",
            "pass" => "pass",
            "db" => "databox-app-server"
        }
    },
    "mongodb" => {
        "name" => "mongodb",
        "user" => "user",
        "pass" => "pass",
        "database" => "db"
    },
    "databox_container_manager" => {
    	"registryUrl" => "registry.docker:5000",
    	"storeUrl" => "http://databox-app-server.docker",
    	"serverPort" => 8080
    }
}

Vagrant.configure(2) do |c|
   
  c.vm.box = "ubuntu/wily64"
    
  c.vm.network "private_network", ip: CONFIG["IP"]
    
  c.vm.provider "virtualbox" do |v|
    v.memory = CONFIG["memory"]
  end
    
  c.vm.provision "shell", inline: <<-SHELL
    
    export DEBIAN_FRONTEND=noninteractive
    curl -sL https://deb.nodesource.com/setup_6.x | sudo -E bash -
    apt-get update
    apt-get install -y git build-essential nodejs
    
    INSTALLDIR="/opt/databox-app-server"
    IP="#{CONFIG["IP"]}"
    
    if [ ! -e "$INSTALLDIR" ]; then
      git clone -o origin -b master \
          https://github.com/dominicjprice/databox-app-server.git $INSTALLDIR
    else
      cd $INSTALLDIR && git pull origin master
    fi
    
    cat << EOM > $INSTALLDIR/config.json
#{JSON.generate(CONSTANTS["databox_application_server"]
    .merge(CONFIG["databox_application_server"]))}
EOM
    
    sed -i -e "s/datashop\\.amar\\.io\\/user/$IP\\/das\\/user/" \
        -e "s/datashop\\.amar\\.io/$IP/" $INSTALLDIR/email.ls
        
    mkdir /opt/certs
    mkdir -p /etc/docker/certs.d/registry.docker:5000
    openssl req -newkey rsa:4096 -nodes -sha256 -keyout \
        /etc/docker/certs.d/registry.docker:5000/ca.key -x509 -days 365 \
        -out /etc/docker/certs.d/registry.docker:5000/ca.cert -subj \
        "/C=GB/ST=Notts/L=Nottingham/O=University of Nottingham/OU=Horizon DER/CN=registry.docker"
    while [ ! -e /etc/docker/certs.d/registry.docker:5000/ca.cert ]; do
        echo "Waiting for certificate"
        sleep 1s
    done
    cp /etc/docker/certs.d/registry.docker:5000/ca.cert \
        /etc/docker/certs.d/registry.docker:5000/ca.crt
    cp /etc/docker/certs.d/registry.docker:5000/* /opt/certs

  SHELL
  
  c.vm.provision "docker" do |d|

    conf = CONSTANTS["databox_application_server"]
        
    d.pull_images "tutum/mongodb"
    
    d.pull_images "gliderlabs/resolvable:master"
    
    d.pull_images "registry:2"
    
    d.pull_images "dominicjprice/docker-apache-2.4-proxy:latest"
    
    d.run "gliderlabs/resolvable:master",
        auto_assign_name: false,
        args: "--name resolvable \
            -v /var/run/docker.sock:/tmp/docker.sock \
            -v /etc/resolv.conf:/tmp/resolv.conf"
      
    d.run "registry:2",
        auto_assign_name: false,
        args: "--name registry -p 5000:5000 \
            -v /opt/certs:/certs \
            -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/ca.cert \
            -e REGISTRY_HTTP_TLS_KEY=/certs/ca.key"
        
    d.run "tutum/mongodb",
        auto_assign_name: false,
        args: "--name #{conf["mongodb"]["host"]} \
            -e MONGODB_USER=\"#{conf["mongodb"]["user"]}\" \
            -e MONGODB_PASS=\"#{conf["mongodb"]["pass"]}\" \
            -e MONGODB_DATABASE=\"#{conf["mongodb"]["db"]}\""
                        
    d.build_image "/opt/databox-app-server", args: "-t #{conf["name"]}"
        
    d.run conf["name"],
        auto_assign_name: false,
        args: "--name #{conf["name"]} -e PORT=80 \
              --link #{conf["mongodb"]["host"]}:#{conf["mongodb"]["host"]}"
                        
  end  

  c.vm.provision "shell", inline: <<-SHELL
            
    if [ ! -e "/opt/frontend-proxy" ]; then
      mkdir /opt/frontend-proxy
    fi
    echo "FROM dominicjprice/docker-apache-2.4-proxy:latest" >> /opt/frontend-proxy/Dockerfile
    echo "ADD proxy.conf /conf/" >> /opt/frontend-proxy/Dockerfile
    echo "ProxyPass /das/ http://#{CONSTANTS["databox_application_server"]["name"]}/" \
        >> /opt/frontend-proxy/proxy.conf
    echo "ProxyPass / http://#{CONFIG["IP"]}:8080/" >> /opt/frontend-proxy/proxy.conf
    
  SHELL

  c.vm.provision "docker" do |d|
        
    d.build_image "/opt/frontend-proxy", args: "-t frontend-proxy"
        
    app_server_name = CONSTANTS["databox_application_server"]["name"]
        
    d.run "frontend-proxy", 
        auto_assign_name: false,
        args: "--name frontend-proxy \
            --link #{app_server_name}:#{app_server_name} \
            -p 80:80"

  end
  
  c.vm.provision "shell", inline: <<-SHELL
  
    docker pull registry.upintheclouds.org/databox-arbiter:latest
    docker tag registry.upintheclouds.org/databox-arbiter:latest \
        registry.docker:5000/databox-arbiter:latest
    docker push registry.docker:5000/databox-arbiter:latest
    docker pull registry.upintheclouds.org/databox-directory:latest
    docker tag registry.upintheclouds.org/databox-directory:latest \
        registry.docker:5000/databox-directory:latest
    docker push registry.docker:5000/databox-directory:latest
  
    INSTALLDIR="/opt/databox-container-manager"
    
    if [ ! -e "$INSTALLDIR" ]; then
      git clone -o origin -b master \
          https://github.com/ktg/databox-container-manager.git $INSTALLDIR
    else
      cd $INSTALLDIR && git pull origin master
    fi
    
    cat << EOM > $INSTALLDIR/src/config.json
#{JSON.generate(CONSTANTS["databox_container_manager"])}
EOM

	sed -i "s/var server = require('\\.\\/server.js');/var server = require('\\.\\/server.js');\\nprocess.env.NODE_TLS_REJECT_UNAUTHORIZED = \\"0\\";/" $INSTALLDIR/src/main.js

    cd $INSTALLDIR
    npm install -g riot pug
    npm install --production
    npm run build 
  
  SHELL
  
end
