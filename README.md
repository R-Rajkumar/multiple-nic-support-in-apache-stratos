Multiple Network Interfaces Support in Apache Stratos
-----------------------------------------------------
Apache Stratos now supports assigning any number of floting IPs to any interface of your cartridge VM instances. This feature is implemented using Jclouds openstack-neutron APIs. Most importantly it is not breaking existing features. You can still use openstack with nova networking environment and stratos will use Jclouds openstack-nova networking APIs to the networking stuffs as previous. If you have openstack with neutron networking environment, you can get benefit from this feature. 

How do you tell Stratos which openstack networking API to use?
---------------------------------------------------------

Which openstack networking API to be used is configurable via cloud-controller xml.
```
    <iaasProvider type="openstack" name="openstack specific details">
        <className>org.apache.stratos.cloud.controller.iaases.OpenstackNovaIaas</className>
        <provider>openstack-nova</provider>
        ------
        <property name="openstack.networking.provider" value="neutron" />
```
The property openstack.networking.provider can be either set to nova or neutron. Default is nova. If you set anything other than these two, nova networking API will be used.

If you are using stratos-installer to set up Stratos, you can set the following parameter in stratos-installer/conf/setup.conf file under openstack configuration.

```
# Openstack
export openstack_provider_enabled=false
export openstack_identity="stratos:stratos" # Openstack project name:Openstack login user
export openstack_credential="password" # Openstack login password
export openstack_jclouds_endpoint="http://hostname:5000/v2.0"
export openstack_keypair_name=""
export openstack_security_groups=""
export openstack_networking_provider="neutron"
```
Then setup.sh will add the property, openstack.networking.provider, to cloud-controller xml automatically.

In addition these, you need to set autoAssignIp property to false in the property section of the cartridge definition. You can set this property in cloud-controller xml too. You can set it in the cartridge definition as shown below.

```
    "property":[  
        {  
            "name":"keyPair",
            "value":"raj"
        },
        {  
            "name":"autoAssignIp",
            "value":"false"
        }
        ----
    ]
```


How do you specify network interfaces in cartridge deployment json
--------------------
A user can specify multiple network interfaces with internal and floating networks and/or fixed private and floating IPs in cartridge deployment. Floating networks should be defined for each network interface. One network interface can have several floating networks. A floating network can be defined for several network interfaces as well. 

VM instances that are created from this cartridge subscription will get one private IP per network interface from internal network and floating IPs per floating network. If fixed floating IPs defined instead of network uuids, floating IPs will be assosicated to the corresponding network interfaces.

A sample cartridge definition
-----------------------------
An example network interfaces section of the cartridge definition looks like below;

~~~
 "networkInterfaces":[
     {
        "name":"eth0",
        "networkUuid":"68aab21d-fc9a-4c2f-8d15-b1e41f6f7bb8"
    },
    {
        "name":"eth1",
        "networkUuid":"512e1f54-1e85-4dac-b2e6-f0b30fc552cf",
        "floatingNetworks":[
            {
                "name":"externalOne",
                "networkUuid":"ba667f72-7ba8-4b24-b360-b74a0211c83c"
            }
        ]
    },
    {
        "name":"eth2",
        "networkUuid":"b55f009a-1cc6-4b17-924f-4ae0ee18db5e",
        "floatingNetworks":[
            {
                "name":"externalThree",
                "floatingIP" : "192.168.17.227"
            }
        ]
    },
    {
        "name":"eth3",
        "networkUuid":"d343d343-1cc6-4b17-924f-4ae0ee18db5e",
        "floatingNetworks":[
            {
                "name":"externalThree",
                "networkUuid":"ba667f72-7ba8-4b24-b360-b74a0211c83c",
                "floatingIP" : "192.168.17.227"
            }
        ]
    },
    {
        "name":"eth4",
        "portUuid":"d343d343-1cc6-4b17-924f-4ae0ee18db5e",
        "fixedIp":"10.5.62.3",
        "floatingNetworks":[
            {
                "name":"externalThree",
                "networkUuid":"ba667f72-7ba8-4b24-b360-b74a0211c83c",
                "floatingIP" : "192.168.17.227"
            }
        ]
    },
    {
        "name":"eth5",
        "networkUuid":"d343d343-1cc6-4b17-924f-4ae0ee18db5e",
        "floatingNetworks":[
            {
                "name":"externalOne",
                "networkUuid":"ba667f72-7ba8-4b24-b360-b74a0211c83c",
                "floatingIP" : "192.168.17.227"
            },
            {
                "name":"externalTwo",
                "networkUuid":"sddsdsd-7ba8-4b24-b360-b74a0211c83c",
            }
        ]
    }
]
~~~

Supported features in detail
----------------------------
1. Floating networks are optional.
    ```
    {
        "name":"eth0",
        "networkUuid":"68aab21d-fc9a-4c2f-8d15-b1e41f6f7bb8"
    }
    ```
    Interface eth0 will get a private IP from 68aab21d-fc9a-4c2f-8d15-b1e41f6f7bb8.
    
2. Floating networks' networkUuid can be specified. A floating IP will be assigned to the the interface from the given floating network. If there any unassigned floting IPs in the given floating network, one will be randomly choosen and assign to the interface. If not, a floating IP will be allocated from the given floating network and assigned to the interface. 
    ```
    {
        "name":"eth1",
        "networkUuid":"512e1f54-1e85-4dac-b2e6-f0b30fc552cf",
        "floatingNetworks":[
            {
                "name":"externalOne",
                "networkUuid":"ba667f72-7ba8-4b24-b360-b74a0211c83c"
            }
        ]
    }
    ```
    Interface eth1 will get a private IP from 512e1f54-1e85-4dac-b2e6-f0b30fc552cf and a floating IP from ba667f72-7ba8-4b24-b360-b74a0211c83c.
    
3. Floating IP can be specified. It will be considered as fixed floating IPs. Stratos will check whether the given IP is available for allocation. If so it will associate to the interface. If not, it will throw an exception.
    ```
    {
        "name":"eth2",
        "networkUuid":"b55f009a-1cc6-4b17-924f-4ae0ee18db5e",
        "floatingNetworks":[
            {
                "name":"externalThree",
                "floatingIP" : "192.168.17.227"
            }
        ]
    }
    ```
    Interface eth2 will get a private IP from b55f009a-1cc6-4b17-924f-4ae0ee18db5e and a floating IP 192.168.17.227 if available.
    
4. Both networkUuid and fixed floating IP can be specified. networkUuid will get higher priority than fixed floating IP.
    ```
    {
        "name":"eth3",
        "networkUuid":"d343d343-1cc6-4b17-924f-4ae0ee18db5e",
        "floatingNetworks":[
            {
                "name":"externalThree",
                "networkUuid":"ba667f72-7ba8-4b24-b360-b74a0211c83c",
                "floatingIP" : "192.168.17.227"
            }
        ]
    }
    ```
    Interface eth3 will get a private IP from d343d343-1cc6-4b17-924f-4ae0ee18db5e and a floating IP from ba667f72-7ba8-4b24-b360-b74a0211c83c. It might get 192.168.17.227 by chance.
    
5. Any number of floating networks can be specified for a network interface. One floating IP per floating networks will be associated to the corresponding interface. This is supported at the Stratos end. This is not confirmed yet that openstack is also supporting it, due to unavailability of openstack environment with several floating networks.
```
    {
        "name":"eth5",
        "networkUuid":"d343d343-1cc6-4b17-924f-4ae0ee18db5e",
        "floatingNetworks":[
            {
                "name":"externalOne",
                "networkUuid":"ba667f72-7ba8-4b24-b360-b74a0211c83c",
                "floatingIP" : "192.168.17.227"
            },
            {
                "name":"externalTwo",
                "networkUuid":"sddsdsd-7ba8-4b24-b360-b74a0211c83c",
            }
        ]
    }
```
Interface eth5 will get a private IP from d343d343-1cc6-4b17-924f-4ae0ee18db5e and a floating IP from ba667f72-7ba8-4b24-b360-b74a0211c83c and another floating IP from sddsdsd-7ba8-4b24-b360-b74a0211c83c.

6. Both private IPs and floating IPs can be predefined. If available they will be associated, otherwise an exception will be thrown
```
    {
        "name":"eth5",
        "portUuid":"d343d343-1cc6-4b17-924f-4ae0ee18db5e",
        "fixedIp":"10.5.62.3",
        "floatingNetworks":[
            {
                "name":"externalThree",
                "floatingIP" : "192.168.17.227"
            }
        ]
    }
```
Interface eth5 will get a fixed private IP 10.5.62.3 and a fixed floating IP 192.168.17.227. If atleast one of IP is unavailable, en exception will be thrown. 

7. Predefined floating IPs will be reserved if there are multiple floating networks defined for a cartridge. Hence an interface is guaranteed to have the floating IP defined to it.

    ```
 {
     "name":"eth1",
     "networkUuid":"512e1f54-1e85-4dac-b2e6-f0b30fc552cf",
     "floatingNetworks":[
         {
             "name":"externalOne",
             "networkUuid":"ba667f72-7ba8-4b24-b360-b74a0211c83c"
         }
     ]
 },
  {
     "name":"eth5",
     "portUuid":"d343d343-1cc6-4b17-924f-4ae0ee18db5e",
     "fixedIp":"10.5.62.3",
     "floatingNetworks":[
         {
             "name":"externalThree",
             "floatingIP" : "192.168.17.227"
         }
     ]
 }
    ```
Floating IP 192.168.17.227 will be reseerved for interface eth5, in case eth1's allocation is taking place first. Hence eth1 will not get 192.168.17.227 by any chance. That is, if 192.168.17.227 is available, it will be definitely assigned to eth5.

8. Topology and topology events now supports multiple public and private IPs per member.

Note
--
Your cartridge base image should contains multiple interface entry in /etc/network/interfaces file. For example, if you want assign a floating IP to eth1, then your cartridge base image's /etc/network/interfaces file should be like below,
```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto eth0
iface eth0 inet dhcp

auto eth1
iface eth1 inet dhcp

```

Then only, you will be able to ping to the floating IP associated to eth1. Otherwise, eventhough the IP is associated, you will not able to ping or ssh. So if you want to associate a floating IP to eth2, you should add another entry in /etc/network/interfaces.

