BOX_IMAGE = "bento/ubuntu-18.04"
KUBERNETES_VERSION = "stable-1.13"
KUBEADM_VERSION = "1.13.5-00"
PHP_VERSION= "7.3"

Vagrant.configure("2") do |config|
  config.vm.provider :virtualbox do |v|
    v.name = "DockerServer"
    v.memory = 4096
    v.cpus = 1
    unless File.exist?('./docker-data.vdi')
      v.customize ['createhd', '--filename', './docker-data.vdi', '--size', 160 * 1024]
    end
    v.customize ['storageattach', :id,  '--storagectl', 'SATA Controller', '--port', 2, '--device', 0, '--type', 'hdd', '--medium', './docker-data.vdi']
  end

  config.vm.provision :shell, privileged: true, inline: $install_common_tools

  config.vm.define :master do |master|
    master.vm.box = BOX_IMAGE
    master.vm.hostname = "docker-server"
    master.vm.network :private_network, ip: "192.168.56.10"
    master.vm.provision :shell, privileged: false, inline: $provision_master
    master.vm.provider :virtualbox do |vb|
      vb.cpus = 2
    end
  end

  config.vm.provision "shell", inline: $install_multicast
end

$install_common_tools = <<-SCRIPT
# disable swap
swapoff -a
sed -i '/swap/d' /etc/fstab

#Create data hard disk
printf "o\nn\np\n1\n\n\nw\n" | fdisk /dev/sdb
mkfs.ext4  /dev/sdb1
mkdir /media/data
#Add data to mount /media/data
cat <<EOT >> /etc/fstab
/dev/sdb1 /media/data ext4 defaults 0 2
EOT
mount /dev/sdb1 /media/data

# Install kubeadm, kubectl and kubelet
export DEBIAN_FRONTEND=noninteractive
apt-get -qq install ebtables ethtool
sudo add-apt-repository ppa:ondrej/php
apt-get -qq update
apt-get -qq upgrade
apt-get -qq install -y net-tools docker.io vim openssh-server samba curl wget php7.3-cli php7.3-zip unzip php7.3-mbstring php7.3-curl php7.3-dom php7.3-gd

#Install composer
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php -r "if (hash_file('sha384', 'composer-setup.php') === 'e0012edf3e80b6978849f5eff0d4b4e4c79ff1609dd1e613307e16318854d24ae64f26d17af3ef0bf7cfb710ca74755a') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"
php composer-setup.php --install-dir=/usr/local/bin --filename=composer
php -r "unlink('composer-setup.php');"

curl -L "https://github.com/docker/compose/releases/download/1.25.3/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose


##Add samba configuration
cat <<EOT >> /etc/samba/smb.conf
[docker_projects]
    comment = Docker projects
    path = /media/data/docker_projects
    browseable = yes
    writeable = yes
    guest ok = yes
    create mask = 0777
    directory mask = 07777
EOT

##Add Keyboard configuration
cat <<EOT > /etc/default/keyboard
  # KEYBOARD CONFIGURATION FILE
  # Consult the keyboard(5) manual page.
  XKBMODEL="pc105"
  XKBLAYOUT="es"
  XKBVARIANT=""
  XKBOPTIONS=""
  BACKSPACE="guess"
EOT

##Add locale configuration
cat <<EOT > /etc/default/locale
  #  File generated by update-locale
  LANG="es_ES.UTF-8"
EOT

##Enable docker on-startup
systemctl enable docker

#Configure user "vagrant"
usermod -aG docker vagrant
su -l vagrant -c "ssh-keygen -b 2048 -t rsa -f /home/vagrant/.ssh/id_rsa -q -N \"\""

#Create workdir folder for docker
mkdir /media/data/docker_projects
chmod -R 775 /media/data/docker_projects
chown -R vagrant:vagrant /media/data/docker_projects
ln -s /media/data/docker_projects /home/vagrant/docker
chown -R vagrant:vagrant /home/vagrant/docker

#Change docker folder
service docker stop
mv /var/lib/docker /media/data/docker_server
ln -s /media/data/docker_server /var/lib/docker
service docker start

#Change the default "/tmp" folder
mv /tmp /media/data/tmp
ln -s /media/data/tmp /tmp

#Add welcome message, before login
cat <<EOT >> /etc/issue
  Welcome to environment to work with Docker. Access to the machine through IP 192.168.56.10 using:
     SSH   --> ssh vagrant@192.168.56.10
     SAMBA --> \\\\192.168.56.10\\docker_projects (as vagrant user)
EOT

SCRIPT

$provision_master = <<-SHELL
OUTPUT_FILE=/vagrant/join.sh
rm -rf $OUTPUT_FILE

SHELL

$install_multicast = <<-SHELL
apt-get -qq install -y avahi-daemon libnss-mdns
SHELL