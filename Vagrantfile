Vagrant.configure(2) do |config|
  config.vm.box = "bento/centos-7.2"
  config.ssh.insert_key = false

  config.vm.define "ng01" do |config|
    config.vm.hostname = "ng01"
    config.vm.network "private_network", ip: "192.168.33.21", virtualbox__intnet: "ha-nagios"
  end

  config.vm.define "ng02" do |config|
    config.vm.hostname = "ng02"
    config.vm.network "private_network", ip: "192.168.33.22", virtualbox__intnet: "ha-nagios"
  end

  config.vm.define "sv01" do |config|
    config.vm.hostname = "sv01"
    config.vm.network "private_network", ip: "192.168.33.11", virtualbox__intnet: "ha-nagios"
  end

  config.vm.define "proxy" do |config|
    config.vm.hostname = "proxy"
    config.vm.network "forwarded_port", guest: 80, host: 8080
    config.vm.network "private_network", ip: "192.168.33.10", virtualbox__intnet: "ha-nagios"
  end

  config.vm.provision "shell", inline: <<-SHELL
    sudo yum -y install vim-enhanced mailx nc rsync tcpdump patch sysstat bash-completion epel-release
    sudo yum -y install ansible jq colordiff
    sudo cp -f /vagrant/hosts /etc/hosts
    sudo cp -f /vagrant/inventory /etc/ansible/hosts
    sudo chmod -x /etc/ansible/hosts
  SHELL

  config.vm.provider :virtualbox do |v|
    v.linked_clone = true
  end
end
