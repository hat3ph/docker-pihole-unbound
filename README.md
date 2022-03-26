## Safe Internet with Pihole and Unbound - Solution
This solution is a combination of Pihole and Unbound in a docker-compose project with the intent of enabling users to quickly and easily create and deploy a personally managed ad blocking capabilities , family safe search, and DNS caching with additional privacy options and DNSSEC validation (via Unbound). 

Docker Compose file contains:
- pihole-unbound - https://hub.docker.com/r/cbcrowe/pihole-unbound

Shoutout to: https://github.com/chriscrowe/docker-pihole-unbound

## Prerequisites:
- Install docker: https://docs.docker.com/engine/install/
- Install docker-compose: https://docs.docker.com/compose/install/
- Run docker as non-root: https://docs.docker.com/engine/install/linux-postinstall/
- Run `disable_dnsstublistener.sh` first to disable systemd-resolved DNS stub listener.
- ☁ If using a cloud provider, you need to allow ingress for below port:

| Port      | Service                 |
|-----------|-------------------------|
| 53/tcp    | Pihole DNS connection   |
| 53/udp    | Pihole DNS connection   |
| 80/tcp    | Pihole web panel HTTP   |
| 443/tcp   | Pihole web panel HTTPS  |
| 5353/tcp  | Unbound DNS connection  |
| 5353/udp  | Unbound DNS connection  |

## Quickstart
To get started all you need to do is git clone the repository and spin up the containers.
```bash
git clone https://github.com/hat3ph/docker-pihole-unbound.git
cd docker-adguard-unbound
docker-compose up -d
```

## Customize Pihole
You can customize pihole in docker-compose.yml follow pihole's official docker's [documentation](https://github.com/pi-hole/docker-pi-hole#readme).
```bash
- TZ=Asia/Kuala_Lumpur # set the correct timezone
- WEBPASSWORD=password # set your own web panel password
- DNSMASQ_LISTENING=single # listen only to interface
- PIHOLE_DNS_=127.0.0.1#5335 # Use unbound as upstream DNS
- TEMPERATUREUNIT=C # set preferred temperature unit
- WEBTHEME=default-dark # web panel theme
#- ADMIN_EMAIL=xxx@gmail.com # set your admin email address
```

## Disable open resolve to prevent DNS Amplication Attack
If you run this in cloud as your provide DNS, advise to restrict DNS access to prevent [DNS Amplication Attack](https://openresolver.com/).
Setup cron job to run `iptables_ddns_update.sh` to update the iptables rule.
Docker will re-create the docker iptables rule if you restart the container hence will mess up with the iptables rule. 
Advice just restart the VPS to let the script setup the iptables rule again from fresh.
