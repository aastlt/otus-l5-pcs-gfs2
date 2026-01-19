Vagrant.configure("2") do |config|
  config.vm.box = "generic/debian12"
  config.ssh.insert_key = false

  # Настройка таргета (Xранилище)
  config.vm.define "iscsi-target" do |storage|
    storage.vm.hostname = "iscsi-target"
    storage.vm.network "private_network", ip: "192.168.56.10"
    storage.vm.disk :disk, name: "iscsi_shared_disk", size: "5GB"

    storage.vm.provider "virtualbox" do |vb|
      vb.name = "gfs2-storage-node"
      vb.memory = 2048
      vb.cpus = 2
      vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
    end
  end

  # Настройка нод GFS2 (Кластер)
  (1..3).each do |i|
    config.vm.define "gfs2-node-0#{i}" do |node|
      node.vm.hostname = "gfs2-node-0#{i}"
      node.vm.network "private_network", ip: "192.168.56.1#{i}"
      node.vm.provider "virtualbox" do |vb|
        vb.name = "gfs2-node-0#{i}"
        vb.memory = 2048
        vb.cpus = 2
        vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
      end

      if i == 3
        node.vm.provision "ansible" do |ansible|
          ansible.playbook = "site.yml"
          ansible.inventory_path = "inventory"
          ansible.limit = "all"
          ansible.host_key_checking = false
          ansible.extra_vars = { ansible_python_interpreter: "/usr/bin/python3" }
        end
      end
    end
  end
end
