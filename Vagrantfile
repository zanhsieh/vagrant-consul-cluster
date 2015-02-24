 -*- mode: ruby -*-
# vi: set ft=ruby :

$IPs = {
  master1: "10.0.40.41",
  master2: "10.0.40.42",
  master3: "10.0.40.43",
  slave1:  "10.0.40.51",
  slave2:  "10.0.40.52",
  slave3:  "10.0.40.53"
}

$ports = [8301, 8400, 8500]

$encrypt_str = "X4SYOinf2pTAcAHRhpj7dA"

$host_check = <<-HOSTCHECK
#{$IPs.map {|k,v|<<-INNER
if [ ! `grep -q #{v} /etc/hosts` ]; then
  echo '#{v} #{k}' | sudo tee -a /etc/hosts
fi
INNER
}.join()}
HOSTCHECK

$firewall_setup = <<-FIREWALLSETUP
#{case $ports.size
when 0
else
<<-INNER
sudo ufw allow #{$ports.join(',')}/tcp
INNER
end
}
FIREWALLSETUP

$bootstrap_json = <<-BOOTSTRAPJSON
sudo tee /etc/consul.d/bootstrap/config.json <<- EOF
{
    "bootstrap": true,
    "server": true,
    "bind_addr": "#{$IPs[:master1]}",
    "datacenter": "hk",
    "data_dir": "/var/consul",
    "encrypt": "#{$encrypt_str}",
    "log_level": "INFO",
    "enable_syslog": true
}
EOF
BOOTSTRAPJSON

$master1_json = <<-MASTER1JSON
sudo tee /etc/consul.d/server/config.json <<- EOF
{
    "bootstrap": false,
    "server": true,
    "bind_addr": "#{$IPs[:master1]}",
    "datacenter": "hk",
    "data_dir": "/var/consul",
    "encrypt": "#{$encrypt_str}",
    "log_level": "INFO",
    "enable_syslog": true,
    "start_join": #{Hash[$IPs.first 3].reject!{|k| k == :master1 }.values.to_s}
}
EOF
MASTER1JSON

$master2_json = <<-MASTER2JSON
sudo tee /etc/consul.d/server/config.json <<- EOF
{
    "bootstrap": false,
    "server": true,
    "bind_addr": "#{$IPs[:master2]}",
    "datacenter": "hk",
    "data_dir": "/var/consul",
    "encrypt": "#{$encrypt_str}",
    "log_level": "INFO",
    "enable_syslog": true,
    "start_join": #{Hash[$IPs.first 3].reject!{|k| k == :master2 }.values.to_s}
}
EOF
MASTER2JSON

$master3_json = <<-MASTER3JSON
sudo tee /etc/consul.d/server/config.json <<- EOF
{
    "bootstrap": false,
    "server": true,
    "bind_addr": "#{$IPs[:master3]}",
    "datacenter": "hk",
    "data_dir": "/var/consul",
    "encrypt": "#{$encrypt_str}",
    "log_level": "INFO",
    "enable_syslog": true,
    "start_join": #{Hash[$IPs.first 3].reject!{|k| k == :master3 }.values.to_s}
}
EOF
MASTER3JSON

$slave_json = <<-SLAVEJSON
sudo tee /etc/consul.d/client/config.json <<- EOF
{
  "bind_addr": "#{$IPs[:slave1]}",
  "server": false,
  "datacenter": "hk",
  "data_dir": "/var/consul",
  "ui_dir": "/home/consul/dist",
  "encrypt": "#{$encrypt_str}",
  "log_level": "INFO",
  "enable_syslog": true,
  "start_join": #{Hash[$IPs.first 3].values.to_s}
}
EOF
SLAVEJSON

$server_init_setup = <<-SERVERINITSETUP
sudo tee /etc/init/consul.conf <<- EOF
description "Consul server process"

start on (local-filesystems and net-device-up IFACE=eth1)
stop on runlevel [!12345]

respawn

setuid consul
setgid consul

exec consul agent -config-dir /etc/consul.d/server
EOF
SERVERINITSETUP

$client_init_setup = <<-CLIENTINITSETUP
sudo tee /etc/init/consul.conf <<- EOF
description "Consul client process"

start on (local-filesystems and net-device-up IFACE=eth1)
stop on runlevel [!12345]

respawn

setuid consul
setgid consul

exec consul agent -config-dir /etc/consul.d/client
EOF
CLIENTINITSETUP

$consul_setup = <<-CONSULSETUP
sudo apt-get update
sudo apt-get install unzip
cd /usr/local/bin
cp /vagrant/0.3.0_linux_amd64.zip .
unzip *.zip && rm *.zip
sudo adduser --disabled-password --gecos "" consul
sudo mkdir -p /etc/consul.d/{bootstrap,server,client}
sudo mkdir /var/consul
sudo chown consul:consul /var/consul
CONSULSETUP

$webui_setup = <<-WEBUISETUP
cd /home/consul
cp /vagrant/0.3.0_web_ui.zip .
unzip *.zip && rm *.zip
sudo chown consul:consul -R /home/consul
WEBUISETUP

Vagrant.configure(2) do |config|
  config.vm.box = "ubuntu/trusty64"
  config.vm.box_check_update = false
  if Vagrant.has_plugin?("vagrant-cachier")
    config.cache.scope = "box"
    config.cache.synced_folder_opts = {
      type: "nfs",
      mount_options: ['rw', 'vers=3', 'tcp', 'nolock']
    }
  end
  if Vagrant.has_plugin?("vagrant-vbguest")
    config.vbguest.auto_update = false
  end
  config.vm.synced_folder '.', '/vagrant', nfs: true
  (1..3).each do |i|
    config.vm.define "master#{i}" do |m|
      m.vm.hostname = "master#{i}"
      m.vm.network "private_network", ip: "10.0.40.4#{i}"
      m.vm.provision "shell", inline:<<-SHELL
      #{$firewall_setup}
      #{$host_check}
      #{$consul_setup}
      #{case i
        when 1
          <<-INNER
          #{$bootstrap_json}
          #{$master1_json}
          INNER
        when 2
          <<-INNER
          #{$master2_json}
          INNER
        when 3
          <<-INNER
          #{$master3_json}
          INNER
        end
      }
      #{$server_init_setup}
      SHELL
    end
  end
  (1..1).each do |j|
    config.vm.define "slave#{j}" do |s|
      s.vm.hostname = "slave#{j}"
      s.vm.network "private_network", ip: "10.0.40.5#{j}"
	  s.vm.network "forwarded_port", guest: 8050, host: 8050
      s.vm.provision "shell", inline:<<-SHELL
      #{$firewall_setup}
      #{$host_check}
      #{$consul_setup}
      #{$webui_setup}
      #{$slave_json}
      #{$client_init_setup}
      SHELL
    end
  end
end
