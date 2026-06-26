---
title: Working with YANG Models
date: 2026-06-25 05:50:15 +0000
categories: [PROGRAMMABILITY, NETWORKING, CISCO, NETWORK AUTOMATION]
tags: [automation, networking]     # TAG names should always be lowercase
---

## Pyang Installation

Let's begin with installing the Pyang package. It's currently on version 2.7.1. I created a virtual environment on my server for this.

```
chris@chris-server:~$ python3 -m venv pyang
chris@chris-server:~$ source pyang/bin/activate
(pyang) chris@chris-server:~$ python -m pip install pyang
```
I'm also going to clone a repository containing Yang models, ranging from standard IETF to Cisco-based.

```
git clone https://github.com/YangModels/yang
```
In the meantime, I'm going to spin up a three routers inside of ContainerLab; they're CSR1000vs. I'll just use only one node. Since I'm going to use RESTCONF, I need to enable HTTP on the node as well as RESTCONF itself.

```
r1(config)#ip http server
r1(config)#ip http authentication local
r1(config)#restconf
```
To verify RESTCONF is working, we'll need to make sure NGINX is running. NGINX acts as an internal web server proving TLS-based HTTPS. When we query our box via RESTCONF, the NGINX process will receive the request first and pass it to the confd web server which performs RESTCONF request validation and actually interfaces with the IOS XE data models. 

```
r1#sh plat soft yang-management pro
confd            : Running    
nesd             : Running    
syncfd           : Running    
ncsshd           : Running    
dmiauthd         : Running    
nginx            : Running    
ndbmand          : Running    
pubd             : Running    
```

## Using RESTCONF Queries

Let's query our device. I'll use a 'curl' command on my terminal. According to the man page, it's a tool for transferring data from or to a server using URLs. In our case, we're crafting an HTTPS GET to understand our device's capabilities. Self-admittingly, I had to use the '/' inside of man to search for different flags to see what they do.

Note: I had a lot of issues trying to get authentication to work. If you experience the same thing, I suggest creating another privilege 15 set of credentials. This fixed the authentication issues I was having.

```
curl -k -u chris:chris -H "Accept: application/yang-data+json" https://172.20.20.3/restconf/data/ietf-yang-library:modules-state
```

Since every HTTPS connection involves verifying the server's TLS certificate, you can use the -k flag to ignore this verification process. You can use the -H flag to specify any additional headers in your query. 

I decided to add 'grep' to find a YANG model related to our configuration on any of the interfaces on the node. I'll use an IETF model for simplicity.

```
curl -k -u chris:chris -H "Accept: application/yang-data+json" https://172.20.20.3/restconf/data/ietf-yang-library:modules-state | grep ietf.*int
```
```
"name": "ietf-interfaces",
        "schema": "https://10.0.0.15:443/restconf/tailf/modules/ietf-interfaces/2014-05-08",
        "namespace": "urn:ietf:params:xml:ns:yang:ietf-interfaces",
        "name": "ietf-interfaces-ext",
        "schema": "https://10.0.0.15:443/restconf/tailf/modules/ietf-interfaces-ext",
        "namespace": "urn:ietf:params:xml:ns:yang:ietf-interfaces-ext",
```

The schema URI provides a reference to the module definition (i.e. ietf-interfaces), defining the available containers and data in the model.  These containers are used to craft entire RESTCONF resource paths which we'll later use with curl to retrieve our interfaces' configurations.

```
curl -k -u chris:chris -H "Accept: application/yang-data+json" "https://172.20.20.3:443/restconf/tailf/modules/ietf-interfaces/2014-05-08" | grep container
```

```
 container interfaces {
  container interfaces-state {
      container statistics {
```

Let's query the regular 'interfaces' container. 

```
curl -k -u chris:chris -H "Accept: application/yang-data+json" https://172.20.20.3/restconf/data/ietf-interfaces:interfaces
```

We've gotten our response formatted in JSON per our instructions in the header of the HTTPS GET query. I don't have anything configured yet because it's a fresh node inside of ContainerLab.  

```
{
  "ietf-interfaces:interfaces": {
    "interface": [
      {
        "name": "GigabitEthernet1",
        "description": "Containerlab management interface",
        "type": "iana-if-type:ethernetCsmacd",
        "enabled": true,
        "ietf-ip:ipv4": {
          "address": [
            {
              "ip": "10.0.0.15",
              "netmask": "255.255.255.0"
            }
          ]
        },
        "ietf-ip:ipv6": {
          "address": [
            {
              "ip": "2001:db8::2",
              "prefix-length": 64
            }
          ]
        }
      },
      {
        "name": "GigabitEthernet2",
        "type": "iana-if-type:ethernetCsmacd",
        "enabled": false,
        "ietf-ip:ipv4": {
        },
        "ietf-ip:ipv6": {
        }
      }
    ]
  }
}
```
## Using Pyang

Earlier, I queried the device directly using curl. We could've used pyang with our locally cloned repository to inspect and 'walk' the ietf-interfaces YANG model in a more human-readable form. You'll see that this method is much more preferable. We can parse the file locally and vividly see the model's structure.

You can use the -f flag to choose the output that Pyang provides. I'm using tree but there are numerous options if you check the man page.

```
pyang -f tree yang/standard/ietf/RFC/ietf-interfaces.yang
module: ietf-interfaces
  +--rw interfaces
  |  +--rw interface* [name]
  |     +--rw name                        string
  |     +--rw description?                string
  |     +--rw type                        identityref
<output omitted for brevity>
```

Using pyang, we're able to glean the container 'interfaces' much easier than the laborious task of using curl. From here, you can use NETCONF or RESTCONF to send either an XML or JSON payload to the device in accordance with the ietf-interfaces YANG model.

Now, we can use curl to accomplish the same thing as earlier, but pyang enabled us to comb through the ietf-interfaces YANG model in a much easier, human-readable manner rather than having to use curl to a) grep for the YANG models with the word 'interface' in it, and b) grab the schema URI and curl to that to see which containers exist in the model. By using pyang, I'm able to immediately see the schema structure.