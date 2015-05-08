# docker-compose-joc
Docker Compose based CloudBees Jenkins Operations Center  environment.

Uses [Docker Machine](http://docs.docker.com/machine/) and [Docker Compose](https://docs.docker.com/compose/) to build a Jenkins Operations Center by CloudBees environment using Docker containers.

Mostly specific to Mac OS X but should work on Windows and Linux as well.

####Base includes:
- [dnsdock](https://github.com/tonistiigi/dnsdock) providing DNS for docker containers and exposed to Mac OS X 
- JOC by CloudBees (non HA) - availabe at http://joc.demo.docker:8080
  - JNLP port: `4001`
- Client Master api-team (non HA): http://apiteam.demo.docker:8080
- Client Master (HA) mobile-team: http://mobileteam.proxy.docker
  - http://mobileteam1.demo.docker:8080
  - http://mobileteam2.demo.docker:8080
- HA Proxy - stats available at: http://mobileteam.proxy.docker:9000
- 4 slaves with git installed, running ssh on port 22, Jenkins Home - /home/jenkins
- Coming soon: Docker enabled slave

###Instructions
- install VirtualBox 4.3.26 or later
- install [Docker Machine](http://docs.docker.com/machine/#installation)
- install [Docker Compose](https://docs.docker.com/compose/install/)
- create a Docker Machine and add `bip` and `dns` docker daemon default options (I use cjoc as `{machine_name}`)
  - `docker-machine create --driver=virtualbox --virtualbox-memory=4096 {machine_name}`
  - set env for newly created machine: `eval "$(docker-machine env)"`
  - ssh into machine: `docker-machine ssh {machine_name}`
  - `vi /var/lib/boot2docker/profile`
  - append to `EXTRA_ARGS` and save: `-bip=172.17.42.1/24 -dns 172.17.42.1 -dns 8.8.8.8 `
  - update the VirtualBox network adapter (vboxnet<x> - number may vary) *Promiscuous Mode* to *Allow All* 
- replace vboxfs /Users share with nfs
  - Create NFS share on Mac OS X side:
    - create exports file: `sudo vi /etc/exports` with contents (IP used here is your `docker-machine ip {machine_name}`): `/Users 192.168.99.100 -alldirs -mapall={your_username}`
    - restart nfsd: `sudo nfsd restart`
  - Setup Docker Machine nfs:
    - Get IP of you VBox for the Docker host you are updating: `VBoxManage showvminfo {machine_name} --machinereadable | grep hostonlyadapter`
    - Run the following command to get the IPAddress for the VBox Network Adapter that matches the name from above: `VBoxManage list hostonlyifs`
    - Add following script to your Docker Machine at `/var/lib/boot2docker/bootlocal.sh`:
      ```shell
      #/bin/bash
      sudo umount /Users
      sudo /usr/local/etc/init.d/nfs-client start
      sudo mount -t nfs -o noacl,async 192.168.99.1:/Users /Users
      ```
    - Make the `bootlocal.sh` file executable: `sudo chmod +x bootlocal.sh`
    - exit ssh and restart Docker Machine: `docker-machine restart`
- Route traffic from Mac OS X to Docker Machine VM IP: `sudo route -n add -net 172.17.0.0 <MACHINE_IP>`
  - MACHINE_IP retrieved via `docker-machine ip {machine_name}`
- Configure OS X to use dnsdock DNS by creating the file `/etc/resolver/docker` with content of `nameserver 172.17.42.1`
- Clone this repo anywhere under your `/Users` directory
- If you wouldl like to have your Jenkins `HOME` directory somewhere else you need to update the `docker-compose.yml` file:
  - Update `data` under dnsdock -> volumes to point to where you want your Jenkins `HOME` directory. 
  NOTE: You could have several different directories configured for different demos and just change this to point to the demo you want to run.

###Gotchas
If you are no longer able to access docker container hosts via Mac OS X:
- check that the route is correct: `sudo route -n add -net 172.17.0.0 192.168.59.103`
  - Gateway should be `boot2docker ip`
- make sure you are able to ping the `docker-machine ip {machine_name}` - ex (the IP may vary): `ping 192.168.59.103` from Mac OS X
- check to see that the `ip route` you added, still points to your `boot2docker ip` - `sudo route -n get 172.17.42.1`
- You may have to flush DNS cache - on Yosemite use: `sudo discoveryutil mdnsflushcache`
- You may want to verify that you are using nfs instead of vboxfs for the `/Users` mount:
  - `docker-machine ssh {machine_name}`
  - `mount`
  - Look for the `/Users` mount and make sure it is using nfs
