# OCP node debugging

With a recently encountered issue, we want to share some debugging for OCP nodes which might not be obvious to everyone, and or, if you have suggestions ideas to enhance, they are more than welcome.

## ocp debug node/<node>
the most often referenced `oc debug node` command is quite limited to functionality. We want to show how this can be taken to the next Level in particular for network tracing functionality.

## toolbox
did you know that there's a `toolbox` command available in an ocp debug session ? 
The toolbox will provide some functionality which is not included in CoreOS and further more, you can run your own, customized image instead of the provided one.

### running toolbox 
to get the benefit of `toolbox` and `functionality` included, you need to first run your debug session
```
oc debug node/node1
Starting pod/node1 ...
To use host binaries, run `chroot /host`
Pod IP: 192.168.0.1
If you don't see a command prompt, try pressing enter.

sh-4.4# chroot /host
```
this brings you in the context of the Node including the file system view as you are used to physical Hosts or VMs.

Now, we have the possibilty to execute the `toolbox` command, which will spawn a `privileged` podman container in the `host` context (file system, processes, ...)

```
sh-4.4# toolbox
Trying to pull registry.redhat.io/rhel8/support-tools:latest...
Getting image source signatures
Checking if image destination supports signatures
Copying blob 995adc376540 done  
Copying blob 1ff2f4febecd done  
Copying config 3fa498ae7c done  
Writing manifest to image destination
Storing signatures
3fa498ae7cae5a9359a2d186fa74921f92c2d73220499dc409be832b48d67c05
Spawning a container 'toolbox-root' with image 'registry.redhat.io/rhel8/support-tools'
Detected RUN label in the container image. Using that as the default...
c3df4948fed87e6bb051579d6a1d978f953c92b90eca745a35dab9ee2b0d4bb0
toolbox-root
Container started successfully. To exit, type 'exit'.
```
within that image you have for example, the possibility to execute `tcpdump` which isn't available in CoreOS.

### customizing toolbox 

now, let's say we want to run tools on the OCP node, we are familiar and comfortable with.
Just create your own image with those tools included and push it to a registry access able (*disconnected deployments*)

```
# Example tool chain
FROM registry.access.redhat.com/ubi9/ubi
RUN dnf install -y epel-release ; \
    dnf install -y wireshark-cli man 389-ds-base tcpdump strace bpftool \
                   telnet traceroute openldap-clients nmap-ncat nmap bwm-ng \
                   net-tools bind-utils blktrace bridge-utils conntrack-tools \
                   ipcalc lvm2 cryptsetup ncdu ethtool ; \
    dnf clean all ; \
    curl -o/usr/bin/kind -L -s https://github.com/kubernetes-sigs/kind/releases/download/v0.17.0/kind-linux-amd64 ; \
    chmod +x /usr/bin/kind ; \
    curl -L -o- https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-linux.tar.gz | tar -xzf- -C /usr/bin oc kubectl ; \
    chmod +x /usr/bin/{oc,kubectl} ; \
    curl -L -o- https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv5.0.0/kustomize_v5.0.0_linux_amd64.tar.gz | tar -xzf- -C /usr/bin/ kustomize ; \
    chmod +x /usr/bin/kustomize ; \
    rm -fR /var/cache/yum /root/.cache

CMD ["/usr/bin/bash"] 
```

after building and pushing your custom image to your preferred registry, open up your debug session once more
```
oc debug node/node1
Starting pod/node1 ...
To use host binaries, run `chroot /host`
Pod IP: 192.168.0.1
If you don't see a command prompt, try pressing enter.

sh-4.4# chroot /host
sh-4.4# cd ~
```

add following content to `overwrite` the default values for the toolbox configuration at `~/.toolboxrc`

```
REGISTRY=quay.io
IMAGE=<your_quay_login>/<repository>/<image>:<tag>
TOOLBOX_NAME=toolbox
```
now call `toolbox` again 
(in case you do not have the registry credentials in your pull secret you will be asked to authenticate)

```
sh-4.4# toolbox 
.toolboxrc file detected, overriding defaults...
Checking if there is a newer version of quay.io/rhn_support_milang/ocp-debug/node available...
There is a newer version of quay.chester.at/milang/ocp-debug/node available. Would you like to pull it? [y/N] y
Trying to pull quay.io/rhn_support_milang/ocp-debug/node:latest...
Getting image source signatures
Copying blob e5623e09afda done  
Copying blob f17e2fd88520 skipped: already exists  
Copying config 89e0d85ce6 done  
Writing manifest to image destination
Storing signatures
89e0d85ce6b957e11dab4d3873ec290512ac571d8321febeaa6f294f92565b05
8b508cf216a928db583422cf3e9cc5a6e1f3d3bd70dc60c1608886efc5cbab96
Spawning a container 'toolbox' with image 'quay.chester.at/milang/ocp-debug/node'
b3ee1ae8d6fe50b049e5662b707768c2f8158da9b66c76a611210249d1d75e29
toolbox
Container started successfully. To exit, type 'exit'.
```

## extended debugging with a customize toolbox image

Let's assume, we are looking for some `http` traffic, call our debug `toolbox` image and execute `tshark` (wireshark cli) for raw http traffic passing

```
tshark -i any \
	   -O http \
	   -Y 'http.request or http.response' \
	   -T fields \
	   -e http.request.version \
	   -e http.request.method \
	   -e http.user_agent \
	   -e http.request.uri \
	   -e ip.src \
	   -e tcp.dstport \
	   -e http.response.code
	   
Running as user "root" and group "root". This could be dangerous.
Capturing on 'any'
HTTP/1.1	GET	kube-probe/1.25	/healthz	10.128.0.2	8000	
HTTP/1.1	GET	kube-probe/1.25	/readyz	10.128.0.2	8000	
				10.128.0.4	58030	200
				10.128.0.4	58028	200
HTTP/1.1	POST		/api/v2/spans	10.130.0.193	9411	
				10.128.0.19	53424	202
HTTP/1.1	POST		/api/v2/spans	10.130.0.193	9411	
				10.128.0.19	53434	202
				10.128.0.19	53434	202
HTTP/1.1	GET	Go-http-client/1.1	/health	127.0.0.1	18080	
				127.0.0.1	44922	200
HTTP/1.1	GET	kube-probe/1.25	/healthz	10.128.0.2	8082	
				10.128.0.30	51082	200
HTTP/1.1	GET	curl/7.61.1	/healthz/ready		1936	
HTTP/1.1	GET	router-probe/.	/_______internal_router_healthz	127.0.0.1	80	
				127.0.0.1	35254	200
					53114	200
HTTP/1.1	GET	kube-probe/1.25	/ready	10.128.0.2	8080	
				10.128.0.111	55400	200
HTTP/1.1	GET	kube-probe/1.25	/healthz/ready	10.128.0.2	15021	
				10.128.0.60	42202	200
^CHTTP/1.1	GET	kube-probe/1.25	/api/health	10.128.0.2	3000	
HTTP/1.1	GET	kube-probe/1.25	/api/health	10.128.0.2	3000	
				10.128.0.24	49272	200
				10.128.0.24	49286	200

```

as we can see for example, there's a `haproxy` probe on`/_______internal_router_healthz` 

Verifying this on our `toolbox` container which has access to the `host` proc and sys file systems.

```
[root@toolbox /]# ss -ntlp 'sport = :80'
State                      Recv-Q                     Send-Q                                         Local Address:Port                                         Peer Address:Port                    Process                    
LISTEN                     0                          2048                                                 0.0.0.0:80                                                0.0.0.0:*                        users:(("haproxy",pid=3290244,fd=6))


[root@toolbox /]# curl http://localhost/_______internal_router_healthz
<html><body><h1>200 OK</h1>
Service ready.
</body></html>

```

or ever questioned how to identify the top talkers to the kube-api service ?

```
# capturing for 60 seconds 
# tshark -qQ -a duration:60 -i any -f 'ip dst <api-ip-address> and tcp dst port 443' -z endpoints,tcp,tcp.port==443
tshark -qQ -a duration:60 -i any -f 'ip dst 172.30.0.1 and tcp dst port 443' -z endpoints,tcp,tcp.port==443
Running as user "root" and group "root". This could be dangerous.
================================================================================
TCP Endpoints
Filter:tcp.port==6443
                       |  Port  ||  Packets  | |  Bytes  | | Tx Packets | | Tx Bytes | | Rx Packets | | Rx Bytes |
172.30.0.1                  443       4998       1162485          0               0        4998         1162485   
10.129.0.91               33632        654        398160        654          398160           0               0   
10.129.1.95               56766        427         91715        427           91715           0               0   
10.129.0.133              49324        414         86353        414           86353           0               0   
10.129.0.29               41078        311         26928        311           26928           0               0   
10.129.0.52               44442        305         66644        305           66644           0               0   
10.129.0.11               53436        298         46277        298           46277           0               0   
10.129.0.61               48048        201         44913        201           44913           0               0   
10.129.0.55               47782        180         35665        180           35665           0               0   
10.129.0.40               60434        158         20071        158           20071           0               0   
10.129.0.62               45792        131         32088        131           32088           0               0   
10.129.0.90               36516        117         17890        117           17890           0               0   
10.129.0.63               51410        113         12716        113           12716           0               0   
10.129.0.132              45376        111         20348        111           20348           0               0   
10.129.0.32               50838        104         12346        104           12346           0               0   
10.129.0.58               45278        103         24397        103           24397           0               0   
10.129.0.52               41020         89         12829         89           12829           0               0   
10.129.0.17               53536         74          6931         74            6931           0               0   
10.129.0.41               53068         54          4573         54            4573           0               0   
10.129.0.51               58460         49          9269         49            9269           0               0   
10.129.0.29               59472         45          6047         45            6047           0               0   
[.. output omitted ..]

# or try with resolution
echo "nameserver $(oc -n openshift-dns get svc dns-default -o jsonpath='{.spec.clusterIP}')" > /etc/resolv.conf
tshark -NnN -qQ -a duration:60 -i any -f 'ip dst 172.30.0.1 and tcp dst port 443' -z endpoints,tcp,tcp.port==443
Running as user "root" and group "root". This could be dangerous.
================================================================================
TCP Endpoints
Filter:tcp.port==443
                       |  Port  ||  Packets  | |  Bytes  | | Tx Packets | | Tx Bytes | | Rx Packets | | Rx Bytes |
kubernetes.default.svc.cluster.local        443        947        237383          0               0         947          237383   
10.129.1.95               45704        236         87407        236           87407           0               0   
10.129.0.91               52096         77         38463         77           38463           0               0   
10-129-0-29.kiali.istio-system.svc.cluster.local      49712         55          4570         55            4570           0               0   
10-129-0-11.istiod-basic.istio-system.svc.cluster.local      58098         53          8234         53            8234           0               0   
10-129-0-133.api.openshift-apiserver.svc.cluster.local      35280         51         12304         51           12304           0               0   
[.. output omitted ..]
```

or ever wanted to see that particular nodes IP traffic ? 

```
[root@toolbox /]# bwm-ng -o plain -c 1 
bwm-ng v0.6.3 (delay 0.500s); input: /proc/net/dev          iface                    Rx                   Tx               Total
==============================================================================
             lo:         535.31 KB/s          535.31 KB/s            1.05 MB/s
           ens3:          29.38 MB/s            4.50 MB/s           33.88 MB/s
[.. output omitted ..]

ca80805c0ce1ec1:           0.00  B/s            0.00  B/s            0.00  B/s
b9bcd00c0b11406:         131.47  B/s            1.83 KB/s            1.96 KB/s
d57a770e03504dc:           0.00  B/s            0.00  B/s            0.00  B/s
35b3224214f4453:           0.00  B/s            0.00  B/s            0.00  B/s
06c18db4ab10857:           0.00  B/s            0.00  B/s            0.00  B/s
------------------------------------------------------------------------------
          total:          59.97 MB/s           10.57 MB/s           70.54 MB/s
```

or ever wanted to trace your `haproxy` ingress process ?

```
[root@toolbox /]# strace -fs200 -p $(ps -ef  | awk ' /\/usr\/sbin\/haproxy/ { print $2 } ' | tail -1) 2>&1 | head -50
strace: Process 3290244 attached with 4 threads
[pid 3290247] clock_gettime(CLOCK_THREAD_CPUTIME_ID,  <unfinished ...>
[.. output omitted ..]

[pid 3290246] <... accept4 resumed>{sa_family=AF_INET, sin_port=htons(36006), sin_addr=inet_addr("127.0.0.1")}, [128 => 16], SOCK_NONBLOCK) = 20
[pid 3290245] accept4(6,  <unfinished ...>
[pid 3290244] clock_gettime(CLOCK_THREAD_CPUTIME_ID,  <unfinished ...>
[pid 3290247] <... accept4 resumed>0x7ff94bff49e0, [128], SOCK_NONBLOCK) = -1 EAGAIN (Resource temporarily unavailable)
[pid 3290246] write(16, "c", 1 <unfinished ...>
[pid 3290245] <... accept4 resumed>0x7ff952cae9e0, [128], SOCK_NONBLOCK) = -1 EAGAIN (Resource temporarily unavailable)
[pid 3290244] <... clock_gettime resumed>{tv_sec=3, tv_nsec=118961730}) = 0

[.. output omitted ..]
```

## openshift-monitoring prometheus data export

taking the official guide on `How to collect cluster Prometheus metrics` and extend it with the idea of the toolbox usage.

* utilizing `promdump` (https://github.com/ihcsim/promdump)
* utilizing `mcli` (https://min.io/docs/minio/linux/reference/minio-mc.html)
* utilizing the toolbox to collect the data

We could still utilize the same procedure as documented in our KCS (tar cvzf - -C /prometheus ... ) but we want to bring it to the next level instead.

First, let's identify which node our prometheus-k8s-0 pod is running on

```
oc -n openshift-monitoring get pod \
   -l statefulset.kubernetes.io/pod-name=prometheus-k8s-0 \
   -o wide
prometheus-k8s-0   6/6     Running   12         11d   10.128.0.37   node1   <none>           <none>
```
 
create the debug node session to node1 as usual

```
oc debug node/node1
Starting pod/node1-debug ...
To use host binaries, run `chroot /host`
Pod IP: 192.168.0.1
If you don't see a command prompt, try pressing enter.

sh-4.4# chroot /host
sh-4.4# toolbox
.toolboxrc file detected, overriding defaults...
Checking if there is a newer version of quay.io/rhn_support_milang/ocp-debug/node available...
Container 'toolbox' already exists. Trying to start...
(To remove the container and start with a fresh toolbox, run: sudo podman rm 'toolbox')
toolbox
Container started successfully. To exit, type 'exit'.
[root@toolbox /]# 
```

if we are utilizing the `official` toolbox image we need to fetch our additional tools accordingly

```
curl -L -o /usr/bin/mcli https://dl.min.io/client/mc/release/linux-amd64/mc 
chmod +x /usr/bin/mcli

curl -L -o /usr/bin/promdump https://github.com/ihcsim/promdump/releases/download/v0.2.5/promdump
chmod +x /usr/bin/promdump
```

now, we need to identify the location of our data (default setup utilizies `emtpydir`) 

```
oc login -u milang https://api.node.example.com:6443 --insecure-skip-tls-verify=true


PROMETHEUS=$(oc -n openshift-monitoring get pod \
                -l statefulset.kubernetes.io/pod-name=prometheus-k8s-0 \
                -o name)

# check on the path existence

ls /host/var/lib/kubelet/pods/$(oc -n openshift-monitoring get ${PROMETHEUS} -o jsonpath='{.metadata.uid}')/volumes/kubernetes.io~empty-dir/prometheus-k8s-db/

# if there's data return, we can go ahead
# configure your S3 bucket to store the prometheus data

mcli --insecure alias set s3 https://s3.example.com demo-example-com demo-example-com

promdump --data-dir /host/var/lib/kubelet/pods/$(oc -n openshift-monitoring get ${PROMETHEUS} -o jsonpath='{.metadata.uid}')/volumes/kubernetes.io~empty-dir/prometheus-k8s-db | \
    mcli --insecure -q pipe s3/backup/prometheus/node1.dump.tar.gz
[.. output omitted ..]

mcli --insecure stat s3/backup/prometheus/node1.dump
Name      : node1.dump
Date      : 2023-04-28 09:54:27 UTC 
Size      : 418 MiB 
ETag      : 3993a8852a9f39a5f65164d82e9ea854-1 
Type      : file 
Metadata  :
  Content-Type: application/octet-stream 

mcli --insecure cat s3/backup/prometheus/node1.dump.tar.gz | tar -tzf -
chunks_head
chunks_head/000171
chunks_head/000172
wal
wal/00000671
wal/00000672
wal/00000673
wal/00000674
wal/checkpoint.00000670
wal/checkpoint.00000670/00000000
01GZ8P7W8AY9G3S1CK3SNQZNN6
01GZ8P7W8AY9G3S1CK3SNQZNN6/chunks
01GZ8P7W8AY9G3S1CK3SNQZNN6/chunks/000001
01GZ8P7W8AY9G3S1CK3SNQZNN6/index
01GZ8P7W8AY9G3S1CK3SNQZNN6/meta.json
01GZ8P7W8AY9G3S1CK3SNQZNN6/tombstones
```

## debugging GARP when frontend loadbalancer fails over

with a recent use-case, to identify if we do receive the Gratuitous Arp (GARP) on the OCP nodes in case, an external Load Balancer fail overs, we want to evaluate our possibilities.
Now with most of our LABs not having an Appliance to failover, we'll send a GARP from a different system and see if we receive it.


check that the IP address isn't in your arp cache already (verification on process)

```
arp -an | grep 10.14.0.1

```

execute in our debug node shell
```
tshark -i any -f 'arp' -O arp -Y 'arp.isgratuitous'
```

execute on your `Appliance` 

```
echo 1 > /proc/sys/net/ipv4/ip_nonlocal_bind
# arping -c1 -A -I <pick-your-out-interface/eth0> -s <pick-an-ip/10.14.0.1> <destination>
arping -c1 -A -I br0 -s 10.14.0.1 192.168.192.128
ARPING 10.14.0.1 from 10.14.0.1 br0
Sent 1 probes (1 broadcast(s))
Received 0 response(s)
```

you'll see in your debug node shell scrolling

```
Frame 15: 62 bytes on wire (496 bits), 62 bytes captured (496 bits) on interface any, id 0
Linux cooked capture v1
Address Resolution Protocol (reply/gratuitous ARP)
    Hardware type: Ethernet (1)
    Protocol type: IPv4 (0x0800)
    Hardware size: 6
    Protocol size: 4
    Opcode: reply (2)
    [Is gratuitous: True]
    Sender MAC address: Elitegro_04:15:37 (f4:4d:30:04:15:37)
    Sender IP address: 10.14.0.1
    Target MAC address: Elitegro_04:15:37 (f4:4d:30:04:15:37)
    Target IP address: 10.14.0.1

Frame 16: 62 bytes on wire (496 bits), 62 bytes captured (496 bits) on interface any, id 0
Linux cooked capture v1
Address Resolution Protocol (reply/gratuitous ARP)
    Hardware type: Ethernet (1)
    Protocol type: IPv4 (0x0800)
    Hardware size: 6
    Protocol size: 4
    Opcode: reply (2)
    [Is gratuitous: True]
    Sender MAC address: Elitegro_04:15:37 (f4:4d:30:04:15:37)
    Sender IP address: 10.14.0.1
    Target MAC address: Elitegro_04:15:37 (f4:4d:30:04:15:37)
    Target IP address: 10.14.0.1
```

to update the remote cache with the ARP, we need to change the arping mode to

```
arping -c1 -U -I br0 -s 10.14.0.1 192.168.192.128
ARPING 10.14.0.1 from 10.14.0.1 br0
Unicast reply from 192.168.192.128 [52:54:00:DB:59:BE]  1.091ms
Unicast reply from 192.168.192.128 [52:54:00:DB:59:BE]  1.111ms
Sent 1 probes (1 broadcast(s))
Received 2 response(s)
```

verify it in your arp cache as well 

```
arp -an | grep 10.14.0.1
? (10.14.0.1) at f4:4d:30:04:15:37 [ether] on br-ex
```

**NOTE**: if you picked an IP that isn't on your Cluster, you'll face the issue that arp will refuse to remove the entry

```
arp -d 10.14.0.1
SIOCDARP(dontpub): Network is unreachable

```

To get the record removed from the cache, add a network definition first and remove it afterwards.

```
ip addr add 10.14.0.0/24 dev br-ex
arp -d 10.14.0.1
ip addr del 10.14.0.0/24 dev br-ex
```

