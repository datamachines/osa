require 'tty-prompt'

openstack_release = ENV["OPENSTACK_RELEASE"]

if !ENV["ANSIBLE_TAGS"].nil?
  ansible_tags = ENV["ANSIBLE_TAGS"].split(",")
end

def get_ansible_tags(openstack_release)
  prompt = TTY::Prompt.new
  tags = prompt.multi_select("What would you like to do?", per_page: 12) do |menu|
    # menu.default 1
    # menu.choice "Full Install", "install_full"
    
    menu.choice "First run actions (bootstrap_aio, setup_hosts, setup_infrastructure, setup_openstack)", "first_run"
    
    # Manila not available in rocky
    if openstack_release != "rocky"
      menu.choice "Install Additional Service: Manila", ["install_manila", "install_additional_services"]
    end

    menu.choice "Install Ceph", ["install_ceph", "install_additional_services"]
    menu.choice "Reconfigure Services", "reconfigure_services"
    menu.choice "------- TOOLS -------", disabled: ""
    menu.choice "Create containers", "create_containers"
    menu.choice "HAProxy Config Update", "haproxy_config"
    menu.choice "Boostrap AIO", "bootstrap_aio"
    menu.choice "Setup Hosts", "setup_hosts"
    menu.choice "Setup Infrastructure", "setup_infrastructure"
    menu.choice "Setup Openstack", "setup_openstack"
  end
  # Along with menu choice, also tag with openstack_release
  return tags.append(openstack_release)
end

def get_openstack_release()
  prompt = TTY::Prompt.new
  return prompt.select("Which version of OpenStack should be targeted?") do |menu|
    menu.choice "Rocky", "rocky"
    menu.choice "Stein", "stein"
    menu.choice "Train", "train"
    menu.choice "Ussuri", "ussuri"
  end
end

if openstack_release.nil? && !ARGV.include?("box")
  openstack_release = get_openstack_release()
  ENV["OPENSTACK_RELEASE"] = openstack_release
end

if (ARGV.include?("up") || (ARGV.include?("provision") || ARGV.include?("--provision"))) && ansible_tags.nil?
  ansible_tags = get_ansible_tags(openstack_release)
  ENV["ANSIBLE_TAGS"] = ansible_tags.join(",")
end

Vagrant.configure("2") do |config|
  N = 1
  (1..N).each do |machine_id|
    config.vm.define "#{openstack_release}#{machine_id}" do |machine|
      machine.vm.box = "centos7:2004"
      machine.vm.synced_folder '.', '/vagrant', disabled: true
      machine.vm.hostname = "#{openstack_release}#{machine_id}"

      # If we end up running multiple releases next to each other, make sure networking does not collide
      if openstack_release.to_s.downcase == 'rocky'
        third_octect = "239"
        port_http   = 8080
        port_https  = 8443
      elsif openstack_release.to_s.downcase == 'stein'
        third_octect = "240"
        port_http   = 8081
        port_https  = 8444
      elsif openstack_release.to_s.downcase == 'train'
        third_octect = "241"
        port_http   = 8082
        port_https  = 8445
      elsif openstack_release.to_s.downcase == 'ussuri'
        third_octect = "242"
        port_http   = 8083
        port_https  = 8446
      else
        third_octect = "243"
        port_http   = 8084
        port_https  = 8447
      end

      machine.vm.network "private_network", ip: "192.168.#{third_octect}.#{10 + machine_id}"
      machine.vm.network "private_network", ip: "10.254.#{third_octect}.#{10 + machine_id}"
      machine.vm.network "forwarded_port", guest: 80, host: port_http
      machine.vm.network "forwarded_port", guest: 443, host: port_https

      machine.vm.provider "virtualbox" do |vb|
        # ----------------------------------------------------------
        # CPUs
        # ----------------------------------------------------------
        vb.cpus = 4

        # ----------------------------------------------------------
        # Memory
        # ----------------------------------------------------------
        vb.memory = "8192"

        # ----------------------------------------------------------
        # Disks
        # ----------------------------------------------------------

        # Create a storage controller
        # vb.customize ['storagectl', :id, '--name', 'SATA Controller', '--add', 'sata', '--controller', 'IntelAHCI']

        # Add disks to the controler
        (0..4).each do |disk_id|
          disk = "tmp/machine-#{openstack_release}-#{machine_id}-osd#{disk_id}.vdi"
          
          # If our install drive, make 100G, else make 5G
          if disk_id == 0
            size = 100 * 1024
          else
            size = 5 * 1024
          end

          # Create the disk in VB if needed
          unless File.exist?(disk)
            # Create a storage controller when we create our first disk
            if disk_id == 0
              vb.customize ['storagectl', :id, '--name', 'SATA Controller', '--add', 'sata', '--controller', 'IntelAHCI']
            end
            vb.customize ['createhd', '--filename', disk, '--variant', 'Fixed', '--size', size]
          end

          # Attach the disk to the storage controller we created
          vb.customize ['storageattach', :id,  '--storagectl', 'SATA Controller', '--port', disk_id, '--device', 0, '--type', 'hdd', '--medium', disk]
        end
      end

      # Only execute once the Ansible provisioner,
      # when all the machines are up and ready.
      if ansible_tags.nil? || ansible_tags.empty?
        puts("Running with no supplimental ansible tags")
      else
        puts("Ansible will run with tags: ", ansible_tags)
      end

      # We only want to run ansible provisioning once
      if machine_id == N
        machine.vm.provision :ansible do |ansible|
          # Disable default limit to connect to all the machines
          ansible.compatibility_mode = "2.0"
          ansible.limit = "all"
          ansible.playbook = "ansible/playbook.yaml"
          ansible.groups = {
            "aio" => ["#{openstack_release}1"],
          }
          ansible.extra_vars = {
            "openstack_release" => openstack_release,
          }
          ansible.tags = ansible_tags
          ansible.raw_ssh_args = ['-o ServerAliveInterval=30']
        end
      end
    end
  end
end
