# Cloud-Gateway builds VPN Among Services in the Cloud

At [x-cellent](https://www.x-cellent.com/), we are building and maintaining [cloud-native systems](https://metal-stack.io/) and often we are faced with complicated networking problems when we try to connect two applications built with different generations of technologies. On the one hand, the network traffic is often, if not always, regulated by multiple firewalls and/or NAT. On the other hand, some legacy systems use private IP addresses outside the RFC 1918 space, which makes direct routing impossible. That's where **_Cloud-Gateway_** comes in handy.

## Demo

Here's a minimal [_docker-compose.yaml_](https://github.com/x-cellent/cloud-gateway-demo/tree/main/demo/minimal/docker-compose.yaml).

```yaml
version: "3.8"
networks:
  legacy:
    name: legacy
  cloudnative:
    name: cloudnative
services:
  nginx:
    networks:
      - legacy
    image: nginx
  server:
    networks:
      - legacy
      - cloudnative
    image: ghcr.io/fi-ts/cloud-gateway:latest
    volumes:
      - /dev/net/tun:/dev/net/tun
      - ./server:/etc/cloudgateway
    cap_add:
      - NET_ADMIN
    environment:
      - CLOUD_GATEWAY_ENGINE=boringtun
      - CLOUD_GATEWAY_LOG_LEVEL=debug
  client:
    networks:
      - cloudnative
    image: ghcr.io/fi-ts/cloud-gateway:latest
    volumes:
      - /dev/net/tun:/dev/net/tun
      - ./client:/etc/cloudgateway
    cap_add:
      - NET_ADMIN
    environment:
      - CLOUD_GATEWAY_ENGINE=boringtun
      - CLOUD_GATEWAY_LOG_LEVEL=debug
    ports:
      - 8080:8080
```

- Two docker networks: _legacy_ and _cloudnative_, representing two private networks
- An nginx process runs in the former network and we want to tap into that from the latter network.
- Two _Cloud-Gateway_ containers: _server_ and _client_. The server sits in both networks, whereas the client sits only in the network _cloudnative_. Note that the configuration of _Cloud-Gateway_ is provided via _volumes_.
- The port _8080_ of the host is mapped to the same port of the client container.

Our target is to tap into the nginx running behind the server. Note that we don't have access to the network _legacy_. How can we go about achieving that? We build a tunnel between _Cloud-Gateway_ server and client. Let's see that in action!

```sh
git clone git@github.com:x-cellent/cloud-gateway-demo.git
cd cloud-gateway-demo/demo/minimal
docker compose up
```

Now, our demo is up and running. Let's open another terminal and run `curl localhost:8080`. Voilà! We shall see the default welcome page of nginx.

- Cloud-Gateway connects apps in different private networks securely and easily.
- Cloud-Gateway is backed by end-to-end VPN and easy-to-configure TCP proxy.

## VPN with WireGuard&reg;

From the logs of `docker compose up`, two specific lines are of interest to us. (They might not be adjacent to each other.)

```terminal
minimal-server-1  | [#] ip -4 address add 10.192.0.121/24 dev wg0
...
minimal-client-1  | [#] ip -4 address add 10.192.0.245/32 dev wg0
```

- Under the hood, Cloud-Gateway is powered by [WireGuard&reg;](https://www.wireguard.com/), which creates the interface _wg0_. This interface is assigned an IP address and a CIDR mask.
- The IP address is rather clear - a private IP address in the VPN.
- The CIDR masks of the server and the client are different.
- For the server, _/24_ implies that there are multiple peers, which means multiple clients in Cloud-Gateway. (There's no differentiation between server and client in WireGuard&reg;, but there is one in Cloud-Gateway.) Those clients are assigned private IP addresses in the range _10.192.0.0/24_.
- For the client, in contrast, there's only one peer - the server, since the client's going to talk to the server exclusively.

Let's `docker exec` into the containers and dig more! In one terminal, run:

```sh
docker exec -it minimal-server-1 bash
```

In another terminal, run:

```sh
docker exec -it minimal-client-1 bash
```

In each terminal, run:

```sh
wg
```

The outputs at the server should read as follows:

```terminal
interface: wg0
  public key: 2o3hItYcvPrcmDMog6rOhmdzZd6PH+QIZtCvZnVrslU=
  private key: (hidden)
  listening port: 8765

peer: QqdKxpGpG1TTuyjCayMpRRKwLm5kjyIKZybxjv8oDnQ=
  endpoint: 172.24.0.2:55748
  allowed ips: 10.192.0.245/32
  latest handshake: 2 minutes, 34 seconds ago
  transfer: 948 B received, 860 B sent
```

The outputs at the client:

```terminal
interface: wg0
  public key: QqdKxpGpG1TTuyjCayMpRRKwLm5kjyIKZybxjv8oDnQ=
  private key: (hidden)
  listening port: 55748

peer: 2o3hItYcvPrcmDMog6rOhmdzZd6PH+QIZtCvZnVrslU=
  endpoint: 172.24.0.3:8765
  allowed ips: 10.192.0.121/32
  latest handshake: 2 minutes, 47 seconds ago
  transfer: 860 B received, 1.07 KiB sent
```

Notice that for the _wg0_ at the server, the _listening port_ _8765_ is assigned by Cloud-Gateway, whereas at the client the _listening port_ is randomly assigned by WireGuard&reg;. The reason behind this is that Cloud-Gateway was conceived where we need a server sitting in the network _legacy_ and this server should respond to requests from applications in a cloud-native environment, like kubernetes. In order to let other clients find the server in the public network, the port for VPN communication at the server needs to be fixed.

The _endpoint_ is the peer's endpoint in the public network, so that the server and client can find each other. In this demo, the endpoint is the IP address of the container. We can prove that in another terminal.

```sh
docker inspect minimal-server-1 -f "{{ .NetworkSettings.Networks.cloudnative.IPAddress }}"
docker inspect minimal-client-1 -f "{{ .NetworkSettings.Networks.cloudnative.IPAddress }}"
```

Let's observe how the packets are flowing in the VPN! First, ping the server's VPN IP address at the client (still inside the container).

```sh
ping 10.192.0.121
```

Then, observe what we've got at the server.

```sh
apt update && apt install tcp -y
tcpdump --interface wg0
```

The outputs should read as follows:

```terminal
listening on wg0, link-type RAW (Raw IP), snapshot length 262144 bytes
13:02:16.129213 IP 10.192.0.245 > 10.192.0.121: ICMP echo request, id 64908, seq 1, length 64
13:02:16.129257 IP 10.192.0.121 > 10.192.0.245: ICMP echo reply, id 64908, seq 1, length 64
...
```

We shall see ICMP echo requests and replies being exchanged between the server and the client through the interface _wg0_. Note that _wg0_ is a virtual interface constructed by the VPN protocol, so there must be another real interface involved. Interrupt the running tcpdump at the server and run it against the interface _eth0_ this time.

```sh
tcpdump --interface eth0
```

The outputs:

```terminal
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
14:03:56.499668 IP minimal-client-1.cloudnative.51764 > e3b9d54cbacf.8765: UDP, length 128
14:03:56.499952 IP e3b9d54cbacf.8765 > minimal-client-1.cloudnative.51764: UDP, length 128
...
```

This time, we see the UDP packets being exchanged between the client and the server, where _minimal-client-1.cloudnative.51764_ represents the domain name and the UDP port of the client container, _e3b9d54cbacf.8765_ those of the server. We can examine that with the tool `nslookup`.

```sh
apt update && apt install nsutils -y
nslookup minimal-client-1
nslookup e3b9d54cbacf
```

We shall see the IP addresses of the containers. Another way is simply running `tcpdump --interface eth0 -n`, which doesn't convert addresses to names.

So far, we've examined the traffic between the VPN IP addresses at the interface _wg0_ and we know it's backed by the underlying UDP communication at the interface _eth0_. How about if we ping the server's public IP address directly? Will we see anything at server's _wg0_?

```sh
# at client
ping 172.24.0.2 # server's public IP address
```

If we run `tcpdump --interface wg0` at the server, we will observe nothing. Why? Remember that only the traffic between VPN IP addresses, in our case 10.192.0.121 of the server and 10.192.0.245 of the client, is going through the interface _wg0_ on both sides. Apart from that, it's just business as usual and the public interface _eth0_ takes charge.

```sh
# at server
tcpdump --interface eth0
```

We shall observe ICMP echos between the server and the client.

```terminal
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
14:54:57.778659 IP minimal-client-1.cloudnative > e3b9d54cbacf: ICMP echo request, id 63736, seq 11, length 64
14:54:57.778687 IP e3b9d54cbacf > minimal-client-1.cloudnative: ICMP echo reply, id 63736, seq 11, length 64
```

## Pipe

Another pillar of Cloud-Gateway is _pipe_, which can be found in Cloud-Gateway's configuration of both the [server](https://github.com/x-cellent/cloud-gateway-demo/tree/main/demo/minimal/server/conf.yaml) and the [client](https://github.com/x-cellent/cloud-gateway-demo/tree/main/demo/minimal/client/conf.yaml). Remember that this file is consumed by Cloud-Gateway via _volumes_ in the _docker-compose.yaml_ above.

```yaml
name: nginx
port: 8080
remote: nginx:80
```

- The _name_ is just a name for humans.
- The _port_ is the listening port of the built-in TCP proxy of Cloud-Gateway. Each pipe has its own port.
- The _remote_ part is the endpoint of a remote service of interest.

Let's examine this port. Run the following command inside the server and client container:

```sh
ss -tulpen
```

The outputs at the server should read as follows:

```terminal
Netid       State        Recv-Q        Send-Q               Local Address:Port                Peer Address:Port       Process
udp         UNCONN       0             0                       127.0.0.11:56608                    0.0.0.0:*           ino:27370005 sk:15 cgroup:unreachable:e8de1 <->
udp         UNCONN       0             0                          0.0.0.0:8765                     0.0.0.0:*           ino:27372057 sk:16 cgroup:unreachable:1 <->
udp         UNCONN       0             0                             [::]:8765                        [::]:*           ino:27372058 sk:17 cgroup:unreachable:1 v6only:1 <->
tcp         LISTEN       0             4096                    127.0.0.11:46841                    0.0.0.0:*           ino:27370006 sk:18 cgroup:unreachable:e8de1 <->
tcp         LISTEN       0             4096                             *:9000                           *:*           users:(("cloudgateway",pid=1,fd=11)) ino:27359215 sk:19 cgroup:unreachable:1 v6only:0 <->
tcp         LISTEN       0             4096                             *:8080                           *:*           users:(("cloudgateway",pid=1,fd=10)) ino:27359212 sk:1a cgroup:unreachable:1 v6only:0 <->
```

- Two ports bound to _127.0.0.11_ relate to docker DNS handlers. They appear when we specify our own docker network.
- The port _8765_ bound to all addresses is for UDP communication in VPN as mentioned above.
- The port _9000_ is for the built-in metrics server.
- The port _8080_ is the listening port of the built-in TCP proxy for the pipe above.

The outputs from the client is basically the same, except the random port for UDP communication in VPN.

The _remote_ part of the pipe, _nginx:80_, is more intricate. At the server, it's left intact since _nginx_ is a known domain name in the network _legacy_. Remember that the server and the nginx containers sit in the same network. At the client, it's translated to _{server's VPN IP}:{proxy port}_, _10.192.0.121:8080_ in our case. Let's see that in action!

```sh
# at server
tcpdump --interface wg0 -n
```

```sh
# at client
curl localhost:8080
```

At the client, we shall see the welcome page of nginx; at the server, we shall observe a couple of packets exchanged by the client and the server as follows.

```terminal
listening on wg0, link-type RAW (Raw IP), snapshot length 262144 bytes
15:44:43.961628 IP 10.192.0.245.57990 > 10.192.0.121.8080: Flags [S], seq 1160242459, win 64860, options [mss 1380,sackOK,TS val 331828805 ecr 0,nop,wscale 7], length 0
15:44:43.961682 IP 10.192.0.121.8080 > 10.192.0.245.57990: Flags [S.], seq 3478160883, ack 1160242460, win 64296, options [mss 1380,sackOK,TS val 1567599720 ecr 331828805,nop,wscale 7], length 0
...
```

We recognize the translated endpoint of the remote service, _10.192.0.121.8080_, but what about the source port _57990_? We don't see this port if we run `ss -tulpen` at the client. Let's dive in!

All the traffic arriving at the pipe's port is handled by the TCP proxy, where there're two TCP connections constantly being synced. One is called source, the other destination. The source connection connects to the listening port _8080_ and the destination connection connects to the endpoint of the remote service. At the client, these two connections look as follows:

```terminal
Source: 127.0.0.1:8080 (local) <-> 127.0.0.1:53236 (remote)
Destination: 10.192.0.245:57990 (local) <-> 10.192.0.121:8080 (remote)
```

Similarly at the server:

```terminal
Source: 10.192.0.121:8080 (local) <-> 10.192.0.245:57990 (remote)
Destination: 172.25.0.2:40160 (local) <-> 172.25.0.3:80 (remote)
```

Note that the destination TCP connection at the client is exactly the source at the server with the reverse local and remote endpoints. Note that the _local_ and _remote_ in the parentheses denote `LocalAddr` and `RemoteAddr` in [`Conn`](https://pkg.go.dev/net#Conn) respectively. The port _57990_ we observed in the outputs of `tcpdump` at the server is exactly the one involved in this very TCP connection.

Let us finish this section by examining the traffic at the interface _eth0_ again.

```sh
# at server
tcpdump --interface eth0 -n
```

```sh
# at client
curl localhost:8080
```

At the server, the outputs should read as follows:

```terminal
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
08:46:58.961805 IP 172.24.0.2.55142 > 172.24.0.3.8765: UDP, length 96
08:46:58.961846 IP 172.24.0.3.8765 > 172.24.0.2.55142: UDP, length 96
...
```

As we expected, the traffic between the server and the client goes through the VPN's UDP ports.

In summary, when we run `curl localhost:8080` at the host, the life of a packet would look as follows:

```terminal
localhost:8080 (mapped to the client container's port 8080)
-> client-container-IP:8080 (configured port of a pipe)
-> Cloud-Gateway's TCP proxy syncing source and destination TCP connections at the client
-> server-VPN-IP:8080 (traffic actually going through VPN UDP ports)
-> Cloud-Gateway's TCP proxy syncing two connections at the server
-> nginx-container-IP:80 (nginx port)
```

## Reverse Pipe

How about if we want to reach a service in the network _cloudnative_ from a client in the network _legacy_? _Reverse pipe_ comes to rescue. In contrast to normal pipes, where a service is resolvable within the server's network without going through the VPN to another network, a revere pipe is for the scenario where a service does sit in another network and the server needs the VPN to fetch it. Here's the [_docker-compose.yaml_](https://github.com/x-cellent/cloud-gateway-demo/tree/main/demo/reverse/docker-compose.yaml) for such a reverse pipe.

```yaml
version: "3.8"
networks:
  legacy:
    name: legacy
  cloudnative:
    name: cloudnative
services:
  nginx:
    networks:
      - cloudnative
    image: nginx
  server:
    networks:
      - legacy
      - cloudnative
    image: ghcr.io/fi-ts/cloud-gateway:latest
    volumes:
      - /dev/net/tun:/dev/net/tun
      - ./server:/etc/cloudgateway
    cap_add:
      - NET_ADMIN
    environment:
      - CLOUD_GATEWAY_ENGINE=boringtun
      - CLOUD_GATEWAY_LOG_LEVEL=debug
  client-cloudnative:
    networks:
      - cloudnative
    image: ghcr.io/fi-ts/cloud-gateway:latest
    volumes:
      - /dev/net/tun:/dev/net/tun
      - ./client/cloudnative:/etc/cloudgateway
    cap_add:
      - NET_ADMIN
    environment:
      - CLOUD_GATEWAY_ENGINE=boringtun
      - CLOUD_GATEWAY_LOG_LEVEL=debug
  client-legacy:
    networks:
      - legacy
    image: ghcr.io/fi-ts/cloud-gateway:latest
    volumes:
      - /dev/net/tun:/dev/net/tun
      - ./client/legacy:/etc/cloudgateway
    cap_add:
      - NET_ADMIN
    environment:
      - CLOUD_GATEWAY_ENGINE=boringtun
      - CLOUD_GATEWAY_LOG_LEVEL=debug
    ports:
      - 8080:8080
```

This time, nginx is sitting in the network _cloudnative_ instead of _legacy_ and there are two clients instead of one, each sitting in their respective network. Note that the port 8080 at the host is mapped to the port 8080 in the _client-legacy_ container to mimic the use case. Let's see the reverse pipe in action!

```sh
# in the project root
cd demo/reverse
docker compose up
```

Then, open another terminal and run `curl localhost:8080`. Voilà! We shall see the default welcome page of nginx again.

Let's go through the life of a packet.

```terminal
localhost:8080 (mapped to the client-legacy container port 8080)
-> client-legacy-container-IP:8080
-> Cloud-Gateway's TCP proxy syncing two TCP connections at client-legacy
-> server-VPN-IP:8080 (traffic actually going through VPN UDP ports)
-> Cloud-Gateway's TCP proxy syncing two TCP connections at the server
-> client-cloudnative-VPN-IP:8080 (VPN UDP communications)
-> Cloud-Gateway's TCP proxy syncing two TCP connections at client-cloudnative
-> nginx-container-IP:80
```

Finally, let's check the _cong.yaml_ of Cloud-Gateway for each container. Note that for the [server](https://github.com/x-cellent/cloud-gateway-demo/tree/main/demo/reverse/server/conf.yaml) and the [client](https://github.com/x-cellent/cloud-gateway-demo/tree/main/demo/reverse/client/cloudnative/conf.yaml) in the network _cloudnative_, we have to specify the peer:

```yaml
name: nginx
port: 8080
remote: nginx:80
peer: client-cloudnative
```

However, for the [client](https://github.com/x-cellent/cloud-gateway-demo/tree/main/demo/reverse/client/legacy/conf.yaml) in the network _legacy_, it's the same as the previous demo:

```yaml
name: nginx
port: 8080
remote: nginx:80
```

Why is that? Actually, in the eyes of the client in the network _legacy_, nothing has changed. It's still a pipe which must go through the server. Therefore, nothing needs to change. For the server and the client in the network _cloudnative_, it's another story. We have to tell Cloud-Gateway where the service, nginx in our case, is resolvable. To put it another way, we have to tell Cloud-Gateway to which peer the traffic has to be sent. Therefore, the name of this _peer_ has to match the one listed under _peers_ in the server's _conf.yaml_. To cut it short: specify the peer of a pipe for the server and the client where the remote endpoint of the service is resolvable.

## Conclusion

- Cloud-Gateway enables encrypted UDP communication between different private networks, which is backed by WireGuard&reg;.
- Cloud-Gateway provides intuitive _pipe_ syntax to configure the built-in TCP proxy.
