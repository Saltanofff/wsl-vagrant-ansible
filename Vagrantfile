vm_ip       = "192.168.56.4"
vm_hostname = "ansible-client" 
ssh_pub_key = File.read(File.expand_path("~/.ssh/ansible_key.pub")).strip
vm_memory = "4096"
vm_cpu = 2
python_version = "3.12"

Vagrant.configure("2") do |config|
    config.vm.box = "ubuntu/jammy64"  
    config.vm.synced_folder ".", "/vagrant", disabled: true # action prevents Vagrant from attempting to sync the project directory, thereby bypassing the error.
    config.vm.network "private_network", ip: vm_ip
    config.vm.hostname = vm_hostname
    
    config.vm.provision "shell" do |s|
      s.args = [python_version]
      s.inline = <<-SHELL
      echo #{ssh_pub_key} >> /home/vagrant/.ssh/authorized_keys
      echo #{ssh_pub_key} >> /root/.ssh/authorized_keys
      add-apt-repository ppa:deadsnakes/ppa -y && apt-get update -y && apt-get install python$1 -y
      echo 'vagrant ALL=(ALL) NOPASSWD:ALL' | sudo tee /etc/sudoers.d/vagrant #disbale for a user password to pop-up when using sudo
      chmod 440 /etc/sudoers.d/vagrant
    SHELL
    end
    
    config.vm.provider "virtualbox" do |vb|
      vb.gui = false
      vb.memory = vm_memory  
      vb.cpus = vm_cpu
      vb.customize ["modifyvm", :id, "--uartmode1", "disconnected"]
    end
end
