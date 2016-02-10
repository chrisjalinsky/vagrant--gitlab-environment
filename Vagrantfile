VAGRANTFILE_API_VERSION = "2"
BOX = "ubuntu/trusty64"

base_dir = File.expand_path(File.dirname(__FILE__))

# Cluster, VM and network settings
NETWORK_SUBNET = "192.168.1"
NETWORK_DOMAIN = "lan"
NUM_HACKS = 1
HACK_IPS_START = 10
HACK_MEMORY = 6124
HACK_CPU = 2

ansible_provision = proc do |ansible|
  ansible.playbook = 'site.yml'
  # Note: Can't do ranges like mon[0-2] in groups because
  # these aren't supported by Vagrant, see
  # https://github.com/mitchellh/vagrant/issues/3539
  ansible.groups = {
    'hack_servers'    => (0..NUM_HACKS - 1).map { |j| "hack#{j}.#{NETWORK_DOMAIN}" },
    'all:children'      => ["hack_servers"],
  }

  # In a production deployment, these should be secret
  ansible.extra_vars = {
    cluster_network: "#{NETWORK_SUBNET}.0/24",
  }
  ansible.limit = 'all'
end

def create_vmdk(name, size)
  dir = Pathname.new(__FILE__).expand_path.dirname
  path = File.join(dir, '.vagrant', name + '.vmdk')
  `vmware-vdiskmanager -c -s #{size} -t 0 -a scsi #{path} \
   2>&1 > /dev/null` unless File.exist?(path)
end

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  config.ssh.insert_key = false
  config.vm.box = BOX

  if Vagrant.has_plugin?("vagrant-cachier")
    config.cache.scope = :machine
    config.cache.enable :apt
  end
  
  (0..NUM_HACKS - 1).each do |i|
    config.vm.define "hack#{i}.#{NETWORK_DOMAIN}" do |ae|
      ae.vm.hostname = "hack#{i}.#{NETWORK_DOMAIN}"
      ae.vm.network :private_network, ip: "#{NETWORK_SUBNET}.#{HACK_IPS_START + i}"

      # Virtualbox
      ae.vm.provider :virtualbox do |vb|
        #(0..1).each do |d|
        #  vb.customize ['createhd',
        #                '--filename', "disk-#{i}-#{d}",
        #                '--size', '11000']
        #  # Controller names are dependent on the VM being built.
        #  # It is set when the base box is made in our case ubuntu/trusty64.
        #  # Be careful while changing the box.
        #  vb.customize ['storageattach', :id,
        #                '--storagectl', "SATAController",
        #                '--port', 3 + d,
        #                '--device', 0,
        #                '--type', 'hdd',
        #                '--medium', "disk-#{i}-#{d}.vdi"]
        #end
        vb.customize ['modifyvm', :id, '--memory', "#{HACK_MEMORY}"]
      end

      # VMware
      ae.vm.provider :vmware_fusion do |v|
        (0..1).each do |d|
          v.vmx["scsi0:#{d + 1}.present"] = 'TRUE'
          v.vmx["scsi0:#{d + 1}.fileName"] =
            create_vmdk("disk-#{i}-#{d}", '11000MB')
        end
        v.vmx['memsize'] = "#{HACK_MEMORY}"
      end

      # Libvirt
      driverletters = ('b'..'z').to_a
      ae.vm.provider :libvirt do |lv|
        (0..1).each do |d|
          lv.storage :file, :device => "vd#{driverletters[d]}", :path => "disk-#{i}-#{d}.disk", :size => '11G'
        end
        lv.memory = HACK_MEMORY
      end
      
      ae.vm.provision 'ansible', &ansible_provision if i == (NUM_HACKS - 1)
    end
  end
end