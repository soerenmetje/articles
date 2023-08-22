![An expressive oil painting of an captain in front of a steering wheel steering of an old wooden ship](images/DALL%C2%B7E%202023-04-15%2022.08.38%20-%20An%20expressive%20oil%20painting%20of%20a%20cabin%20of%20an%20old%20wooden%20ship.png)

# Debug Kubernetes Applications - A Swiss Army Knife Pod

To debug Kubernetes applications, e.g. if pods can reach a service, an interactive container shell including all essential tools is great - a Swiss army knife so to speak. Unfortunately, in popular base images such as Ubuntu, Debian, CentOS, and Busybox essential debugging tools are not included due to image size reduction and security. While this is great in production environments, it is counterproductive for debugging. Here, we presented an approach based on the extended Ubuntu image [leodotcloud/swiss-army-knife](https://hub.docker.com/r/leodotcloud/swiss-army-knife) created by [leodotcloud](https://github.com/leodotcloud).

## Usage 
### Create Debugging-Pod
Spin up the pod:

``` shell
kubectl create -n mynamespace -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: swiss-army-knife
  labels:
    app: swiss-army-knife
spec:
  containers:
  - name: swiss-army-knife
    image: leodotcloud/swiss-army-knife:latest
    command: ["/bin/sleep", "3650d"]
    imagePullPolicy: IfNotPresent
  restartPolicy: Always
EOF
```


### Get Interactive Shell

Get access to the container command line and debug your stuff:

``` shell
kubectl exec -n mynamespace swiss-army-knife  -it -- /bin/bash
```

As an example the following command checks whether the port `3306` (MySQL) is accessible on `10.152.183.115`:

``` shell
netcat -v -z -w 4 10.152.183.115 3306
```

### Remove Debugging-Pod

After debugging, you can delete the pod:
``` shell
kubectl delete pod -n mynamespace swiss-army-knife
```

## Tools and Packages

Following tools and packages are included. An up-to-date list is available in the [Dockerfile](https://github.com/leodotcloud/swiss-army-knife/blob/main/package/Dockerfile).

- arping
- arptables
- bridge-utils
- ca-certificates
- conntrack
- curl
- dnsutils
- ethtool
- iperf
- iperf3
- iproute2
- ipsec-tools
- ipset
- iptables
- iputils-ping
- jq
- kmod
- ldap-utils
- less
- libpcap-dev
- man
- manpages-posix
- mtr
- net-tools
- netcat
- netcat-openbsd
- openssl
- openssh-client
- psmisc
- socat
- tcpdump
- telnet
- tmux
- traceroute
- tcptraceroute
- tree
- ngrep
- vim
- wget

## Sources
- https://github.com/leodotcloud/swiss-army-knife
- https://hub.docker.com/r/leodotcloud/swiss-army-knife
- https://www.digitalocean.com/community/tutorials/how-to-use-netcat-to-establish-and-test-tcp-and-udp-connections
