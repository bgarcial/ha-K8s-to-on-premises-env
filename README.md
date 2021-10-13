# Designing a K8s environment

A system design approach for landing a resilient and highly available system based in Kubernetes and hybrid environment.

Look for an optimal solution when it comes to performance, security, scalability and availability.

## Requirements

The system to design is a Kubernetes cluster that has the following requirements:

- The cluster is running in Azure

- Authentication/authorization for interacting with the cluster is in place. 
There are 2 or more levels of authorization. A regular user that has access to 
a namespace and the admin user that can manage the cluster

- Needs to be deployed in a Vnet that can communicate with an on-premise network

- Needs to achieve the highest possible uptime for the api and the workers 
(calculate the SLA and explain it)

- Needs to be scalable on demand/based on load

- Needs to be able to run dozens of different workloads with different cpu/mem/disk requirements

- Needs a robus and highly available container registry

- Needs fixed ingress/egress ips to be able to whitelist this cluster ips from other systems

Based on the previous information, please provide a detailed solution on how would you architect 
the cluster explaining your decisions and the exact use of each component.

Keep in mind that you can add extra components to the cluster to satisfy the requirements.

---

## 1. Specifying the on-premises - Azure communication:

The [options](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/hybrid-networking/) 
Azure cloud platform presents for connecting an on-premises network to an Azure virtual
network are the following:

- VPN Connection
- Azure ExpressRoute connection
- ExpressRoute with VPN failover
- Hub-spoke network topology

I selected the [ExpressRoute with VPN failover](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/hybrid-networking/expressroute-vpn-failover)
option as the most suitable according to the security, high availability, performance, scalability and resilient
requirements are being demanded when thinking about this solution. Before to try to justify why I selected 
this architecture approach I will mention first briefly why I discarded the other options:

### **VPN Connection**, was discarded because: 

- Despite this alternative allows traffic between on-premises hardware and the cloud 
it is intended to be light, otherwise we have to be willing to get some more latency if we want some
processing power.

- Although Microsoft guarantees 99.9% availability for each VPN Gateway, this [SLA](https://azure.microsoft.com/support/legal/sla/vpn-gateway/) 
only covers the VPN gateway, and not the network connection to the gateway, so the performance here is not the desired.

- It requires a virtual network gateway, which is essentially two or more VMs that contains routing tables
and execute specific gateway services which determine how the virtual network will be used and the action to take.

Ok, here basically the idea about get in charge of the routing traffic and connectivity to a couple of VMs to allow the communication
is not something that sounds at some point limited when it comes to scalability and performance.

- The SKU plans for VPN gateway on Azure stack side, does not provide good vertical [scalability](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/hybrid-networking/vpn?tabs=portal#scalability-considerations).

- >For virtual networks that expect a large volume of VPN traffic, consider distributing the different 
workloads into separate smaller virtual networks and configuring a VPN gateway for each of them.

It means that require redesign of the Azure deployments I want to communicate if the traffic or demands of my
environment grows. Let's say I want to upload files to a pvc and execute some calculations with that information from
an application pod. 

- For [availability](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/hybrid-networking/vpn?tabs=portal#availability-considerations) is necessary to implement redundancy of the on-premises VPN Gateway 

Well, these were some of the reasons behind to discard it, but also VPN is considered more for connecting users
from one network to another, and not for supporting a HA distributed operation system, at least not under
kubernetes workload context plus on-premises to cloud communication included.

---

### **Express Route connection** is one of the candidates to work with.

This can be perfectly the option to pickup, it offers performance and scalability since this involves to
work with a connectivity provider having a dedicated channel. For sure it also make it more complex to setup. 
I discarded because there is a combination of its features with a VPN in a failover scheme, bringing this
option more availability rather than Express Route onlying. 

For [troubleshooting:](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/hybrid-networking/expressroute#troubleshooting)
>if a previously functioning ExpressRoute circuit now fails to connect, in the absence of any configuration 
changes on-premises or within your private VNet, you may need to contact the connectivity provider and work 
with them to correct the issue.

If we combine this with a VPN solution, we will get some time to solve this problem without downtime

However, if we make a balance, is not necessary to have this option accompanied for a VPN to choose it in our
propose, I mean this can be perfectly the way to go, is just that the next option offer more availability
I will explain their benefits in the next option:

---

### Express Route with a site-to-site virtual private network (VPN) as a failover connection.

This is the option I selected: 

Let's consider this option as similar to use Express Route alone, in terms of features, since the 
traffic flows between the on-premises network and the Azure VNet through an ExpressRoute connection. 
its main benefit is that if there is a loss of connectivity in the ExpressRoute circuit, 
traffic is routed through an IPSec VPN tunnel, that is why it provides more availability than Express Route connection

- >Note that if the ExpressRoute circuit is unavailable, the VPN route will only handle private peering connections. Public peering and Microsoft peering connections will pass over the Internet.
[More info](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/hybrid-networking/expressroute-vpn-failover)

Some of the reasons why I selected it:

- A dedicated private channel is created between on-premises network and Azure Vnet, via an Express Route circuit
and using a connectivity provider, so public Internet is not involved here. This allow a high bandwith path between
networks, and it can be increased ([the higher the bandwith, the greater the costs](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/hybrid-networking/expressroute#scalability-considerations))

- The numbers of Vnets destinations can be increased per Express Route circuit.

- For [security purposes](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/hybrid-networking/expressroute#security-considerations), 
this archiecture allows to add firewall between on-premises networks and provider edge routers (yes these kind of devices are needed on the customer side 
and the MSFT side, it is "a bit" elaborated). 
    -  Adding firewall capabilities helps to restrict unauthorized traffic from azure Vnets and allow
    to restrict access from resources within the Vnet to Internet. This can be work together with NSGs (inbound and outbond rules) on the VM
    scale sets that support the AKS cluster.
    - It also allows [these security features](https://docs.microsoft.com/en-us/security/benchmark/azure/baselines/expressroute-security-baseline?toc=%2fazure%2fexpressroute%2fTOC.json) 
    in terms of monitoring networks, alerts, MFA with AAD, analyzing logs patterns on express route and using Activity logs as a central place for diagnostic
    - All information in transit is encrypted at IP layer 3 level by using IPsec

- From [availability](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/hybrid-networking/expressroute#availability-considerations) 
perspective is opportune to describe that Express Route uses/create [private domain peerings](https://docs.microsoft.com/en-us/azure/expressroute/expressroute-circuit-peerings#routingdomains), 
those peerings are like a direct communication with either Azure cloud resources and MSFT cloud resources like Office , Dynamics, etc.

    - The fact that those peerings are created, it means they are considered like an extension of the on-premises
    network into Azure. 
    - Those peerings are configured in the router on premises and the MSFT routers using redundant sessions
    of Border Gateway Protocols per peering. 

- Consider that there are [two main plans](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/hybrid-networking/expressroute#azure-expressroute), if costs are some kind of factor decision: 
    - In the Metered Data plan, all inbound data transfer is free. All outbound data transfer is charged based on a pre-determined rate.
    - The Unlimited Data plan in which all inbound and outbound data transfer is free. 
    Users are charged a fixed monthly port fee based on high availability dual ports. these plans pricing approaches