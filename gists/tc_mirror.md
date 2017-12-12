# Connecting a veth device to tap

- veth device from CNI/CNM plugin: eth0
- tap device that connects to the VM: tap0

## Redirecting traffic between the two devices

```
tc qdisc add dev eth0 ingress
tc filter add dev eth0 parent ffff: protocol all u32 match u8 0 0 action mirred egress redirect dev tap0
tc qdisc add dev tap0 ingress
tc filter add dev tap0 parent ffff: protocol all u32 match u8 0 0 action mirred egress redirect dev eth0
```

