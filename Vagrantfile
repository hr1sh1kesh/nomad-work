
# -*- mode: ruby -*-
# vim:ft=ruby:sw=3:et:

vm_nodes = {            # EDIT to specify VM node names, and their private IP (vboxnet#)
   'nomad-sv' => "192.168.55.10",
   'n1' => "192.168.55.11",
   'n2' => "192.168.55.12",
   'n3' => "192.168.55.13",
}
# EDIT or specify ENV variable to define OS-Type (see `vm_conf` below)
ostype = ENV['KUBE_OSTYPE'] || 'ubuntu16'
#ostype = ENV['KUBE_OSTYPE'] || 'centos7'

# VM config, format: <type-label> => [ 0:vagrant-box, 1:vm-net-iface, 2:vm-disk-controller, 3:vm-start-port, 4:vm-drives-map ]
# see https://atlas.hashicorp.com/search? for more VM images (ie. "box-names")
vm_conf = {
   'ubuntu16' => [ 'ubuntu/xenial64', 'enp0s8', 'SCSI', 2, { "sdc" => 40*1024, "sdd" => 30*1024 } ],
   'centos7'  => [ 'centos/7', 'eth1', 'IDE', 1, { "sdb" => 20*1024 } ],
}

# (internal variables)
mybox, myvmif, mycntrl, myport, extra_disks = vm_conf[ostype]
mystorage = "/dev/"+extra_disks.keys().join(",/dev/")
nomad_server_ip = vm_nodes["nomad-sv"]
#etcd_server_ip = vm_nodes["etcd"]

etc_hosts = ""
vm_nodes.each do |host,ip|
   etc_hosts += "\n#{ip}\t#{host}"
end

#
# Install scriplets
#
# install_prereqs - usually run first, does the base packages install and configuration
$install_prereqs = <<SCRIPT
echo ':: Installing Prerequisites ...'
if [ -d /etc/apt/sources.list.d ]; then # ----> Ubuntu/Debian distro
   export DEBIAN_FRONTEND=noninteractive
   apt-get clean && apt-get update && \
   apt-get install -y apt-transport-https unzip vim ca-certificates software-properties-common lsb-release curl linux-image-$(uname -r)
sudo apt-get update
elif [ -d /etc/yum.repos.d ]; then      # ----> CentOS/RHEL distro
   sed -i -e 's/^SELINUX=enforcing/SELINUX=disabled  # VAGRANT/' /etc/selinux/config && \
   setenforce 0
   yum make clean && yum makecache fast && \
   yum install -y curl make net-tools bind-utils
else    # ------------------------------------> (unsuported)
   echo "Your platform is not supported" >&2
   exit 1
fi
SCRIPT

# install_docker
$install_docker = <<SCRIPT
echo "Installing Docker..."
if [[ -f /etc/apt/sources.list.d/docker.list ]]; then
    echo "Docker repository already installed; Skipping"
else
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
    sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
    sudo apt-get update
fi

sudo DEBIAN_FRONTEND=noninteractive apt-get install -y docker-ce

# Restart docker to make sure we get the latest version of the daemon if there is an upgrade
sudo service docker restart

# Make sure we can actually use docker as the vagrant user
SCRIPT

$install_nomad_and_consul = <<SCRIPT
cd /tmp/
echo "Fetching Consul..."
curl -sSL https://releases.hashicorp.com/consul/1.0.0/consul_1.0.0_linux_amd64.zip > consul.zip

echo "Installing Consul..."

sudo mkdir -p /etc/consul.d
sudo chmod a+w /etc/consul.d

unzip consul.zip
sudo install consul /usr/bin/consul
rm consul.zip

hostname -I | grep -wq #{nomad_server_ip}
if [ $? -eq 0 ]; then

sudo mkdir -p /etc/consul-server
sudo chmod a+w /etc/consul-server

(
cat <<-EOF
	[Unit]
	Description=consul agent
	Requires=network-online.target
	After=network-online.target
	
	[Service]
	Restart=on-failure
	ExecStart=/usr/bin/consul agent -server -bootstrap-expect=1 -bind="#{nomad_server_ip}" -data-dir=/etc/consul-server -config-dir=/etc/consul.d
	ExecReload=/bin/kill -HUP $MAINPID
	
	[Install]
	WantedBy=multi-user.target
EOF
) | sudo tee /etc/systemd/system/consul.service
sudo systemctl enable consul.service
sudo systemctl start consul

else 

sudo mkdir -p /etc/consul-client
sudo chmod a+w /etc/consul-client

(
cat <<-EOF
        [Unit]
        Description=consul agent
        Requires=network-online.target
        After=network-online.target

        [Service]
        Restart=on-failure
        ExecStart=/usr/bin/consul agent -bind=$1 -data-dir=/etc/consul-server -config-dir=/etc/consul.d
        ExecReload=/bin/kill -HUP $MAINPID

        [Install]
        WantedBy=multi-user.target
EOF
) | sudo tee /etc/systemd/system/consul.service
sudo systemctl enable consul.service
sudo systemctl start consul
consul join #{nomad_server_ip}
fi 

for bin in cfssl cfssl-certinfo cfssljson
do
	echo "Installing $bin..."
	curl -sSL https://pkg.cfssl.org/R1.2/${bin}_linux-amd64 > /tmp/${bin}
	sudo install /tmp/${bin} /usr/local/bin/${bin}
done

# Download Nomad
NOMAD_VERSION=0.7.0

echo "Fetching Nomad..."
cd /tmp/
curl -sSL https://releases.hashicorp.com/nomad/${NOMAD_VERSION}/nomad_${NOMAD_VERSION}_linux_amd64.zip -o nomad.zip

echo "Installing Nomad..."
unzip nomad.zip
sudo install nomad /usr/bin/nomad

sudo mkdir -p /etc/nomad.d
sudo chmod a+w /etc/nomad.d

# Set hostname's IP to made advertisement Just Work
#sudo sed -i -e "s/.*nomad.*/$(ip route get 1 | awk '{print $NF;exit}') nomad/" /etc/hosts
rm nomad.zip

hostname -I | grep -wq #{nomad_server_ip}
if [ $? -eq 0 ]; then

sudo mkdir -p /etc/n-server
sudo chmod a+w /etc/n-server

echo "Installing autocomplete..."
nomad -autocomplete-install
(
cat <<-_aeof
server {
  enabled          = true
  bootstrap_expect = 1
}
# Increase log verbosity
log_level = "DEBUG"

# Setup data dir
data_dir = "/etc/n-server"
bind_addr = "#{nomad_server_ip}"
_aeof
) | sudo tee /etc/server.hcl

(
cat <<-EOF
        [Unit]
        Description=Nomad agent
        Requires=network-online.target
        After=network-online.target

        [Service]
        Restart=on-failure
        ExecStart=/usr/bin/nomad agent -config=/etc/server.hcl
        ExecReload=/bin/kill -HUP $MAINPID

        [Install]
        WantedBy=multi-user.target
EOF
) | sudo tee /etc/systemd/system/nomad.service
sudo systemctl enable nomad.service
sudo systemctl start nomad
else

sudo mkdir -p /etc/n-client
sudo chmod a+w /etc/n-client

(
cat <<-eof
client {
  enabled = true
  servers = ["#{nomad_server_ip}:4647"]

options {
    "driver.raw_exec.enable" = "1"
    "docker.privileged.enabled" = "true"
}
}
# Increase log verbosity
log_level = "DEBUG"

# Setup data dir
data_dir = "/etc/n-client"
bind_addr = "$1"

eof
) | sudo tee /etc/client.hcl

(
cat <<-EOF
        [Unit]
        Description=Nomad agent
        Requires=network-online.target
        After=network-online.target

        [Service]
        Restart=on-failure
        ExecStart=/usr/bin/nomad agent -config=/etc/client.hcl
        ExecReload=/bin/kill -HUP $MAINPID

        [Install]
        WantedBy=multi-user.target
EOF
) | sudo tee /etc/systemd/system/nomad.service
sudo systemctl enable nomad.service
sudo systemctl start nomad
fi
SCRIPT


$install_portworx =<<SCRIPT

hostname -I | grep -wq #{nomad_server_ip}
if [ $? -eq 0 ]; then

curl -o /tmp/portworx.nomad https://raw.githubusercontent.com/portworx/px-docs/gh-pages/scheduler/nomad/portworx.nomad
sed -i -e 's/\beth0\b/enp0s8/' \
     -e 's`http://127.0.0.1:8500`http://#{nomad_server_ip}:8500`' /tmp/portworx.nomad
nomad run -address=http://#{nomad_server_ip}:4646 /tmp/portworx.nomad

fi
SCRIPT

#
# VAGRANT SETUP
#
Vagrant.configure("2") do |config|

   vm_nodes.each do |host,ip|
      config.vm.define "#{host}" do |node|
         node.vm.box = "#{mybox}"
         node.vm.hostname = "#{host}"
         node.vm.network "private_network", ip: "#{ip}", :netmask => "255.255.255.0"

         node.vm.provider "virtualbox" do |v|
            v.gui = false
            v.memory = 6144

            # Extra customizations
            v.customize 'pre-boot', ["modifyvm", :id, "--cpus", "2"]
            v.customize 'pre-boot', ["modifyvm", :id, "--chipset", "ich9"]
            v.customize 'pre-boot', ["modifyvm", :id, "--audio", "none"]
            v.customize 'pre-boot', ["modifyvm", :id, "--usb", "off"]
            v.customize 'pre-boot', ["modifyvm", :id, "--accelerate3d", "off"]
            v.customize 'pre-boot', ["storagectl", :id, "--name", "#{mycntrl}", "--hostiocache", "on"]

            # force Virtualbox to sync the time difference w/ threshold 10s (defl was 20 minutes)
            v.customize [ "guestproperty", "set", :id, "/VirtualBox/GuestAdd/VBoxService/--timesync-set-threshold", 10000 ]

            # Net boot speedup (see https://github.com/mitchellh/vagrant/issues/1807)
            v.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
            v.customize ["modifyvm", :id, "--natdnsproxy1", "on"]

            if defined?(extra_disks)
            v.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
            v.customize ["modifyvm", :id, "--natdnsproxy1", "on"]

            if defined?(extra_disks)
               # NOTE: If you hit errors w/ extra disks provisioning, you may need to run "Virtual
               # Media Manager" via VirtualBox GUI, and manually remove $host_sdX drives.
               port = myport
               extra_disks.each do |hdd, size|
                  vdisk_name = ".vagrant/#{host}_#{hdd}.vdi"
                  unless File.exist?(vdisk_name)
                     v.customize ['createhd', '--filename', vdisk_name, '--size', "#{size}"]
                  end
                  v.customize ['storageattach', :id, '--storagectl', "#{mycntrl}", '--port', port, '--device', 0, '--type', 'hdd', '--medium', vdisk_name]
                  port = port + 1
               end
            end
         end

         # Custom post-install script below:
         node.vm.provision "shell", inline: <<-SHELL
            echo ':: Fixing ROOT access ...'
            echo root:Password1 | chpasswd
            sed -i -e 's/.*UseDNS.*/UseDNS no  # VAGRANT/' \
               -e 's/.*PermitRootLogin.*/PermitRootLogin yes  # VAGRANT/' \
               -e 's/.*PasswordAuthentication.*/PasswordAuthentication yes  # VAGRANT/' \
               /etc/ssh/sshd_config && systemctl restart sshd

            echo ':: Fixing /etc/hosts ...'
            sed -i -e 's/.*#{host}.*/# \\0  # VAGRANT/' /etc/hosts
            cat << xyz >> /etc/hosts
		#{etc_hosts}
xyz
SHELL

         node.vm.provision "shell", inline: $install_prereqs
         node.vm.provision "shell", inline: $install_docker
	 node.vm.provision :shell do |s|
	  s.inline = $install_nomad_and_consul 
	  s.args = "#{ip}"
	 end	 
         node.vm.provision "shell", inline: $install_portworx
        end  
      end
   end
 end
