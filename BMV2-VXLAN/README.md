# Experiment with BMV2 + VXLAN interface to forward traffic between machines

## Requirement
1. Two Linux Machines(Ex.Ubuntu)
2. Install docker
   
## Configuration

Prepare VXLAN interface between 2 machine

Machine A
```
sudo ip link add peering type vxlan id 100 local 172.20.14.48 remote 172.20.14.49 dstport 4789 dev ens160
sudo ip link set peering up
```
Machine B
```
sudo ip link add peering type vxlan id 100 local 172.20.14.49 remote 172.20.14.48 dstport 4789 dev ens160
sudo ip link set peering up
```

Prepare a namespace to simulate as host connecting to BMV2 switch

Machine A
```
sudo ip netns add blue
sudo ip link add blue-in type veth peer name blue-out
sudo ip link set blue-in netns blue
sudo ip netns exec blue ip addr add 192.168.60.48/24 dev blue-in
sudo ip netns exec blue ip link set blue-in up
sudo ip link set blue-out up
```

Machine B
```
sudo ip netns add blue
sudo ip link add blue-in type veth peer name blue-out
sudo ip link set blue-in netns blue
sudo ip netns exec blue ip addr add 192.168.60.49/24 dev blue-in
sudo ip netns exec blue ip link set blue-in up
sudo ip link set blue-out up
```

Create a directory and put in P4info file and JSON file, in this example; /home/netadmin/p4. This directory need to mount to container later
```
mkdir p4
```

Initialize BMV2 Container and start the Simple_switch_grpc target

Machine A
```
sudo docker run --privileged --net=host --rm -it -v /home/netadmin/p4:/p4 --name bmv2 -d p4lang/behavioral-model 
sudo docker exec -it bmv2 /bin/bash
simple_switch_grpc --log-console --device-id 1 -i 1@peering -i 2@blue-out --pcap=/p4 -Ldebug --no-p4 -- --cpu-port 255 --grpc-server-add 0.0.0.0:50001
```

Machine B
```
sudo docker run --privileged --net=host --rm -it -v /home/netadmin/p4:/p4 --name bmv2 -d p4lang/behavioral-model simple_switch_grpc --log-console --device-id 2 -i 1@peering -i 2@blue-out --pcap=/p4 -Ldebug --no-p4 -- --cpu-port 255 --grpc-server-add 0.0.0.0:50001
```

The --pcap=/p4 is for packet capture on the switch interfaces


Use of P4RuntimeShell container as our P4RuntimeClient to program the switch table entry

Machine A
```
sudo docker run -ti -v /home/netadmin/p4:/tmp/ p4lang/p4runtime-sh --grpc-addr 172.20.14.48:50001 --device-id 1 --election-id 0,1 --config /tmp/p4info.txt,/tmp/basic.json

te=table_entry["MyIngress.ipv4_lpm"](action="MyIngress.ipv4_forward")
te.match["hdr.ipv4.dstAddr"] = "192.168.60.49/32"
te.action["port"] = "1"
te.action["dstAddr"] = "1e:6c:9f:94:03:71"
te.insert()

te=table_entry["MyIngress.ipv4_lpm"](action="MyIngress.ipv4_forward")
te.match["hdr.ipv4.dstAddr"] = "192.168.60.48/32"
te.action["port"] = "1"
te.action["dstAddr"] = "e2:68:55:d6:53:e1"
te.insert()
```

Machine B
```
sudo docker run -ti -v /home/netadmin/p4:/tmp/ p4lang/p4runtime-sh --grpc-addr 172.20.14.49:50001 --device-id 2 --election-id 0,1 --config /tmp/p4info.txt,/tmp/basic.json

te=table_entry["MyIngress.ipv4_lpm"](action="MyIngress.ipv4_forward")
te.match["hdr.ipv4.dstAddr"] = "192.168.60.49/32"
te.action["port"] = "2"
te.action["dstAddr"] = "1e:6c:9f:94:03:71"
te.insert()

te=table_entry["MyIngress.ipv4_lpm"](action="MyIngress.ipv4_forward")
te.match["hdr.ipv4.dstAddr"] = "192.168.60.48/32"
te.action["port"] = "1"
te.action["dstAddr"] = "e2:68:55:d6:53:e1"
te.insert()
```

## Compiling P4 program

Put your .p4 program at directory, and mount it to p4c compiler container
```
p4c --target bmv2 --arch v1model basic.p4
```













