![Oil painting of wale breaking through a firewall](images/DALL%C2%B7E%202023-02-23%2023.59.59%20-%20add%20wall.png)


# How to Secure a Docker Host Using Firewalld

If you are using a firewall like ufw or firewalld and docker you may encounter the problem that docker bypasses the firewall rules.

### Goal
The firewall rules should count for whole host system - so including docker containers with port mappings. 
The host ports in container port mappings can be allowed in firewall and therefore be exposed to the internet.
Also, the approach should not break container networking.

### Existing Approaches 
I found following approaches that try to fix the problem. However, each approach introduced another problem:

- Just do not use docker. Podman for example obeys firewall rules by default. Problem: Some can not or may not want to switch to a different container runtime, but I generally recommend checking if this is an option for you. [This article](https://phoenixnap.com/kb/podman-vs-docker) includes a comparison. 
- Use external firewall like security groups in Openstack instead of ufw or firewalld. Problem: Not available in my case.
- Just do not map ports in docker. Problem: may introduce security risk because of no single source of truth for exposed host ports.
- Disabling iptables for docker. Problem: Containers can not access internet.
- Configuring the firewall to ignore port mappings. Problem: The port inside the container have to be allowed in the host firewall. If multiple containers use same port and only one should be allowed we have to additionally specify container IP or service name. In summary: Counterintuitive, complex and error-prone.

## Final Solution
Idea: Disable iptables for docker and configure firewalld to allow container networking. It is based on this [Medium article](https://erfansahaf.medium.com/why-docker-and-firewall-dont-get-along-with-each-other-ddca7a002e10) by Erfan Sahafnejad and several posts. I tested it with Ubuntu 22.04.2 LTS but the concept should also work on other Linux.

### Preparation

If you added any configuration to iptables regarding docker before, remove it first.

If ufw is installed and active, disable it:

``` bash
ufw disable
```

Install and activate firewalld:

``` bash
apt update && apt install firewalld -y
systemctl enable --now firewalld

# Confirm that the service is running
firewall-cmd --state
```

> Compared to ufw, firewalld is more powerful - it provides features that we need for upcoming firewall configurations. However, it also just a convenient frontend for iptables. Learn more about firewalld here: https://docs.rockylinux.org/guides/security/firewalld-beginners/

### Disable iptables for Docker
Disable iptables for docker in `/etc/docker/daemon.json` so it should look like follows: 

``` json
{
"iptables": false
}
```

If `/etc/docker/daemon.json` does not exist, create the file first.

Restart docker:
``` bash
systemctl restart docker
```

Already at this point, only container ports that are allowed in firewall should be reachable from the internet. However, as a side effect of disabling iptables in docker, we broke container internet access: From the inside of containers we can not access the internet anymore.

``` bash
docker run --rm busybox ping -c4 8.8.8.8
# PING 8.8.8.8 (8.8.8.8): 56 data bytes
#
# --- 8.8.8.8 ping statistics ---
# 4 packets transmitted, 0 packets received, 100% packet loss
```

### Configure firewalld
At next, we configure firewalld to enable docker container networking.

Add Masquerading to the zone which leads out to the Internet, typically `public`: 

``` bash
# Masquerading allows for docker ingress and egress (this is the juicy bit)
firewall-cmd --zone=public --add-masquerade --permanent
# Reload firewall to apply permanent rules
firewall-cmd --reload
```

Sources: https://serverfault.com/a/987687 and https://serverfault.com/a/1046550


Additionally, in order to enable docker containers accessing host ports, add docker interface to the `trusted` zone: 

``` bash
# Show interfaces to find out docker interface name
ip link show

# Assumes docker interface is docker0
firewall-cmd --permanent --zone=trusted --add-interface=docker0
firewall-cmd --reload
systemctl restart docker
```

Sources: https://unix.stackexchange.com/a/225845 and https://unix.stackexchange.com/a/333356


So far, docker containers that are not attached to a docker network can access the internet. But containers that are attached can still not. This is often the case when using docker compose.

If we try to ping Google DNS server this is the result:

``` bash
docker run --rm busybox ping -c4 8.8.8.8
# PING 8.8.8.8 (8.8.8.8): 56 data bytes
# 64 bytes from 8.8.8.8: seq=0 ttl=58 time=3.699 ms
# 64 bytes from 8.8.8.8: seq=1 ttl=58 time=3.588 ms
# 64 bytes from 8.8.8.8: seq=2 ttl=58 time=3.587 ms
# 64 bytes from 8.8.8.8: seq=3 ttl=58 time=3.518 ms
#
# --- 8.8.8.8 ping statistics ---
# 4 packets transmitted, 4 packets received, 0% packet loss
# round-trip min/avg/max = 3.518/3.598/3.699 ms


# Create docker network for testing purpose (can be deleted later)
docker network create --driver bridge mynet

docker run --rm --net mynet busybox ping -c4 8.8.8.8
# PING 8.8.8.8 (8.8.8.8): 56 data bytes
#
# --- 8.8.8.8 ping statistics ---
# 4 packets transmitted, 0 packets received, 100% packet loss
```

To fix this, add your network interface to `public` zone: 

``` bash
# Show public ip
curl -4 ip.gwdg.de
# Show interfaces to find out network interface name with your public IP
ip addr

# Assumes network interface with your public IP is eth0
# (ens18 is also a name I came accross)
firewall-cmd --permanent --zone=public --add-interface=eth0
firewall-cmd --reload
```

Networking should work properly now and therefore containers should be able to access the internet.

If we now try to ping Google DNS server again, it works as expected:
``` bash
docker run --rm busybox ping -c4 8.8.8.8
# PING 8.8.8.8 (8.8.8.8): 56 data bytes
# 64 bytes from 8.8.8.8: seq=0 ttl=58 time=3.641 ms
# 64 bytes from 8.8.8.8: seq=1 ttl=58 time=3.565 ms
# 64 bytes from 8.8.8.8: seq=2 ttl=58 time=3.605 ms
# 64 bytes from 8.8.8.8: seq=3 ttl=58 time=3.546 ms
#
# --- 8.8.8.8 ping statistics ---
# 4 packets transmitted, 4 packets received, 0% packet loss
# round-trip min/avg/max = 3.546/3.589/3.641 ms

docker run --rm --net mynet busybox ping -c4 8.8.8.8
# PING 8.8.8.8 (8.8.8.8): 56 data bytes
# 64 bytes from 8.8.8.8: seq=0 ttl=58 time=3.671 ms
# 64 bytes from 8.8.8.8: seq=1 ttl=58 time=3.644 ms
# 64 bytes from 8.8.8.8: seq=2 ttl=58 time=3.561 ms
# 64 bytes from 8.8.8.8: seq=3 ttl=58 time=3.508 ms
#
# --- 8.8.8.8 ping statistics ---
# 4 packets transmitted, 4 packets received, 0% packet loss
# round-trip min/avg/max = 3.508/3.596/3.671 ms
```

### Open Firewall Ports 
In the end, open the desired ports for your service to allow incoming traffic, e.g. on port 8080: 

``` bash
firewall-cmd --permanent --zone=public --add-port=8080/tcp
# Reload firewall to apply permanent rules
firewall-cmd --reload
```

## Extra: Test Networking
Run checks to ensure the firewall is working properly. Following things should hold true:

- Container running on allowed port can be accessed from internet
- Container running on **not** allowed port can **not** be accessed from internet
- Container can access internet
- Container with new docker network can access internet
- Container can access service running on host system port
- Container can access other container inside same docker network


To start a webserver on 8080 you can run:

``` bash
docker run --restart unless-stopped -p 8080:80 -d nginx
```

To curl a service running on host system port from inside of a container you can run:

``` bash
# Show docker interface ip address
ip addr
# ...
# 3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
#    link/ether 02:42:37:6b:6e:2a brd ff:ff:ff:ff:ff:ff
#    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
#       valid_lft forever preferred_lft forever
# ...

# Assumes docker interface ip is 172.17.0.1
# and service runs on port 80
docker run --rm curlimages/curl -v http://172.17.0.1:80/
```


## Further Reading

- Good firewalld guide: https://docs.rockylinux.org/guides/security/firewalld-beginners/


## Sources
- https://erfansahaf.medium.com/why-docker-and-firewall-dont-get-along-with-each-other-ddca7a002e10
- https://github.com/docker/for-linux/issues/955#issuecomment-621141128
- https://docs.docker.com/network/iptables/#prevent-docker-from-manipulating-iptables
- https://serverfault.com/a/987687
- https://serverfault.com/a/1046550
- https://unix.stackexchange.com/a/225845