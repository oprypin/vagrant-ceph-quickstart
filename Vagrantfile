CEPH_RELEASE = "infernalis"


# Vagrant.require_plugin "vagrant-vbguest"

Vagrant.configure(2) do |config|
  config.vm.box = "debian/jessie64"

  config.vm.synced_folder ".", "/vagrant", disabled: true
  config.vm.synced_folder "ssh", "/home/vagrant/ssh"

  config.vm.provision :shell, inline: <<-SHELL
    apt-get update
    apt-get install -y ntp openssh-server

    systemctl start ntp
    systemctl enable ntp
  SHELL

  config.vm.network :public_network

  config.vm.provider "virtualbox" do |vb|
    #vb.gui = false
    vb.memory = 2048
    vb.cpus = 1
    vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
  end

  (1..3).each do |i|
    config.vm.define "node#{i}" do |node|
      node.vm.host_name = "node#{i}"

      node.vm.provision :shell, privileged: false, inline: <<-SHELL
        if [ ! -f ~/ssh/id_rsa.pub ]; then
          ssh-keygen -t rsa -N '' -f ~/ssh/id_rsa -C "ceph-vagrant"
        fi
        cat ~/ssh/id_rsa.pub >>~/.ssh/authorized_keys
      SHELL

      if i > 1
        disk = "disk/osd#{i-2}.vdi"
        node.vm.provider "virtualbox" do |vb|
          unless File.exist?(disk)
            vb.customize ['createmedium', '--filename', disk, '--size', 500]
          end
          vb.customize ['storageattach', :id, '--storagectl', 'SATA Controller', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', disk]
        end

        node.vm.provision :shell, inline: <<-SHELL
          apt-get install -y parted
          /sbin/parted /dev/sdb mklabel msdos
          /sbin/parted /dev/sdb mkpart primary 512 100%
        SHELL
      end
    end
  end

  config.vm.define "admin-node", primary: true do |node|
    node.vm.host_name = "admin-node"

    node.vm.synced_folder "cluster", "/home/vagrant/cluster"

    node.vm.provision :shell, inline: <<-SHELL
      curl https://download.ceph.com/keys/release.asc | apt-key add -
      echo deb http://download.ceph.com/debian-#{CEPH_RELEASE}/ $(lsb_release -sc) main >/etc/apt/sources.list.d/ceph.list
      apt-get update
      apt-get install -y ceph-deploy
    SHELL

    node.vm.provision :shell, privileged: false, inline: <<-SHELL
      set -e  # stop on any error
      set -x  # echo every command

      if [ ! -f ~/.ssh/config ]; then
        mv ~/ssh/id_rsa ~/.ssh/
        rm ~/ssh/id_rsa.pub

        echo 'Host node*'                      >~/.ssh/config
        echo '  User vagrant'                 >>~/.ssh/config
        echo '  StrictHostKeyChecking no'     >>~/.ssh/config
        echo '  UserKnownHostsFile=/dev/null' >>~/.ssh/config
      fi

      if [ ! -d cluster ]; then
        mkdir cluster
      else
        ceph-deploy purge node{1..3}
        ceph-deploy purgedata node{1..3}
        ceph-deploy forgetkeys
      fi

      cd cluster

      ceph-deploy new node1
      echo 'osd pool default size = 2' >>ceph.conf

      ceph-deploy install --release #{CEPH_RELEASE} admin-node node{1..3}

      ceph-deploy mon create-initial

      for i in 2 3; do
        #ssh node$i sudo chown ceph: /dev/sdb1
        ceph-deploy osd create --fs-type ext4 node$i:/dev/sdb1
      done

      ceph-deploy admin admin-node node{1..3}
      sudo chmod +r /etc/ceph/ceph.client.admin.keyring

      ceph health

      ceph-deploy mds create node1
      ceph-deploy rgw create node1
    SHELL
  end
end


ENV["LC_ALL"] = "en_US.UTF-8"


# References:
# http://docs.ceph.com/docs/master/start/quick-start-preflight/
# http://docs.ceph.com/docs/master/start/quick-ceph-deploy/
# https://github.com/infn-bari-school/ceph/wiki/How-to-install-Ceph-cluster-%28version-Infernalis%29
# https://gist.github.com/leifg/4713995
# http://superuser.com/questions/125324/how-can-i-avoid-sshs-host-verification-for-known-hosts