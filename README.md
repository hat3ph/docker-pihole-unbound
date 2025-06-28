## Safe Internet with Pihole and Unbound - Solution
This solution is a combination of Pihole and Unbound in a docker-compose project with the intent of enabling users to quickly and easily create and deploy a personally managed ad blocking capabilities , family safe search, and DNS caching with additional privacy options and DNSSEC validation (via Unbound). 

Docker Compose file contains:
- pihole-unbound - https://github.com/hat3ph/pihole-unbound

Shoutout to: https://github.com/chriscrowe/docker-pihole-unbound

## Prerequisites:
- Run `disable_dnsstublistener.sh` first to disable systemd-resolved DNS stub listener.
- ☁ If using a cloud provider, you need to allow ingress for below port:

| Port      | Service                 |
|-----------|-------------------------|
| 53/tcp    | Pihole DNS connection   |
| 53/udp    | Pihole DNS connection   |
| 80/tcp    | Pihole web panel HTTP   |
| 443/tcp   | Pihole web panel HTTPS  |
| 5335/tcp  | Unbound DNS connection  |
| 5335/udp  | Unbound DNS connection  |

### For using Docker:
- Install docker: https://docs.docker.com/engine/install/
- Install docker-compose: https://docs.docker.com/compose/install/
- Run docker as non-root: https://docs.docker.com/engine/install/linux-postinstall/

### For using Podman
- Install podman and podman-compose
  - Ubuntu 24.04: `sudo apt-get install podman podman-compose`
  - CentOS 9: `sudo yum install epel-release && sudo yum install podman podman-compose`

## Quickstart
To get started all you need to do is git clone the repository and spin up the containers.
```bash
git clone https://github.com/hat3ph/docker-pihole-unbound.git
cd docker-pihole-unbound

# using docker
docker compose up -d

# using podman as root
sudo podman compose up -d

# using rootless podman
podman compose up -d
```

## Podman FAQ
### Run rootful VS rootless podman container
You can run rootful container with root full access or rootless container for enhanced security using podman.
```bash
# enable rootfull system wide podman service
sudo systemctl enable --now podman.socket
sudo podman compose -f docker-compose.yml up -d

# enable rootless podman service
systemctl --user enable --now podman.socket
podman compose -f docker-compose.yml up -d
```
### Not able to bind port 53
```bash
Error: cannot listen on the UDP port: listen udp4 :53: bind: address already in use
exit code: 126
podman start adguard-unbound
Error: unable to start container "2ef2c01bc0ba476095b851a8b71dc24ebff94d4fe681ce66e4b2db78b8589922": cannot listen on the UDP port: listen udp4 :53: bind: address already in use
exit code: 125
```
Not able to bind port 53 due to already used by Podman's [aardvark-dns](https://github.com/containers/podman/discussions/14242). Edit `/etc/containers/containers.conf` to use another port.
```bash
[network]
dns_bind_port=5000 # or well whatever you like
```
### Rootless podman with privileged port 53
```bash
Error: rootlessport cannot expose privileged port 53, you can add 'net.ipv4.ip_unprivileged_port_start=53' to /etc/sysctl.conf (currently 1024), or choose a larger port number (>= 1024): listen tcp 0.0.0.0:53: bind: permission denied
exit code: 126
podman start adguard-unbound
Error: unable to start container "73d392a9d0df2d4f92492c4f8702aba40d7b6909e97492c4da69ab7853afae13": rootlessport cannot expose privileged port 53, you can add 'net.ipv4.ip_unprivileged_port_start=53' to /etc/sysctl.conf (currently 1024), or choose a larger port number (>= 1024): listen tcp 0.0.0.0:53: bind: permission denied
exit code: 125
```
Update sysctl for port 53. Reboot the server with the new changes.
```bash
echo "net.ipv4.ip_unprivileged_port_start=53" | sudo tee /etc/sysctl.d/20-dns-privileged-port.conf
```
### Podman container restart policy
To enable container auto-start during host reboot, enable the podman-restart service.
```bash
# enable system-wide podman service
sudo systemctl enable --now podman-restart.service
# enable rootless podman service
systemctl --user enable --now podman-restart.service
```
Edit `docker-compose.yml` and change restart policy to [always](https://github.com/containers/podman/issues/20418).
```yml
restart: always
```
### Podman mounted volume permission denied with SELinux 
```bash
# /var/log/message
Jun 28 19:17:40 centos9 setroubleshoot[1818]: SELinux is preventing /usr/sbin/unbound from read access on the file unbound.conf. For complete SELinux messages run: sealert -l 6951e854-eb81-4d61-b11f-87dc2fb3db7f
Jun 28 19:17:40 centos9 setroubleshoot[1818]: SELinux is preventing /usr/sbin/unbound from read access on the file unbound.conf.#012#012*****  Plugin catchall (100. confidence) suggests   **************************#012#012If you believe that unbound should be allowed read access on the unbound.conf file by default.#012Then you should report this as a bug.#012You can generate a local policy module to allow this access.#012Do#012allow this access for now by executing:#012# ausearch -c 'unbound' --raw | audit2allow -M my-unbound#012# semodule -X 300 -i my-unbound.pp#012

# podman logs adguard-unbound
[1751109574] unbound[3:0] error: Could not open /opt/unbound/unbound.conf: Permission denied
[1751109574] unbound[3:0] fatal error: Could not read config file: /opt/unbound/unbound.conf. Maybe try unbound -dd, it stays on the commandline to see more errors, or unbound-checkconf
Failed to start unbound: 1
```
Either [disable SELinux](https://linuxconfig.org/how-to-disable-selinux-on-linux) or edit `docker-compose.yml` with [label](https://blog.christophersmart.com/2021/01/31/podman-volumes-and-selinux/)
```yml
    volumes:
      - "./adguard/opt-adguard-work:/opt/adguardhome/work:Z" # adguard container work directory
      - "./adguard/opt-adguard-conf:/opt/adguardhome/conf:Z" # adguard container conf directory
      - "./unbound:/opt/unbound:Z" # map custom unbound config
```

## Customize Pihole
You can customize pihole in docker-compose.yml follow pihole's official docker's [documentation](https://github.com/pi-hole/docker-pi-hole#readme).
```bash
- TZ=Asia/Kuala_Lumpur # set the correct timezone
- FTLCONF_webserver_api_password=password # set your own web panel password
- FTLCONF_dns_listeningMode=all # listen only to interface
- FTLCONF_dns_upstreams=127.0.0.1#5335 # Use unbound as upstream DNS
- TEMPERATUREUNIT=C # set preferred temperature unit
- WEBTHEME=default-dark # web panel theme
#- ADMIN_EMAIL=xxx@gmail.com # set your admin email address
```

## Disable open resolve to prevent DNS Amplication Attack
If you run this in cloud as your provide DNS, advise to restrict DNS access to prevent [DNS Amplication Attack](https://openresolver.com/).
Setup cron job to run `iptables_ddns_update.sh` to update the iptables rule.
Docker will re-create the docker iptables rule if you restart the container hence will mess up with the iptables rule. 
Advice just restart the VPS to let the script setup the iptables rule again from fresh.
