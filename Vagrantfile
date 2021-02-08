# Define the operating system of the machines and the name of the internal network
INTRANET="innet"
PUBNET="pubnet"
DISTRO="generic/debian10"

# Ansible ctes, including the pass file, remote pass and username
ANSIBLE_PASS="/vagrant/ansible_passphrase.txt"
ANSIBLE_USER="ansible"
ANSIBLE_REMOTE="ygqD-jh3LII1oNhurzQwAhoYe"

# Client user
CLIENT_UNAME="user"
CLIENT_PASSWORD="password"

# Admin user
ADMIN_UNAME="admin"
ADMIN_PASSWORD="admin"
ADMIN_REMOTE_PASSWORD="CKLwhksWtn"

# VM properties
CPU_CAP_PERCENTAGE=60
MEM_CAP=512
CLIENT_MEM_CAP=1024
VRAM_CAP=8
CLIENT_VRAM_CAP=64

# Master VM for ansible
master = {
    "ansibleadmin" => { 
        :ip => "192.168.33.5",
        :pub_ip => "192.167.33.71",
    }
}

# Set up VMs
hosts = {
    "webserverhost" => { 
        :ip => "192.168.33.35",
        :pub_ip => "192.167.33.81",
    },
    "certhost" => { :ip => "192.168.33.15"},
    #"backuphost" => { :ip => "192.168.33.55" },
    #"nwfirewall" => { :ip => "192.168.33.45"},
    "dbhost" => { :ip => "192.168.33.25"},
}

clients = {
    #"client" => { :pub_ip => "192.167.33.91" },
}

Vagrant.configure(2) do |config|
    # Define VM provider and update the os
    config.vm.provider "virtualbox"
    config.vagrant.plugins = "vagrant-vbguest"

    # US keyboard
    ENV['LC_ALL']="en_US.UTF-8"

    # Limit resource usage
	config.vm.provider "virtualbox" do |vb|
		vb.customize ["modifyvm", :id, "--cpuexecutioncap", CPU_CAP_PERCENTAGE]
	end 

    # Iterate hosts
    hosts.each do |hostname, net|
        config.vm.define hostname do |hostconf|
            # For each VM set OS, hostname and the network
            hostconf.vm.box = DISTRO
            hostconf.vm.hostname = hostname

            hostconf.vm.network "private_network",
                ip: "#{net[:ip]}",
                virtualbox__intnet: INTRANET

            if net.key?(:pub_ip) then
                hostconf.vm.network "private_network", 
                ip: "#{net[:pub_ip]}",
                virtualbox__intnet: PUBNET
            end

            hostconf.vm.provider "virtualbox" do |vb|
                vb.customize ["modifyvm", :id, "--name", "#{hostname}"]
                vb.customize ["modifyvm", :id, "--vram", VRAM_CAP]
                vb.memory = MEM_CAP
            end 

            if hostname == "webserverhost" then
                hostconf.vm.synced_folder "./webserver", "/vagrant", SharedFoldersEnableSymlinksCreate: false

                hostconf.vm.provision "shell", inline: <<-SHELL
                    sudo cp -r /vagrant "/home/webserver"
                SHELL
            end    

             if hostname == "dbhost" then
                hostconf.vm.synced_folder "./db", "/vagrant", SharedFoldersEnableSymlinksCreate: false

                hostconf.vm.provision "shell", inline: <<-SHELL
                    sudo cp -r /vagrant "/home/db"
                SHELL
            end   

            if hostname == "certhost" then
                hostconf.vm.synced_folder "./core_ca", "/vagrant", SharedFoldersEnableSymlinksCreate: false

                hostconf.vm.provision "shell", inline: <<-SHELL
                    sudo cp -r /vagrant "/home/vagrant/core_ca"
                SHELL
            end             

            # When the machine spawns execute the following shell script (set up ansible)
            hostconf.vm.provision "shell", inline: <<-SHELL
                # --gecos doesn't ask for finger information --disabled-password login only possible with SSH key
                sudo adduser --disabled-password --gecos "" ansible

                # give a key to ansible user with chpasswd
                echo "ansible:#{ANSIBLE_REMOTE}" | sudo chpasswd

                # give the ansible user passwordless sudo
                echo 'ansible ALL=(ALL) NOPASSWD:ALL' | sudo EDITOR='tee -a' visudo

                # disable login for root
                sudo passwd -d root

                # remove sensitive data
                history -c; unset HISTFILE; rm -f ~/.bash_history;
            SHELL

            # Add hostnames with the following shell script (add hostname for ip resolution)
            hosts.each do |peer_hostname, peer_info|
                hostconf.vm.provision "shell", inline: <<-SHELL
                    # add all hostnames to hosts
                    echo "#{peer_info[:ip]} #{peer_hostname}" | sudo tee -a /etc/hosts
                SHELL

                if peer_info.key?(:pub_ip) then
                    hostconf.vm.provision "shell", inline: <<-SHELL
                        # Add hostname
                        # use :ip for internal hosts
                        echo "#{peer_info[:ip]} imovies.ch" | sudo tee -a /etc/hosts
                        echo "#{peer_info[:ip]} www.imovies.ch" | sudo tee -a /etc/hosts
                    SHELL
                end
            end
        end
    end
    
    # Ansible master
    master.each do |mastername, net|
        config.vm.define mastername do |masterconf|
            masterconf.vm.box = DISTRO
            masterconf.vm.hostname = mastername

            masterconf.vm.network "private_network",
                ip: "#{net[:ip]}",
                virtualbox__intnet: INTRANET

            if net.key?(:pub_ip) then
                masterconf.vm.network "private_network", 
                ip: "#{net[:pub_ip]}",
                virtualbox__intnet: PUBNET
            end

            # Define a synced folder /shared that will appear in master as /vagrant. Use to pass code, files and playbook to hosts
            masterconf.vm.synced_folder "./shared", "/vagrant", SharedFoldersEnableSymlinksCreate: false

            masterconf.vm.provider "virtualbox" do |vb|
                vb.customize ["modifyvm", :id, "--name", "#{mastername}"]
                vb.customize ["modifyvm", :id, "--vram", VRAM_CAP]
                vb.memory = MEM_CAP
            end 

            # Once the master spawns execute the following shell script (set up ansible)
            masterconf.vm.provision "shell", inline: <<-SHELL
                # Install ansible and sshpass, use noninteractive to avoid promts
                sudo apt-get update
                DEBIAN_FRONTEND=noninteractive sudo -E apt-get install -y ansible sshpass
                
                # Add ansible user
                sudo adduser --disabled-password --gecos "" ansible
                echo 'ansible ALL=(ALL) NOPASSWD: ALL' | sudo EDITOR='tee -a' visudo

                # Add user for remote admin
				sudo adduser --disabled-password --gecos "" #{ADMIN_UNAME}
				echo "#{ADMIN_UNAME}:#{ADMIN_REMOTE_PASSWORD}" | sudo chpasswd
				sudo usermod -aG sudo #{ADMIN_UNAME}
                
                # Remove root user passwordless sudo
                sudo passwd -d root

                # Generate SSH keys for ansible remote configuration, using ecdsa 521bits 128round and the specified ansible_passphrase
                sudo rm -f /home/ansible/.ssh/id_rsa
                sudo su - ansible -c "ssh-keygen -t ecdsa -b 521 -a 128 -f /home/ansible/.ssh/id_rsa -N $(cat #{ANSIBLE_PASS})" 
            SHELL

            masterconf.vm.provision "shell", inline: <<-SHELL
                # Copy ansible playbooks and generate ansible inventory
                sudo cp -r /vagrant/ansible/* "/home/ansible"
                sudo cp -r /vagrant/*.txt "/home/passphrase"

                # add Ansible host itself to inventory
                echo -e '\n[master]\n#{mastername} ansible_connection=local' | sudo tee -a "/home/ansible/inventory"
                echo -e "\n[rest]" | sudo tee -a "/home/ansible/inventory"                 
            SHELL

            # Add hostnames, install SSH keys
            hosts.each do |hostname, hostnet|
                masterconf.vm.provision "shell", inline: <<-SHELL
                    # Add hostname to hosts and ansible inventory
                    echo "#{hostnet[:ip]} #{hostname}" | sudo tee -a /etc/hosts
					echo "#{hostname}" | sudo tee -a "/home/ansible/inventory"

                    # copy ssh id of master to ansible@hostname
					sudo sshpass -p "#{ANSIBLE_REMOTE}" ssh-copy-id -i /home/ansible/.ssh/id_rsa -o StrictHostKeyChecking=accept-new ansible@#{hostname}
					
                    # accept host key by ssh with ansible@ansible_master to ansible@hostname
					sudo su - ansible -c 'sshpass -P "Enter" -p $(cat #{ANSIBLE_PASS}) ssh -o StrictHostKeyChecking=accept-new ansible@#{hostname} exit'

                    # Remove temporary password for ansible users of remote servers + remove password authentication
                    sudo sshpass -p "#{ANSIBLE_REMOTE}" ssh ansible@#{hostname} <<-EOF
sudo passwd -d ansible
sudo usermod --lock ansible
EOF
                SHELL
            end

            # execute playbooks, first create an agent that has sshkeys in memory and unlock the keys
            masterconf.vm.provision "shell", inline: <<-SHELL
                sudo -i -u ansible bash <<-EOF
exec ssh-agent bash -s 10<&0 << EOF2
sshpass -P 'Enter' -p $(cat #{ANSIBLE_PASS}) ssh-add ~/.ssh/id_rsa
exec bash <&10-
EOF2
ansible-galaxy install -p roles -r requirements.yml -f
ansible-galaxy install nickjj.iptables

ansible-playbook -e 'FORCE_ROOT_CA_CERT_REGEN=true' -i inventory list.yml
history -c; unset HISTFILE; rm -f ~/.bash_history;
EOF
            history -c; unset HISTFILE; rm -f ~/.bash_history;
            SHELL

        end 
    end


    # Iterate hosts
    clients.each do |clientname, net|
        config.vm.define clientname do |clientconf|
            # For each VM set OS, hostname and the network
            clientconf.vm.box = DISTRO
            clientconf.vm.hostname = clientname

            clientconf.vm.network "private_network",
                ip: "#{net[:pub_ip]}",
                virtualbox__intnet: PUBNET

            clientconf.vm.synced_folder "./client", "/vagrant", SharedFoldersEnableSymlinksCreate: false

            # Install root certificate on client
            clientconf.vm.provision "shell", inline: <<-SHELL
                sudo cp /vagrant/sshkey_store/client/root_cert.pem /usr/local/share/ca-certificates/root_cert.crt
                sudo update-ca-certificates
            SHELL

            hosts.each do |host_name, host_net|
                if host_net.key?(:pub_ip)
                    clientconf.vm.provision "shell", inline: <<-SHELL
                        # Add hostname
						echo "#{host_net[:pub_ip]} imovies.ch" | sudo tee -a /etc/hosts
						echo "#{host_net[:pub_ip]} www.imovies.ch" | sudo tee -a /etc/hosts
                    SHELL
                end
            end

            clientconf.vm.provider "virtualbox" do |vb|
                # vb.gui = true
				vb.customize ["modifyvm", :id, "--vram", CLIENT_VRAM_CAP]
				vb.customize ["modifyvm", :id, "--name", "#{clientname}"]
				vb.memory = CLIENT_MEM_CAP
            end 

            clientconf.vm.provision "shell", inline: <<-SHELL
                # Add user for administrating our infrastructure from remote
                sudo adduser --disabled-login --gecos "User" #{ADMIN_UNAME}
                echo "#{ADMIN_UNAME}:#{ADMIN_PASSWORD}" | sudo chpasswd
                # Add a normal user
                sudo adduser --disabled-login --gecos "User" #{CLIENT_UNAME}
                echo "#{CLIENT_UNAME}:#{CLIENT_PASSWORD}" | sudo chpasswd
                # Install sshpass
                DEBIAN_FRONTEND=noninteractive sudo -E apt-get install -y sshpass
            SHELL

            master.each do |mastername, masternet|
                clientconf.vm.provision "shell", inline: <<-SHELL
                    # Add hostname
					echo "#{masternet[:pub_ip]} #{mastername}" | sudo tee -a /etc/hosts

                    # Generate admin key material, install them on the config servers and
                    # disable password login
                    sudo -i -u #{ADMIN_UNAME} bash <<-EOF
mkdir ~/.ssh;
chmod 700 ~/.ssh
#ssh-keygen -t ecdsa -b 521 -f ~/.ssh/imovies_#{mastername} -P '' -C '#{ADMIN_UNAME}@imovies.ch'
cp /vagrant/modrsa ~/.ssh/imovies_#{mastername}
cp /vagrant/modrsa.pub ~/.ssh/imovies_#{mastername}.pub
sshpass -p '#{ADMIN_REMOTE_PASSWORD}' ssh-copy-id -i ~/.ssh/imovies_#{mastername} -o StrictHostKeyChecking=accept-new #{ADMIN_UNAME}@#{mastername}
# scp -i ~/.ssh/imovies_#{mastername} #{ADMIN_UNAME}@#{mastername}:/home/ansible/ca_admin_cert_with_priv_key.pfx ~/ca_admin_cert_with_priv_key_#{mastername}.pfx
EOF

                    # Copy the generated key material to the shared directory.
                    mkdir /vagrant/sshkey_store/#{clientname}
					sudo cp /home/#{ADMIN_UNAME}/.ssh/imovies_#{mastername} /vagrant/sshkey_store/#{clientname}/

                    sudo cp -r /vagrant/* "/home/vagrant"
                SHELL
            end 

            clientconf.vm.provision "shell", inline: <<-SHELL
                # Upgrade all packages, install user interface, Firefox, and other needed tools.
                sudo apt-get update
                DEBIAN_FRONTEND=noninteractive sudo -E apt-get upgrade -y
                DEBIAN_FRONTEND=noninteractive sudo -E apt-get install -y x-window-system lightdm xfce4 firefox-esr libnss3-tools --no-install-recommends

                # Use default XFCE panel without user prompt
                sudo cp /etc/xdg/xfce4/panel/default.xml /etc/xdg/xfce4/xfconf/xfce-perchannel-xml/xfce4-panel.xml

                # Activate autologin
                sudo mkdir /etc/lightdm/lightdm.conf.d
                sudo bash -c "echo -e '[SeatDefaults]\nautologin-user=#{CLIENT_UNAME}' > /etc/lightdm/lightdm.conf.d/12-autologin.conf"

                # Switch to GUI.
                sudo systemctl set-default graphical.target
                systemctl isolate graphical.target

                history -c; unset HISTFILE; rm -f ~/.bash_history;
            SHELL

            master.each do |mastername, net|
                config.vm.define mastername do |masterconf|
                    masterconf.vm.provision "shell", inline: <<-SHELL
                        sudo sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/g' /etc/ssh/sshd_config
                        sudo service ssh restart
                    SHELL
                end
            end
        end
    end
end
