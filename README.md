# Azure Internal Load Balancer (ILB) hairpin

1\. Introduction
----------------

  

As per Azure documentation - [https://docs.microsoft.com/en-us/azure/load-balancer/load-balancer-overview#limitations](https://docs.microsoft.com/en-us/azure/load-balancer/load-balancer-overview#limitations) – the Azure Internal Load Balancer default behaviour is as follows 

> ..if an outbound flow from a VM in the backend pool attempts a flow to frontend of the internal Load Balancer in which pool it resides and is mapped back to itself, both legs of the flow don't match and the flow will fail.

So **what happens if your application design requires backend pool members to make calls to the private frontend of the same load balancers they are associated with?**

  

![](https://1.bp.blogspot.com/-fFyLgHgDqKs/XUgFOXKMlDI/AAAAAAAArHw/Km4-EVb-6Sgdid91KUDcC7HPiB1WoNfGACLcBGAs/s400/ilb-1.PNG)


In the above example, if VM-WE-02-Web01 initiates a connection to 10.2.1.100:80 (ILB VIP) there is a 100% chance this connection will fail. If the backend pool happened to contain other VMs (E.g. backend pool with 2 instances) then there is a chance (50/50) the frontend request would get mapped, successfully, to another backend member. As shown below:

  

![](https://1.bp.blogspot.com/-uSt0leKGPYM/XUgFyWFTAyI/AAAAAAAArH4/uoUSz9WbPdgAMdAXZC1AKVUBxj6C6CVPwCEwYBhgL/s640/ilb-2.PNG)


###   1.1 Why is this not working?

From the same documentation link 

> _When the flow maps back to itself the outbound flow appears to originate from the VM to the frontend and the corresponding inbound flow appears to originate from the VM to itself._

Let's take a look at the default NAT behaviour of the ILB to understand the problem in more detail.  

*   The Azure ILB does not perform inbound Source-NAT (SNAT) and therefore the original source IP is preserved. 
*   When using the _default_ LB rule setting of DSR (aka floating IP) disabled, we **do** perform Destination-NAT (DNAT)

![](https://1.bp.blogspot.com/-cSHYt4J2Ov8/XUgHjHWI1RI/AAAAAAAArIE/zajRDuhYzUIJ95UwyLi_J2QejOFqWtt4QCLcBGAs/s640/ilb-3.PNG)

All of the above results in the following, again from the original documentation link:

> From the guest OS's point of view, the inbound and outbound parts of the same flow don't match inside the virtual machine. The TCP stack will not recognize these halves of the same flow as being part of the same flow as the source and destination don't match

We can confirm this behaviour using WireShark. Firstly, for a flow that does work, showing a successful 3-way TCP handshake. (FYI this flow is sourced from the on-premise location, see topology diagram in the next section)  

[![](https://1.bp.blogspot.com/-67CsQqmhIHo/XUgKn6ZG-PI/AAAAAAAArIQ/QntkZXOU1eQOY6TCYqFuFKcFUKCo3FXfwCLcBGAs/s1600/wireshark-1.png)](https://1.bp.blogspot.com/-67CsQqmhIHo/XUgKn6ZG-PI/AAAAAAAArIQ/QntkZXOU1eQOY6TCYqFuFKcFUKCo3FXfwCLcBGAs/s1600/wireshark-1.png)
Wireshark - working flow

  

Now for a flow that does not work, showing a failure of the TCP handshake. We do not get past the SYN stage. As the Azure ILB performs DNAT (see frame number 7647 on the screenshot below for confirmation of this) on the return traffic, the O/S is unable to reconcile the flow and we therefore fail to observe a TCP SYN ACK.

[![](https://1.bp.blogspot.com/-WelkEP2llPc/XUgLDlMaXJI/AAAAAAAArIY/_WSsXDVN-kEfJXGqmaPTXm-jbBfWOrEXwCLcBGAs/s1600/wireshark-2.png)](https://1.bp.blogspot.com/-WelkEP2llPc/XUgLDlMaXJI/AAAAAAAArIY/_WSsXDVN-kEfJXGqmaPTXm-jbBfWOrEXwCLcBGAs/s1600/wireshark-2.png)
Wireshark - non working flow

2\. Lab Topology
----------------

Now that we have detailed the behaviour, lets look at possible workarounds. To do this I will use the following lab environment. 

[![](https://1.bp.blogspot.com/-H1cqmAUCJyA/XUgM61yHLoI/AAAAAAAArIk/q-gKVaYksDUX9kxFBX9YBwUwOBuSDoRrACLcBGAs/s640/lab.png)](https://1.bp.blogspot.com/-H1cqmAUCJyA/XUgM61yHLoI/AAAAAAAArIk/q-gKVaYksDUX9kxFBX9YBwUwOBuSDoRrACLcBGAs/s1600/lab.png)

*   Azure spoke Virtual Network (VNet) containing Windows 2016 Server VM running IIS, hosting simple web page
*   Azure spoke VNet containing Azure ILB
*   Azure hub VNet containing ExpressRoute Gateway
*   VNet peering between Hub and Spoke VNets
*   InterCloud ExpressRoute circuit providing connectivity to On-Premises
*   On-Premise DC with test client Virtual Machine

### 2.1 Baseline

From the client VM (192.168.2.1) we are able to successfully load the web page via the ILB front end.

[![](https://1.bp.blogspot.com/-gl4_Z_-CP9w/XUgOgryJwDI/AAAAAAAArIw/UvA2Wxt9ECc4tO9AzkGAkGaUH3HXjRAbQCLcBGAs/s400/ilb-10.png)](https://1.bp.blogspot.com/-gl4_Z_-CP9w/XUgOgryJwDI/AAAAAAAArIw/UvA2Wxt9ECc4tO9AzkGAkGaUH3HXjRAbQCLcBGAs/s1600/ilb-10.png)

However, from the backend VM (10.2.1.4) we are only able to load the web page using the local VM IP address. Access via the frontend ILB VIP fails, due to the condition described in section 1.

  
```
**// show single NIC**

c:\\pstools>ipconfig | findstr /i "ipv4"

   IPv4 Address. . . . . . . . . . . : 10.2.1.4

  

**// show working connectivity using localhost address**

c:\\pstools>psping -n 3 -i 0 -q 10.2.1.4:80

  

PsPing v2.10 - PsPing - ping, latency, bandwidth measurement utility

Copyright (C) 2012-2016 Mark Russinovich

Sysinternals - www.sysinternals.com

  

TCP connect to 10.2.1.4:80:

4 iterations (warmup 1) ping test: 100%

  

TCP connect statistics for 10.2.1.4:80:

  Sent = 3, Received = 3, Lost = 0 (0% loss),

  Minimum = 0.09ms, Maximum = 0.13ms, Average = 0.11ms

  

**// show baseline failure condition to front end of LB**

c:\\pstools>psping -n 3 -i 0 -q 10.2.1.100:80

  

PsPing v2.10 - PsPing - ping, latency, bandwidth measurement utility

Copyright (C) 2012-2016 Mark Russinovich

Sysinternals - www.sysinternals.com

  

TCP connect to 10.2.1.100:80:

4 iterations (warmup 1) ping test: 100%

  

TCP connect statistics for 10.2.1.100:80:

  Sent = 3, Received = 0, Lost = 3 (100% loss),

  Minimum = 0.00ms, Maximum = 0.00ms, Average = 0.00ms
```
  

  

3\. Workarounds
---------------

### 3.1. Workaround Option \[1\] - Second NIC

[![](https://1.bp.blogspot.com/-8DoJFVwB17s/XUgPUn-v0yI/AAAAAAAArI4/c2SAbpWJfissHGzNoHx36nCHOfgaDEq0ACLcBGAs/s640/second-nic.PNG)](https://1.bp.blogspot.com/-8DoJFVwB17s/XUgPUn-v0yI/AAAAAAAArI4/c2SAbpWJfissHGzNoHx36nCHOfgaDEq0ACLcBGAs/s1600/second-nic.PNG)

*   Add a second NIC to the Virtual Machine (from within the Azure VM config) with different IP address (we use .5 in the diagram above)
*   Configure local (O/S level) static route forcing traffic destined to the LB VIP out of the secondary NIC

This works as the packet from backend-to-frontend now has a different source (10.2.1.5) and destination IP address (10.2.1.100 > DNAT > 10.2.1.4). With verification as per below:

  

  
```
**//command line from web server**

  

**// show multiple NIC**  
c:\\pstools>ipconfig | findstr /i "ipv4"

   IPv4 Address. . . . . . . . . . . : 10.2.1.4

   IPv4 Address. . . . . . . . . . . : 10.2.1.5

  

**// show baseline failure condition**

c:\\pstools>psping -n 3 -i 0 -q 10.2.1.100:80

  

PsPing v2.10 - PsPing - ping, latency, bandwidth measurement utility

Copyright (C) 2012-2016 Mark Russinovich

Sysinternals - www.sysinternals.com

  

TCP connect to 10.2.1.100:80:

4 iterations (warmup 1) ping test: 100%

  

TCP connect statistics for 10.2.1.100:80:

  Sent = 3, Received = 0, Lost = 3 (100% loss),

  Minimum = 0.00ms, Maximum = 0.00ms, Average = 0.00ms

  

**// static route traffic destined to LB front end out of second NIC**

c:\\pstools>route add 10.2.1.100 mask 255.255.255.255 10.2.1.1 if 18

 OK!

  

**// show working connectivity to LB front end**

c:\\pstools>psping -n 3 -i 0 -q 10.2.1.100:80

  

PsPing v2.10 - PsPing - ping, latency, bandwidth measurement utility

Copyright (C) 2012-2016 Mark Russinovich

Sysinternals - www.sysinternals.com

  

TCP connect to 10.2.1.100:80:

4 iterations (warmup 1) ping test: 100%

  

TCP connect statistics for 10.2.1.100:80:

  Sent = 3, Received = 3, Lost = 0 (0% loss),

  Minimum = 0.68ms, Maximum = 1.45ms, Average = 0.99ms
```
  

### 3.2 Workaround Option \[2\] -Loopback VIP (+ DSR)

[![](https://1.bp.blogspot.com/-DMfVcMVHZNM/XUgRahyM_QI/AAAAAAAArJE/29h-7d-hPWYwjUipp_Kx5NnOYQFi4Y46gCLcBGAs/s640/loopback.png)](https://1.bp.blogspot.com/-DMfVcMVHZNM/XUgRahyM_QI/AAAAAAAArJE/29h-7d-hPWYwjUipp_Kx5NnOYQFi4Y46gCLcBGAs/s1600/loopback.png)

[![](https://1.bp.blogspot.com/-3QvH2c2fwSM/XUgSNaKIxeI/AAAAAAAArJM/G2KOijI3paY-MhIdS-nFYFRg6bH6N_m-QCLcBGAs/s400/floating.png)](https://1.bp.blogspot.com/-3QvH2c2fwSM/XUgSNaKIxeI/AAAAAAAArJM/G2KOijI3paY-MhIdS-nFYFRg6bH6N_m-QCLcBGAs/s1600/floating.png)

*   Re-create the Load Balancer rule with DSR enabled, Enabling DSR causes the packet to be delivered to the destination VM with the original destination IP address intact. In our case this is the frontend IP of the ILB (10.2.1.100). With DSR disabled (the default), the packet delivered to the backend VM would have a destination IP address of the backend IP itself (10.2.1.4 in our case). Further reading:
    
    @ [https://docs.microsoft.com/en-us/azure/load-balancer/load-balancer-multivip-overview#rule-type-2-backend-port-reuse-by-using-floating-ip](https://docs.microsoft.com/en-us/azure/load-balancer/load-balancer-multivip-overview#rule-type-2-backend-port-reuse-by-using-floating-ip)

*   Configure loopback interface on backend VM with same IP address as ILB VIP (10.2.1.100)

[![](https://1.bp.blogspot.com/-wtUuxD7fKA4/XUgS52I-qRI/AAAAAAAArJU/Y6QKUMO6SC4nIvLhWfhJ1TKkGyJA0yKXgCEwYBhgL/s320/msloop.png)](https://1.bp.blogspot.com/-wtUuxD7fKA4/XUgS52I-qRI/AAAAAAAArJU/Y6QKUMO6SC4nIvLhWfhJ1TKkGyJA0yKXgCEwYBhgL/s1600/msloop.png)

  

*   Configure backend VM applicaton (IIS in our case) to listen on additional IP address. _NB. If using Windows Server 2016, enable weakhostsend on both the NIC and Loopback interface. See RFC1122._

[![](https://1.bp.blogspot.com/-JeZuii9pcZQ/XUgTKkicrhI/AAAAAAAArJc/PONMPXkMe30DBtzzYKKfmCHzsKAKzzFJgCLcBGAs/s400/iis-bind.png)](https://1.bp.blogspot.com/-JeZuii9pcZQ/XUgTKkicrhI/AAAAAAAArJc/PONMPXkMe30DBtzzYKKfmCHzsKAKzzFJgCLcBGAs/s1600/iis-bind.png)

  

  
```
Connectivity is now working externally from the on-premise VM using ILB VIP 10.2.1.100, with DSR enabled.

c:\\pstools>ipconfig | findstr /i "ipv4"

   IPv4 Address. . . . . . . . . . . : 192.168.2.1

  

c:\\pstools>psping -n 3 -i 0 -q 10.2.1.100:80

  

PsPing v2.10 - PsPing - ping, latency, bandwidth measurement utility

Copyright (C) 2012-2016 Mark Russinovich

Sysinternals - www.sysinternals.com

  

TCP connect to 10.2.1.100:80:

4 iterations (warmup 1) ping test: 100%

  

TCP connect statistics for 10.2.1.100:80:

  Sent = 3, Received = 3, Lost = 0 (0% loss),

  Minimum = 19.39ms, Maximum = 20.17ms, Average = 19.78ms

  

Connectivity from the web server itself is also working when accessing the service on 10.2.1.100, **as this exists locally on the server, aka on-link.**

c:\\pstools>ipconfig | findstr /i "ipv4"

   IPv4 Address. . . . . . . . . . . : 10.2.1.4

   IPv4 Address. . . . . . . . . . . : 10.2.1.100

  

c:\\pstools>route print 10.2.1.100

Active Routes:

Network Destination        Netmask          Gateway       Interface  Metric

       10.2.1.100  255.255.255.255         **On-link**        10.2.1.100    510

  

c:\\pstools>psping -n 3 -i 0 -q 10.2.1.100:80

  

PsPing v2.10 - PsPing - ping, latency, bandwidth measurement utility

Copyright (C) 2012-2016 Mark Russinovich

Sysinternals - www.sysinternals.com

  

TCP connect to 10.2.1.100:80:

4 iterations (warmup 1) ping test: 100%

  

TCP connect statistics for 10.2.1.100:80:

  Sent = 3, Received = 3, Lost = 0 (0% loss),

  Minimum = 0.11ms, Maximum = 0.20ms, Average = 0.14ms
```
  
  

There are three key call outs here:

1.  The backend call to the frontend VIP never leaves the backend VM. This may or may not suit your application requirements, the request can only be served locally.
2.  DSR is optional, but allows the backend VM to listen on a common IP (The ILB VIP) for all connecitons, locally originated and remote. 
3.  You must continue to listen on the physical primary NIC IP address for application connections, otherwise LB health probes will fail

### 3.3 Workaround Option \[3\] -Application Gateway / NVA

A simple option for HTTP/S traffic is to utilise Azure Application Gateway instead. Note, use of either APGW or an NVA has cost, performance and scale limitations as these are fundamentally different products. These solutions are based on additional compute resources that sit inline with the datapath, where as the Azure Load Balancer can be thought of more as a function of the Azure SDN.

  

![](https://1.bp.blogspot.com/-cj_uvDAZl50/XUgVkFxcZ7I/AAAAAAAArJo/2-rWEFSASWANF44csc5Dv8R3BnULqVa-QCLcBGAs/s640/apgw.png)

Application Gateway only supports HTTP/S frontend listeners, therefore, if a LB solution for other TCP/UDP ports is required an NVA (Network Virtual Appliance) is required. NGINX is one of these 3rd party NVA options.

  

![](https://1.bp.blogspot.com/-iy1Kr6NwM-0/XUgWD-CA9aI/AAAAAAAArJw/X0qO9MGNHCwBn6o-f7sDNPu746ajPN_NQCLcBGAs/s640/nginx.png)

See [https://github.com/jgmitter/nginx-vmss](https://github.com/jgmitter/nginx-vmss) for fast start NGINX config on Azure including ARM template. Also see https://github.com/microsoft/PL-DNS-Proxy for a similar NGINX template with ability to deploy to custom VNet.  
  

Two NGINX instances are used for high availability. Each instance contains the same proxy/LB configuration. These instances are fronted by an Azure internal load balancer themselves to provide a single front end IP address for client access. This front end IP also works for backend members specified in the NGINX config file, as shown on the diagram above.  
  

The simple NGINX proxy configuration is shown below.
```
upstream backend {

      server 10.2.1.4;

   }

  

   server {

      listen 80;

  

      location / {

          proxy\_pass http://backend;

      }

   }
```
  

The number of backend servers in my example is a single VM, in production there would be multiple nodes and additional lines within the upstream module. E.g.
```
upstream backend {

      server 10.2.1.4;

      server 10.2.1.5;

      server 10.2.1.6;

   }

  

   server {

      listen 80;

  

      location / {

          proxy\_pass http://backend;

      }

   }
   ```

# Contributing

This project welcomes contributions and suggestions.  Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit https://cla.opensource.microsoft.com.

When you submit a pull request, a CLA bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., status check, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.

# Legal Notices

Microsoft and any contributors grant you a license to the Microsoft documentation and other content
in this repository under the [Creative Commons Attribution 4.0 International Public License](https://creativecommons.org/licenses/by/4.0/legalcode),
see the [LICENSE](LICENSE) file, and grant you a license to any code in the repository under the [MIT License](https://opensource.org/licenses/MIT), see the
[LICENSE-CODE](LICENSE-CODE) file.

Microsoft, Windows, Microsoft Azure and/or other Microsoft products and services referenced in the documentation
may be either trademarks or registered trademarks of Microsoft in the United States and/or other countries.
The licenses for this project do not grant you rights to use any Microsoft names, logos, or trademarks.
Microsoft's general trademark guidelines can be found at http://go.microsoft.com/fwlink/?LinkID=254653.

Privacy information can be found at https://privacy.microsoft.com/en-us/

Microsoft and any contributors reserve all other rights, whether under their respective copyrights, patents,
or trademarks, whether by implication, estoppel or otherwise.
