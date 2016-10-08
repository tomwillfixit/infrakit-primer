# InfraKit primer

InfraKit was announced at this weeks LinuxCon in Berlin.  The announcement blog can be found [here](https://blog.docker.com/2016/10/introducing-infrakit-an-open-source-toolkit-for-declarative-infrastructure).

The following primer will use InfraKit plugins to create and maintain a simple vagrant VM based infrastructure.  You will need virtualbox and vagrant installed.

As more plugins are created by the community, the potential of InfraKit will become apparent.  Combined with Swarm mode in Docker 1.12, InfraKit will empower developers to build and distribute applications using the Docker CLI.

## Setup

Step 1 :
```
export PATH=/usr/local/go/bin:$PATH
mkdir -p ~/go
export GOPATH=~/go
export PATH=$GOPATH/bin:$PATH
```

Step 2 :
```
mkdir -p ~/go/src/github.com/docker
cd ~/go/src/github.com/docker
git clone https://github.com/docker/infrakit.git
cd infrakit/
go get -u github.com/kardianos/govendor
go get -u github.com/golang/lint/golint
```

Step 3 :

Build the plugin binaries.
```
make -k infrakit
```

Step 4 :

Create the default location for plugins.
```
mkdir -p /run/infrakit/plugins/
chmod 777 /run/infrakit/plugins/
```

## Start your engines ... plugins

We will start 4 plugins; a default "Group" plugin, 2 "Instance" plugins (file and vagrant) and a vanilla "Flavor" Plugin

Open 4 terminals/tabs and start a plugin in each :

```
cd ~/go/src/github.com/docker/infrakit

infrakit/group --log 5
infrakit/vanilla --log 5
infrakit/file --log 5 --dir ./tutorial/
infrakit/vagrant --log 5
```

You will see output like :

```
INFO[0000] Starting discovery                           
DEBU[0000] Opening: /run/infrakit/plugins               
DEBU[0000] Discovered plugin at unix:///run/infrakit/plugins/flavor-vanilla.sock 
DEBU[0000] Discovered plugin at unix:///run/infrakit/plugins/instance-file.sock 
DEBU[0000] Discovered plugin at unix:///run/infrakit/plugins/instance-vagrant.sock 
INFO[0000] Starting plugin                              
INFO[0000] Starting                                     
INFO[0000] Listening on: unix:///run/infrakit/plugins/group.sock 
INFO[0000] listener protocol= unix addr= /run/infrakit/plugins/group.sock err= <nil> 
```

Open another terminal/tab and confirm the plugins are listed correctly :

```
infrakit/cli plugin ls

NAME                	LISTEN

instance-file       	unix:///run/infrakit/plugins/instance-file.sock
instance-vagrant    	unix:///run/infrakit/plugins/instance-vagrant.sock
flavor-vanilla      	unix:///run/infrakit/plugins/flavor-vanilla.sock
group               	unix:///run/infrakit/plugins/group.sock
```

## Create a group of Vagrant instances

First we need to grab a Centos7 box :

```
vagrant box add centos7 https://github.com/vezzoni/vagrant-vboxes/releases/download/0.0.1/centos-7-x86_64.box
```

This is the configuration JSON for a group of Vagrant Centos7 VMs :

```
{
    "ID": "vagrant-centos7-vm",
    "Properties": {
        "Instance": {
            "Plugin": "instance-vagrant",
            "Properties": {
                "Note": "Instance properties for vagrant vm (v0.1)",
                "Box": "centos7",
		        "Memory": 1024,
		        "CPUs": 1
            }
        },
        "Flavor": {
            "Plugin": "flavor-vanilla",
            "Properties": {
                "Size": 3,
                "UserData": [
                    "curl https://experimental.docker.com |sudo bash",
		            "sudo service docker start"
                ],

                "Labels": {
                    "tier": "docker-engines",
                    "project": "infrakit"
                }
            }
        }
    }
}
```


### Start a group of VMs and watch their status

The group plugin is responsible for ensuring that the infrastructure state matches with your specifications. Since we started out with nothing, it will create 3 vagrant vm instances and maintain that state by monitoring the instances.

```
infrakit/cli group --name group watch vagrant-centos7-vm.json
watching vagrant-centos7-vm
```

It will take a minute or 2 for the VMs to start and to be provisioned.  If you look in the terminal running the group plugin you will see output similar to :

```
DEBU[0026] REQ -- unix://127.0.0.1/run/infrakit/plugins/instance-vagrant.sock url= http://127.0.0.1/Instance.Validate request= {"Note":"Instance properties for vagrant vm (v0.1)","Box":"centos7","Memory":1024,"CPUs":1} err= <nil> 
DEBU[0026] RESP - unix://127.0.0.1/run/infrakit/plugins/instance-vagrant.sock url= http://127.0.0.1/Instance.Validate response=  err= <nil> 
INFO[0026] Watching group 'vagrant-centos7-vm'          
DEBU[0036] REQ -- unix://127.0.0.1/run/infrakit/plugins/instance-vagrant.sock url= http://127.0.0.1/Instance.DescribeInstances request= {"infrakit.group":"vagrant-centos7-vm"} err= <nil> 
DEBU[0036] RESP - unix://127.0.0.1/run/infrakit/plugins/instance-vagrant.sock url= http://127.0.0.1/Instance.DescribeInstances response= [] err= <nil> 
DEBU[0036] Found existing instances: []                 
INFO[0036] Adding 3 instances to group to reach desired 3 
DEBU[0036] REQ -- unix://127.0.0.1/run/infrakit/plugins/flavor-vanilla.sock url= http://127.0.0.1/Flavor.PreProvision request= {"Properties":{"Size":3,"UserData":["curl https://experimental.docker.com |sudo bash","sudo service docker start"],"Labels":{"tier":"docker-engines","project":"infrakit"}},"InstanceSpec":{"Properties":{"Note":"Instance properties for vagrant vm (v0.1)","Box":"centos7","Memory":1024,"CPUs":1},"Tags":{"infrakit.config_sha":"yk7pDwf0Zw0mmgtPOCXoErR6rBU=","infrakit.group":"vagrant-centos7-vm"},"Init":"","LogicalID":null,"Attachments":null}} err= <nil> 
DEBU[0036] RESP - unix://127.0.0.1/run/infrakit/plugins/flavor-vanilla.sock url= http://127.0.0.1/Flavor.PreProvision response= {"Properties":{"Note":"Instance properties for vagrant vm (v0.1)","Box":"centos7","Memory":1024,"CPUs":1},"Tags":{"infrakit.config_sha":"yk7pDwf0Zw0mmgtPOCXoErR6rBU=","infrakit.group":"vagrant-centos7-vm","project":"infrakit","tier":"docker-engines"},"Init":"\ncurl https://experimental.docker.com |sudo bash\nsudo service docker start","LogicalID":null,"Attachments":null} err= <nil> 


DEBU[1412] Group has 3 instances, no action is needed 
```

This is verification that the VMs have been started and docker is being installed.  The logs from the vm setup can be found in the vagrant terminal/tab.

Check vagrant status :

```
vagrant global-status
id       name    provider   state   directory                                                  
-----------------------------------------------------------------------------------------------
98b838e  default virtualbox running /root/go/src/github.com/docker/infrakit/infrakit-326889880 
d6c529f  default virtualbox running /root/go/src/github.com/docker/infrakit/infrakit-869582871 
d90f43f  default virtualbox running /root/go/src/github.com/docker/infrakit/infrakit-098046429
```

Verify docker was installed :

```
vagrant ssh -c "sudo docker info" d90f43f

Containers: 0
 Running: 0
 Paused: 0
 Stopped: 0
Images: 0
Server Version: 1.12.2-rc3
```

You can find the Vagrantfile, boot.sh and tags for each vm here :

```
~/go/src/github.com/docker/infrakit# cat infrakit-760192827/Vagrantfile 

Vagrant.configure("2") do |config|
  config.vm.box = "centos7"
  config.vm.hostname = "infrakit.box"
  config.vm.network "private_network", type: "dhcp"

  config.vm.provision :shell, path: "boot.sh"

  config.vm.provider :virtualbox do |vb|
    vb.memory = 1024
    vb.cpus = 1
  end
end

~/go/src/github.com/docker/infrakit# cat infrakit-760192827/boot.sh 

curl https://experimental.docker.com |sudo bash
sudo service docker start

~/go/src/github.com/docker/infrakit# cat infrakit-760192827/tags
{"infrakit.config_sha":"yk7pDwf0Zw0mmgtPOCXoErR6rBU=","infrakit.group":"vagrant-centos7-vm","project":"infrakit","tier":"docker-engines"}
```

### Ok so what's the point? docker-machine can do this.

Let's see how the plugins manage the state of the infrastruture as the configuration changes.

Add another VM. Edit vagrant-centos7-vm.json and update the Size field to 4.

Update the group :

```
infrakit/cli group --name group update vagrant-centos7-vm.json
```

Now check the logs and you'll see :

```
INFO[1602] Adding 1 instances to group to reach desired 4
```

That's cool.

Let's try removing a VM.

```
vagrant destroy -f d90f43f
```

Ok nothing happened.  When you remove the directory containing the vm config then the group plugin detects the vm has been removed :

```
rm -rf infrakit-760192827
```

Head over to the logs and you'll see something like :

```
INFO[1992] Adding 1 instances to group to reach desired 4
```

The group plugin is ensuring the infrastructure state matches the configuration.  That's really cool.

The flavor plugin can check the health of the infrastructure configuration :

```
# infrakit/cli flavor --name flavor-vanilla healthy vagrant-centos7-vm.json 

Output : true

```

## Summary

That's all for now.  InfraKit was easy to pick up and use, as with all other tools in the Docker ecosystem.  When InfraKit is combined/built-in to the docker CLI it will empower developers to deploy their distributed applications on cloud agnostic, self healing, robust infrastructure.



