# -*- mode: ruby -*-
# vi: set ft=ruby :

$swarm_workers = 1
# https://github.com/mitchellh/vagrant/issues/8322
$interface = "enp0s8"

Vagrant.configure("2") do |config|
  config.ssh.pty = true
  config.vm.box = "minimal/xenial64"
  config.vm.provision :docker

  config.vm.network "private_network",
    type: "dhcp"

  config.vm.define "swarm-manager" do |config|
    config.vm.hostname = "swarm-manager"
    config.vm.provision "shell", inline: <<-SHELL
        ifconfig #{$interface} | awk '/(cast)/ {print $2}' | cut -d: -f2 > /vagrant/swarm_manager_ip
        docker swarm init --advertise-addr $(cat /vagrant/swarm_manager_ip)
        docker swarm join-token -q worker > /vagrant/swarm_worker_token
        docker swarm join-token -q manager > /vagrant/swarm_manager_token
    SHELL
  end

  $swarm_workers.times do |n|
    config.vm.define "swarm-worker-#{n}" do |config|
      config.vm.hostname = "swarm-worker-#{n}"
      config.vm.provision "shell", inline: <<-SHELL
          docker swarm join --token=$(cat /vagrant/swarm_worker_token) $(cat /vagrant/swarm_manager_ip):2377
      SHELL
    end
  end

  config.vm.define "main", primary: true do |config|
    config.vm.hostname = "main"
    config.vm.provision "shell", inline: <<-SHELL
      set -x
      docker swarm join --token=$(cat /vagrant/swarm_manager_token) $(cat /vagrant/swarm_manager_ip):2377
      docker node update --availability drain main
      docker node ls
      curl --max-time 60 localhost:8080 | head -3
      docker stack deploy --compose-file /vagrant/docker-compose.yml app-swarm
      sleep 30 # maybe --wait until all services run option for stack deploy?
      docker service ls
      docker service ps app-swarm_nginx
      docker stack ps app-swarm
      curl --max-time 60 localhost:8080 | head -3
      set +x
    SHELL
  end
end
