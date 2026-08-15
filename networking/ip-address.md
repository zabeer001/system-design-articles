## Networking Summary — What We Covered

The big picture is:

```text
IPv4
  ↓
IP Address
  ↓
CIDR
  ↓
Network / Subnet
  ↓
VPC
  ↓
EC2 machines inside subnets
  ↓
Communication between those machines
```

### 1. IPv4

IPv4 is an **IP addressing system/protocol** used to identify network interfaces and organize networks.

An IPv4 address has **32 bits**, divided into four 8-bit sections:

```text
192.168.1.25

192  . 168  . 1    . 25
8-bit  8-bit  8-bit  8-bit

Total = 32 bits
```

Each 8-bit section has:

```text
2^8 = 256 possibilities

0 → 255
```

That's why IPv4 addresses look like:

```text
10.0.2.15
192.168.1.100
172.16.5.20
```

---

## 2. Network vs Host

This was one of the most important distinctions.

Suppose:

```text
192.168.1.0/24
```

This represents a **network/subnet**.

Inside it, there can be host addresses:

```text
Network: 192.168.1.0/24

├── 192.168.1.1     Host
├── 192.168.1.2     Host
├── 192.168.1.3     Host
├── 192.168.1.25    Host
└── 192.168.1.100   Host
```

A **host** is basically a device/network interface using an IP address—for your DevOps examples, think EC2/server.

---

## 3. What `/24` means

IPv4 has **32 bits**.

```text
192.168.1.0/24
              ↑
```

`/24` means:

> The first **24 bits identify the network**.

That leaves:

```text
32 - 24 = 8 bits
```

for addresses within that subnet.

Therefore:

```text
2^8 = 256 addresses
```

So:

```text
192.168.1.0/24

192.168.1.0
192.168.1.1
192.168.1.2
...
192.168.1.255
```

There are **256 total addresses** in that `/24` range. Not all are usable host addresses, and AWS reserves some addresses as well.

---

## 4. CIDR

The notation:

```text
10.0.0.0/16
192.168.1.0/24
```

is called **CIDR notation**.

The `/16`, `/24`, etc. tell us how many bits are being used for the **network prefix**.

For example:

```text
/16
32 - 16 = 16 remaining bits

2^16 = 65,536 total addresses
```

While:

```text
/24
32 - 24 = 8 remaining bits

2^8 = 256 total addresses
```

So a `/16` network is much larger than a `/24`.

---

## 5. One large network can contain smaller subnets

This became important when we moved to AWS.

Suppose your VPC is:

```text
10.0.0.0/16
```

You can divide part of that address space into `/24` subnets:

```text
10.0.0.0/16
      │
      ├── 10.0.1.0/24
      ├── 10.0.2.0/24
      ├── 10.0.3.0/24
      ├── 10.0.4.0/24
      │
      └── ...
```

These are **different subnets**, but they're all inside the same VPC address space.

---

# 6. AWS VPC

A **VPC (Virtual Private Cloud)** is your logically isolated network environment in AWS.

For example:

```text
VPC
10.0.0.0/16
```

Then you create subnets inside that VPC.

For your microservice example:

```text
VPC: 10.0.0.0/16

│
├── Public Subnet
│   10.0.1.0/24
│
├── Private App Subnet
│   10.0.2.0/24
│
└── Private DB Subnet
    10.0.3.0/24
```

All three are **IPv4 network ranges** written using CIDR notation.

---

# 7. Your microservice architecture

You had:

* HR service
* Accounts service
* Database
* Load balancer

A simplified architecture could be:

```text
                    INTERNET
                       │
                       ↓
              ┌─────────────────┐
              │  Load Balancer  │
              └────────┬────────┘
                       │
        Public Subnet: 10.0.1.0/24
                       │
                       ↓
        Private App Subnet
             10.0.2.0/24
               │       │
               │       │
        ┌──────┘       └──────┐
        ↓                     ↓
   HR EC2                Accounts EC2
 10.0.2.10                10.0.2.20
        │                     │
        └──────────┬──────────┘
                   ↓
          Private DB Subnet
             10.0.3.0/24
                   │
                   ↓
             Database EC2
              10.0.3.10
```

The EC2 instances are **different machines**, even though HR and Accounts can be in the same subnet.

---

# 8. Network address vs host IP

Don't mix these two up.

### Network/subnet

```text
10.0.2.0/24
```

means:

> This is the subnet/network range.

### Host

```text
10.0.2.10
```

means:

> This is an IP assigned to a particular network interface, such as your HR EC2.

Therefore:

```text
Private App Network
10.0.2.0/24
       │
       ├── HR EC2
       │   10.0.2.10
       │
       └── Accounts EC2
           10.0.2.20
```

This distinction is fundamental.

---

# 9. Different subnet ≠ different VPC

You can have:

```text
10.0.1.0/24
10.0.2.0/24
10.0.3.0/24
```

They are **three different subnets**.

But all can belong to:

```text
VPC: 10.0.0.0/16
```

So conceptually:

```text
VPC = large network environment

Subnet = smaller network/range inside it

EC2 = machine/interface receiving an IP from a subnet
```

---

# 10. The mental model to keep

For now, keep this picture in your head:

```text
IPv4
│
│  32-bit addressing
│
↓
10.0.0.0/16
│
│  CIDR defines network prefix/size
│
↓
VPC
│
├── Subnet 10.0.1.0/24
│      └── Load Balancer
│
├── Subnet 10.0.2.0/24
│      ├── HR EC2      → 10.0.2.10
│      └── Accounts EC2 → 10.0.2.20
│
└── Subnet 10.0.3.0/24
       └── DB EC2      → 10.0.3.10
```

And the **next networking concept I'd learn is routing**.

Because now you have a very natural question:

> **If HR is `10.0.2.10` and Database is `10.0.3.10`, how does a packet know where to go?**

That question takes you directly into **routers, route tables, gateways, and `0.0.0.0/0`**—which is the right next step for DevOps networking.
