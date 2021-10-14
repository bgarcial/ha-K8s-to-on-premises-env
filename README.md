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
is something that sounds at some point limited when it comes to scalability and performance.

- The SKU plans for VPN gateway on Azure stack side, does not provide good vertical [scalability](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/hybrid-networking/vpn?tabs=portal#scalability-considerations).   

- >For virtual networks that expect a large volume of VPN traffic, consider distributing the different 
workloads into separate smaller virtual networks and configuring a VPN gateway for each of them.

It means that require redesign of the Azure deployments I want to communicate if the traffic or demands of my
environment grows. 
Let's say I want to upload large files to a pvc by splitting the process and/or execute 
some calculations with that information from a backend application pod and publish the results
on a frontend application.

- For [availability](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/hybrid-networking/vpn?tabs=portal#availability-considerations) 
is necessary to implement redundancy of the on-premises VPN Gateway 

Well, these were some of the reasons behind to discard it, but also VPN is considered more for connecting users
from one network to another in order to access to services from an external place like if I were
at the destination place, and not for supporting a HA distributed operation system, at least not under
kubernetes workload context plus on-premises to cloud communication included.

---

### **Express Route connection** is one of the candidates to work with.

This can be perfectly the option to pickup, it offers performance and scalability since this involves to
work with a connectivity provider having a dedicated channel. For sure it also make it more complex to set it up. 

I discard it, because Azure also has another option that consists  in a combination 
of Express Route features with a VPN in a failover scheme, bringing this
option more availability rather than Express Route alone. 

- For [troubleshooting:](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/hybrid-networking/expressroute#troubleshooting)
    >if a previously functioning ExpressRoute circuit now fails to connect, in the absence of any configuration 
    changes on-premises or within your private VNet, you may need to contact the connectivity provider and work 
    with them to correct the issue.

    If we combine this with a VPN solution, we will get some time to solve this problem without downtime

**NOTE**:

However, if we make a balance, is not necessary to have this option accompanied for a VPN to choose it in our
propose, I mean this can be perfectly the way to go, is just that the next option offer more availability
I will explain their benefits in the next option:

---

### Express Route with a site-to-site virtual private network (VPN) as a failover connection.

**This is the option I selected**: 

Let's consider this option as similar to use Express Route alone, in terms of features, since the 
traffic flows between the on-premises network and the Azure VNet through an ExpressRoute connection. 
Its main benefit is that if there is a loss of connectivity in the ExpressRoute circuit, 
traffic is routed through an IPSec VPN tunnel, that is why it provides more availability than Express Route connection.

- >Note that if the ExpressRoute circuit is unavailable, the VPN route will only handle private peering connections. 
Public peering and Microsoft peering connections will pass over the Internet.

    This is another availability mechanism that can provide maximum uptime of the network connectivity
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
    - Those peerings and the vpn failover scheme are considered as a [disaster recovery approach]( https://docs.microsoft.com/en-us/azure/expressroute/designing-for-disaster-recovery-with-expressroute-privatepeering)

- Consider that there are [two main plans](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/hybrid-networking/expressroute#azure-expressroute), if costs are some kind of factor decision: 
    - In the Metered Data plan, all inbound data transfer is free. All outbound data transfer is charged based on a pre-determined rate.
    - The Unlimited Data plan in which all inbound and outbound data transfer is free. 
    Users are charged a fixed monthly port fee based on high availability dual ports. these plans pricing approaches

#### Some challenges to consider:

- We have to pick up a [telecommunications/connectivity provider to work with](https://docs.microsoft.com/en-us/azure/expressroute/expressroute-locations#partners) 
in order to set up the dedicated channel and in that point things like routing and NAT (if we have internal IPs on on-premises networks) become kind of blurred to me 
because that will depends on the provider.
    - For sure, the network team from company get into action here. 
- For VPN failover scheme, it requires redundant hardware (VPN appliances), and a redundant Azure VPN Gateway connection for which you pay charges.
- Requires to setup Local Edges routers on customer side to connect with the Router telco provider and the and the 
MSFT edge routers, creating in this way the Azure Router Express circuit that allow the communication on-premises to Azue cloud
    - There are [many factors to consider here](https://marckean.com/2018/09/03/azure-expressroute-demystified/) which ones goes beyond of the scope of this assigment since has to do
    with hardware and hard 1-3 layers networking knowledge.


---

### Hub Spoke Network Topology 

- It was discarded, since I see as cons fact that every communication from spoke vnets has to cross via Hub Vnet
to get on-premises network or even other vnets and that is not practical for the needs of performance and high 
availability, also it is a common single of failure if the hub gets down.

- Let's say many customers (dev teams or SRE/Infra teams at Tom Tom) have resources deployed in 
different environments (DTAP). If those resources need to share services like DNS, AAD, or K8s clusters, 
those services can be placed in the hub Vnet and every environment team will be a spoke Vnet, 
but not sure if we got the desired performance and HA here.

- I also saw we can use express route service in the hub Vnet, but the schema keeps forcing us to cross all 
the communications via Hub, that is why I see Express Route more suitable.

---

#### TO CLARIFY

Again, either Express Route and Express Route with VPN failover scheme are suitable options to
pickup for designing this HA environment based on a kubernetes cluster to allow on-premises to cloud 
connection. Any of those options can be selected according to demands of availability, being the Express Route with VPN failover which 
presents more availability. Having said that this option will be the focus from now on forward since the
assessment require to pickup **"<< the highest possible uptime and resilient system >>"**.


## 2. Architecting the AKS cluster and on-premises communication

![High level architecture deployment](https://cldup.com/jPECdvfyVA.png)

The above diagram presents a high level view of the deploymnent in order to allow a
communication between on-premises network and the Vnet where AKS cluster is.

### Detailed Architecture Deployment



![Detailing Express Route communication](https://cldup.com/aZPme3sfwM.png)

---

## 3. Deeping dive in Kubernetes deployment

- RBAC enabled
    - Specify the Roles and ClusterRoles to allow see a namespace and admin user on the cluster
    with aad
- Autoscaling
- Virtual Nodes
- K8s network policies
    - Check how involves here the egress traffic.
    While whitelisting is generally bad option, we would like to have teams pick 
    option to use network policies.  
    What I can mentioned here is enable calico network policy feature on aks, and describe
    how will be the behavior for restricting egress traffic
    https://docs.projectcalico.org/about/about-kubernetes-egress 
    Lets assume that we do not need for this usecase. 

- AAD integration
- ACR multiregion? We are only one region, so we don't need a high available  ACR
- AKS in three or 4 availability zones, specify this in a separate diagram.
- HPA POD autoscaling with metrics collections by deploying prometheus for cpu/mem/disks
and perhaps using custom metrics like requests counts of the app pods

---

## 4. - needs to achieve the highest possible uptime for the api and the workers (calculate the SLA and explain it)

- needs to achieve the highest possible uptime for the api and the workers (calculate the SLA and explain it)
