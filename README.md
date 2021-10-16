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

#### Express Route variants - To Clarify / consider 

I consider opportune to highlight some Express Route implementation variants.
Again, either Express Route and Express Route with VPN failover scheme are suitable options to
pickup for designing this HA environment based on a kubernetes cluster to allow on-premises to cloud 
connection.

In addition ER offers a third option, the possibility of achieve high availability 
[by creating various circuits](https://docs.microsoft.com/en-us/azure/expressroute/expressroute-faqs#how-do-i-ensure-high-availability-on-a-virtual-network-connected-to-expressroute)
which ones will be operating in the communication between the Vnet and the on-premises network(s):

>You can achieve high availability by connecting up to 4 ExpressRoute circuits in the same peering location 
to your virtual network, or by connecting up to 16 ExpressRoute circuits in different peering locations 
(for example, Singapore, Singapore2) to your virtual network. If one ExpressRoute circuit goes down, 
connectivity will fail over to another ExpressRoute circuit ...

So we even can achieve HA without set up a VPN. Any of those options can be selected according to demands of availability. 
I selected Express Route with VPN failover as a solution that in addition to connectivity brings a VPN, which in most cases
is also useful when hybrid communications takes place.


## 2. Architecting the AKS cluster and on-premises communication

The following diagram presents a high level view of the deployment in order to allow a
communication between on-premises network and the Vnet where AKS cluster is.

![High level architecture deployment](https://cldup.com/jPECdvfyVA.png)


### 2.1 Detailing Express Route communication and K8s RBAC namespace-cluster restrictions

![Detailing Express Route communication](https://cldup.com/QP4NfAEqgg.png)


Regarding access control from users to the kubernetes cluster, as the number of cluster nodes, 
applications, and team members increases, we might want to limit the resources the team members 
and applications can access, as well as the actions they can take. 
In order a regular user can get access only to a `dev` namespace and other user get the admin cluster role, the Kubernetes
cluster should be created with RBAC enabled in order to work with `Role`, `ClusterRole`, `RoleBinding`
and `ClusterRoleBinding` resources. 

In addition we will take in advance of the [Azure Active Directory Integration](https://docs.microsoft.com/en-us/azure/aks/managed-aad) to use 
AAD as identity provider such as follow:

- Keeping in mind future users to be added, we will use an AAD group membership 
approach, where:

- Two AAD groups will be created: 
    - **`Namespace-DEV`**:
        ```
        > NAMESPACE_DEV_ID_AAD_GROUP=$(az ad group create --display-name Namespace-DEV --mail-nickname appdev --query objectId -o tsv)
        ```
        So we got the AAD groupId:
        ```
        echo $NAMESPACE_DEV_ID_AAD_GROUP
        e5b4617b-b318-47f2-90fa-ea06f5964d20
        ```

    - **`AKS-Admin-Users`**:
        ```
        > AKS_ADMIN_USERS_ID_AAD_GROUP=$(az ad group create --display-name AKS-Admin-Users --mail-nickname appdev --query objectId -o tsv)
        ```
        We got the AAD groupId
        ```
        > echo $AKS_ADMIN_USERS_ID_AAD_GROUP
        c007d29b-e404-4b54-b37e-c2ca955f38ac
        ```
    If we go to Azure portal we can see the groups created:
    ![](https://cldup.com/Gm8iHzXjUH.png)

- We get the AKS ID cluster:
  ```
  > AKS_ID=$(az aks show \
    --resource-group test-aks \
    --name test \
    --query id -o tsv)
  ```  
  Basically this is the Resource ID of the cluster which can also be gotten on azure portal.
  ```
  > echo $AKS_ID
  /subscriptions/9148bd11-f32b-4b5d-a6c0-5ac5317f29ca/resourcegroups/test-aks/providers/Microsoft.ContainerService/managedClusters/test
  ```

- Create a role assignment for the `Namespace-DEV` AAD group, to let any member of the group
use `kubectl` to interact with the cluster by granting them the `Azure Kubernetes Service Cluster User Role`
    ```
    > az role assignment create \
        --assignee $NAMESPACE_DEV_ID_AAD_GROUP \
        --role "Azure Kubernetes Service Cluster User Role"  \
        --scope $AKS_ID
        {
            "canDelegate": null,
            "condition": null,
            "conditionVersion": null,
            "description": null,
            "id": "/subscriptions/9148bd11-f32b-4b5d-a6c0-5ac5317f29ca/resourcegroups/test-aks/providers/Microsoft.ContainerService/managedClusters/test/providers/Microsoft.Authorization/roleAssignments/b40b05fa-ebcc-4138-8ca2-02143c88bf9c",
            "name": "b40b05fa-ebcc-4138-8ca2-02143c88bf9c",
            "principalId": "e5b4617b-b318-47f2-90fa-ea06f5964d20",
            "principalType": "Group",
            "resourceGroup": "test-aks",
            "roleDefinitionId": "/subscriptions/9148bd11-f32b-4b5d-a6c0-5ac5317f29ca/providers/Microsoft.Authorization/roleDefinitions/4abbcc35-e782-43d8-92c5-2d3f1bd2253f",
            "scope": "/subscriptions/9148bd11-f32b-4b5d-a6c0-5ac5317f29ca/resourcegroups/test-aks/providers/Microsoft.ContainerService/managedClusters/test",
            "type": "Microsoft.Authorization/roleAssignments"
        }
    ```
    If we go to azure portal we can see the role created for this AAD group
    ![](https://cldup.com/SpqQBI8ve4.png)

- Create a role assignment for the `AKS-Admin-Users` AAD group, to let any member of the group
use `kubectl` to interact with the cluster by granting them the `Azure Kubernetes Service RBAC Cluster Admin `
    ```
    > az role assignment create \
        --assignee $AKS_ADMIN_USERS_ID_AAD_GROUP \
        --role "Azure Kubernetes Service RBAC Cluster Admin"  \
        --scope $AKS_ID
        {
            "canDelegate": null,
            "condition": null,
            "conditionVersion": null,
            "description": null,
            "id": "/subscriptions/9148bd11-f32b-4b5d-a6c0-5ac5317f29ca/resourcegroups/test-aks/providers/Microsoft.ContainerService/managedClusters/test/providers/Microsoft.Authorization/roleAssignments/18a7021c-2799-4de2-a4d2-e4e5842977b0",
            "name": "18a7021c-2799-4de2-a4d2-e4e5842977b0",
            "principalId": "c007d29b-e404-4b54-b37e-c2ca955f38ac",
            "principalType": "Group",
            "resourceGroup": "test-aks",
            "roleDefinitionId": "/subscriptions/9148bd11-f32b-4b5d-a6c0-5ac5317f29ca/providers/Microsoft.Authorization/roleDefinitions/b1ff04bb-8a4e-4dc4-8eb5-8693973ce19b",
            "scope": "/subscriptions/9148bd11-f32b-4b5d-a6c0-5ac5317f29ca/resourcegroups/test-aks/providers/Microsoft.ContainerService/managedClusters/test",
            "type": "Microsoft.Authorization/roleAssignments"
        }
    ```
    If we go to Azure portal we can see the role:
    ![](https://cldup.com/tvKyh1Vu7l.png)

- Creating AAD Users to be used to sign in to the AKS cluster:
    - Creating AKS Dev (`devuser@bgarcialoutlook.onmicrosoft.com`) user: 
        ```
        > echo "Please enter the UPN for application developers: " && read AAD_DEV_UPN
        Please enter the UPN for application developers:
        devuser@bgarcialoutlook.onmicrosoft.com
        ```
        ```
        > echo "Please enter the secure password for application developers: " && read AAD_DEV_PW
        Please enter the secure password for application developers:
        mypasswd2021*
        ```
        ```
        DEV_USER_ID=$(az ad user create \
            --display-name "AKS Dev" \
            --user-principal-name $AAD_DEV_UPN \
            --password $AAD_DEV_PW \
            --query objectId -o tsv)
        ```
        ```
        > echo $DEV_USER_ID
        877d014f-cddb-4b2a-ad4f-2f0425b465bd
        ```

    - Adding AKS Dev (`devuser@bgarcialoutlook.onmicrosoft.com`) user to `Namespace-DEV` AAD group
        ```
        > az ad group member add --group Namespace-DEV --member-id $DEV_USER_ID
        ```
        The user was added to the AAD group
        ![](https://cldup.com/WcELfJa7UK.png)
    
    
    - Creating Ops (`opsuser@bgarcialoutlook.onmicrosoft.com`) user:
        ```
        > echo "Please enter the UPN for SREs: " && read AAD_OPS_UPN
        Please enter the UPN for OPS:
        opsuser@bgarcialoutlook.onmicrosoft.com
        ```
        ```
        > echo "Please enter the secure password for SREs: " && read AAD_OPS_PW
        Please enter the secure password for SREs:
        mypassword2021*
        ```
        ```
        > # Create a user for the SRE role
            OPS_USER_ID=$(az ad user create \
            --display-name "AKS Ops" \
            --user-principal-name $AAD_OPS_UPN \
            --password $AAD_OPS_PW \
            --query objectId -o tsv)
        ```
        ```
        > echo $OPS_USER_ID
        32d390c2-0b59-4ecb-bd8a-55621272ce16
        ```
    - Adding Ops (`opsuser@bgarcialoutlook.onmicrosoft.com`) user to `AKS-Admin-Users` AAD group:
        ```
        > az ad group member add --group AKS-Admin-Users --member-id $OPS_USER_ID
        ```
- **Testing the access to the cluster resources:**    
    - Get the cluster admin credentials:
        ```
        # This command will use the credentials of your normal daily basis work user on Azure (no the above created)
        az aks get-credentials --resource-group myResourceGroup --name myAKSCluster --admin
        ``` 
    - Create `dev` namespace
        ```
        > k create ns dev
        + kubectl create ns dev
        namespace/dev created
        ```
        ```
        > kgnsall
        + kubectl get namespaces --all-namespaces
        NAME              STATUS   AGE
        calico-system     Active   3h3m
        default           Active   3h4m
        dev               Active   7s
        kube-node-lease   Active   3h4m
        kube-public       Active   3h4m
        kube-system       Active   3h4m
        tigera-operator   Active   3h4m
        ```
    - Now create a `Role` for `dev` namespace: It is the Role which will allow access only to `dev` ns.
      
        ```
        # role-dev-namespace.yaml
        kind: Role
        apiVersion: rbac.authorization.k8s.io/v1
        metadata:
          name: dev-user-full-access
          namespace: dev
        rules:
        - apiGroups: ["", "extensions", "apps"]
          resources: ["*"]
          verbs: ["*"]
        - apiGroups: ["batch"]
          resources:
          - jobs
          - cronjobs
          verbs: ["*"]
        ```
        ```
        > ka role-dev-namespace.yaml
        + kubectl apply --recursive -f role-dev-namespace.yaml
        role.rbac.authorization.k8s.io/dev-user-full-access created   
        ```
        So far we are adding all the verbs or actions (`*`).  If you want to assign just `read-write` access consider to use:
        ```
        verbs:
        - get
        - list
        - watch
        - create
        - update
        - patch
        - delete
        ```
    - Now, create a `RoleBinding` to associate the previous `Role` to the `Namespace-DEV` AAD group.
    So first get the `Namespace-DEV` AAD groupObjectId:
        ```
        > az ad group show --group Namespace-DEV --query objectId -o tsv
        e5b4617b-b318-47f2-90fa-ea06f5964d20
        ```
    - Create the `RoleBinding` and use that groupObjectId

        ```
        kind: RoleBinding
        apiVersion: rbac.authorization.k8s.io/v1
        metadata:
          name: dev-user-access
          namespace: dev
        roleRef:
          apiGroup: rbac.authorization.k8s.io
          kind: Role
          name: dev-user-full-access
        subjects:
        - kind: Group
          namespace: dev
          name: e5b4617b-b318-47f2-90fa-ea06f5964d20 # groupObjectId      
        ```
        ```
        > ka rolebinding-dev-namespace.yaml
        + kubectl apply --recursive -f rolebinding-dev-namespace.yaml
        rolebinding.rbac.authorization.k8s.io/dev-user-access created
        ```
    - Now create a `ClusterRole`: It is the ClusterRole which will allow access to the entire cluster.
        ```
        kind: ClusterRole
        apiVersion: rbac.authorization.k8s.io/v1
        metadata:
          name: ops-user-full-access
        rules:
        - apiGroups: ["", "extensions", "apps"]
          resources: ["*"]
          verbs: ["*"]
        - apiGroups: ["batch"]
          resources:
        - jobs
        - cronjobs
        verbs: ["*"]
        ```
        ```
        > ka cluster-role-ops.yaml
        + kubectl apply --recursive -f cluster-role-ops.yaml
        clusterrole.rbac.authorization.k8s.io/ops-user-full-access created
        ```
    - Now, create a `ClusterRoleBinding` to associate the previous `ClusterRole` to the `AKS-Admin-Users` AAD group.
    So first get the `AKS-Admin-Users` AAD groupObjectId:
        ```
        > az ad group show --group AKS-Admin-Users --query objectId -o tsv
        c007d29b-e404-4b54-b37e-c2ca955f38ac
        ```
    - Create the `ClusterRoleBinding` and use that groupObjectId
        ```
        apiVersion: rbac.authorization.k8s.io/v1
        kind: ClusterRoleBinding
        metadata:
          name: ops-user-full-access-binding
        roleRef:
          kind: ClusterRole
          name: ops-user-full-access
          apiGroup: rbac.authorization.k8s.io
        subjects:
        - kind: Group
          name: c007d29b-e404-4b54-b37e-c2ca955f38ac # groupObjectId
          apiGroup: rbac.authorization.k8s.io
        ```
        ```
        > ka cluster-role-binding-ops.yaml
        + kubectl apply --recursive -f cluster-role-binding-ops.yaml
        clusterrolebinding.rbac.authorization.k8s.io/ops-user-full-access-binding created
        ```
- **Interacting with the cluster using AKS Dev (`devuser@bgarcialoutlook.onmicrosoft.com`) identity:**
  
  Let's remember this user only has access to `dev` namespace
  
  - Reset the credentials of the cluster 
    ```
    > az aks get-credentials --resource-group test-aks --name test --overwrite-existing
    The behavior of this command has been altered by the following extension: aks-preview
    Merged "test" as current context in /Users/bgarcial/.kube/config
    ```
  - Create an NGINX pod on `dev` ns. The cluster will require the AAD credentials
    ```
    > kubectl run nginx-dev --image=mcr.microsoft.com/oss/nginx/nginx:1.15.5-alpine --namespace dev
    + kubectl run nginx-dev --image=mcr.microsoft.com/oss/nginx/nginx:1.15.5-alpine --namespace dev
    To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code DAZ9UTXEE to authenticate.
    ```
    Enter the code and check:
    ![](https://cldup.com/CUZzrKaDPM.png)
    ![](https://cldup.com/Wzmc2o-7TP.png)
    
    The pod was created.
    ```
    > kubectl run nginx-dev --image=mcr.microsoft.com/oss/nginx/nginx:1.15.5-alpine --namespace dev
    + kubectl run nginx-dev --image=mcr.microsoft.com/oss/nginx/nginx:1.15.5-alpine --namespace dev
    pod/nginx-dev created
    ```
  - List `kube-system` namespace resources. It shouldn't be possible.
    ```
    > k get pods,deploy,svc  -n kube-system
    + kubectl get pods,deploy,svc -n kube-system
    Error from server (Forbidden): pods is forbidden: User "devuser@bgarcialoutlook.onmicrosoft.com" cannot list resource "pods" in API group "" in the namespace "kube-system"
    Error from server (Forbidden): deployments.apps is forbidden: User "devuser@bgarcialoutlook.onmicrosoft.com" cannot list resource "deployments" in API group "apps" in the namespace "kube-system"
    Error from server (Forbidden): services is forbidden: User "devuser@bgarcialoutlook.onmicrosoft.com" cannot list resource "services" in API group "" in the namespace "kube-system"
    ```

- **Interacting with the cluster using AKS Ops (`opsuser@bgarcialoutlook.onmicrosoft.com`) identity:**
  
  Let's remember this user has access to the entire cluster
  
  - Reset the credentials of the cluster 
    ```
    > az aks get-credentials --resource-group test-aks --name test --overwrite-existing
    The behavior of this command has been altered by the following extension: aks-preview
    Merged "test" as current context in /Users/bgarcial/.kube/config
    ```
    - List `kube-system` ns:
        ```
        > k get pods,deploy,svc  -n kube-system
        + kubectl get pods,deploy,svc -n kube-system
        To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code D7N7ZVD5G to authenticate.
        ```
        ![](https://cldup.com/yx5p7x-sie.png)
        ![](https://cldup.com/_7F-2Bu8kJ.png)

        We can see all the resources requested were listed:
        ```
        > k get pods,deploy,svc  -n kube-system
        + kubectl get pods,deploy,svc -n kube-system
        To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code D7N7ZVD5G to authenticate.
        NAME                                       READY   STATUS    RESTARTS   AGE
        pod/aci-connector-linux-7dc49975b4-bzvw7   1/1     Running   3          4h18m
        pod/azure-ip-masq-agent-4rj4q              1/1     Running   0          4h17m
        pod/azure-ip-masq-agent-9b9vf              1/1     Running   0          4h18m
        pod/azure-ip-masq-agent-vkltr              1/1     Running   0          4h18m
        pod/coredns-autoscaler-54d55c8b75-v26sr    1/1     Running   0          4h18m
        pod/coredns-d4866bcb7-6p245                1/1     Running   0          4h16m
        pod/coredns-d4866bcb7-d6qsr                1/1     Running   0          4h16m
        pod/coredns-d4866bcb7-fmcnz                1/1     Running   0          4h17m
        pod/coredns-d4866bcb7-mhdct                1/1     Running   0          4h18m
        pod/coredns-d4866bcb7-vfm87                1/1     Running   0          4h16m
        pod/kube-proxy-j8csv                       1/1     Running   0          4h18m
        pod/kube-proxy-sdgzk                       1/1     Running   0          4h17m
        pod/kube-proxy-t7phr                       1/1     Running   0          4h18m
        pod/metrics-server-569f6547dd-kwlrp        1/1     Running   0          4h18m
        pod/omsagent-dvndb                         2/2     Running   0          4h18m
        pod/omsagent-qf9pq                         2/2     Running   0          4h18m
        pod/omsagent-rn94f                         2/2     Running   0          4h17m
        pod/omsagent-rs-6cdd976f5c-d85pc           1/1     Running   0          4h18m
        pod/tunnelfront-576dfb478f-4tmng           1/1     Running   0          4h18m

        NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
        deployment.apps/aci-connector-linux   1/1     1            1           4h18m
        deployment.apps/coredns               5/5     5            5           4h18m
        deployment.apps/coredns-autoscaler    1/1     1            1           4h18m
        deployment.apps/metrics-server        1/1     1            1           4h18m
        deployment.apps/omsagent-rs           1/1     1            1           4h18m
        deployment.apps/tunnelfront           1/1     1            1           4h18m

        NAME                                     TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)         AGE
        service/healthmodel-replicaset-service   ClusterIP   10.0.95.152   <none>        25227/TCP       4h18m
        service/kube-dns                         ClusterIP   10.0.0.10     <none>        53/UDP,53/TCP   4h18m
        service/metrics-server                   ClusterIP   10.0.76.27    <none>        443/TCP         4h18m
        ```

        - Even we can check `dev` ns:
        ```
        > k get pods,deploy,svc  -n dev
        + kubectl get pods,deploy,svc -n dev
        NAME            READY   STATUS    RESTARTS   AGE
        pod/nginx-dev   1/1     Running   0          8m26s
        ```









---

## 3. Deeping dive in Kubernetes deployment

### 3.1

### 3.2. - needs to achieve the highest possible uptime for the api and the workers (calculate the SLA and explain it)

kEEP nin mind the end to end solution, not just the aks. I did it with the Express Route circuit,
consider dependecy of aks with AAD. MAYBE i AM using a db for a stateless app. 
Those SLAS should be the same or greater than AKS
- AZ, in a region?
    - deploy to at least two regions
I want that level of resiliency in case of a natural disaster, i am not going to impact the other copy of my data

I wanted to propose a Multiregion deployment with two clusters in westeurope and northeurope and use
traffic manager, that is possible, but since we are using Express Route, I think is not possible
to put them to work all together. 
Indeed is possible use Express Route to redirect or receive traffic from Vnets located in 
different regions ([by implementing circuits in each of them](https://docs.microsoft.com/en-us/azure/expressroute/expressroute-optimize-routing#suboptimal-routing-between-virtual-networks)) 
being Traffic Manager a DNS LB and makes use of endpoints to be exposed in order to contact a target 
to be redirected, and being Express Route a technology that demands involves a Gateway in the cloud Vnet target 
is not clear for me how to put them to talk each other under networking protocol layer perspective.


I will have in some how expose an endpoint of express route gateway or circuit in order
Traffic manager can redirect a connection attempt from onpremises network. Being Traffic Manager operating
under layer 7 application and Express Route between 2-5 layers, I thonk is not possible to work with both
In addition, i didn't find something similar in the documentation.
That is why I decided to use just one cluster with 3 availability zones across westeurope region and the following features
Maybe express route global can helps here toconnect to more than 1 on premises network
https://docs.microsoft.com/en-us/azure/expressroute/expressroute-faqs#i-have-more-than-two-on-premises-networks-each-connected-to-an-expressroute-circuit-can-i-enable-expressroute-global-reach-to-connect-all-of-my-on-premises-networks-together

![](https://cldup.com/y7v0IZCyJ5.png)


### 3.3 Other K8s features

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
    - Maybe put it across the availability zones that the deployment will have
- AKS in three or 4 availability zones, specify this in a separate diagram.
- HPA POD autoscaling with metrics collections by deploying prometheus for cpu/mem/disks
and perhaps using custom metrics like requests counts of the app pods

---
