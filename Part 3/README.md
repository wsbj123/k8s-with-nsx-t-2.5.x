# NSX-T 2.5.x & K8S  - PART 3
[Home Page](https://github.com/dumlutimuralp/k8s-with-nsx-t-2.5.x)

# Table of Contents
[Configuring NSX Objects for K8S](#Configuring-NSX-Objects-for-K8S)  
[NSX Downloadables for K8S](#NSX-Downloadables-for-K8S)  
[Content of Manifest File](#Content-of-Manifest-File)

# Configuring NSX Objects for K8S
[Back to Table of Contents](https://github.com/dumlutimuralp/k8s-with-nsx-t-2.5.x/tree/master/Part%203#Table-of-Contents)

## Tagging the Segment Ports for K8S Nodes

In this section the ports of the "K8S-NodeTransport" segment, to which the Kubernetes Nodes second vNIC is connected to, will be configured with specific scopes as "ncp/cluster" , "ncp/node_name" and with specific tags as "k8s-cluster" and K8S Node hostname.

Navigate to "Networking -> Segments" in the NSX-T Simplified UI and then CLICK three dots on the left of the  "K8S-NodeTransport" segment and then click edit in the pop up menu. Click on the right arrow next to where it says "Ports" and then click on the number "3" on the right to edit the tagging of the ports. 

![](2019-12-18_13-25-54.jpg)

When the ports screen come up then click on three dots on the left of the port on top, and then click on edit, this will bring up the settings screen for that segment port.

![](2019-12-18_13-31-39.jpg)

Configure the tag and scope as shown below.

![](2019-12-18_13-34-39.jpg)

![](2019-12-18_13-44-34.jpg)

Repeat the steps for the remaining nodes. the nodename should match the tag for the scope "ncp/node_name".

![](2019-12-18_13-49-10.jpg)

Note#1 : The tag should match the node name which should match the hostname of the Ubuntu node, since that specific hostname will be used by Kubernetes as the node name.

Note#2 : If the admin changes the Kubernetes node name, then the tag ncp/node_name should also be updated and NCP needs to be restarted. Once Kubernetes is installed, "kubectl get nodes" can be used to provide the node names in the output. The tags should be added to the segment port before that node gets added to the K8S cluster by using the "kubeadm join" command. Otherwise, the K8S Pods on the new node will not have network connectivity. In case of tags being incorrect or missing, then to fix the issue, correct tags should be applied and NCP should be restarted.

## Configuring IP Address Pools and IP Address Blocks

Navigate to "Networking -> IP Address Pools -> IP Address Pools (again)" and then configure an IP address pool "K8S-LB-POOL" as following; this pool will be used for the IP address assignment of the K8S Ingress or for each service of K8S Service Type LoadBalancer.

![](2019-12-18_14-10-50.jpg)

Navigate to "Networking -> IP Address Pools -> IP Address Pools (again)" and then configure an IP address pool "K8S-NAT-POOL" as following; this pool will be used for source NATing the K8S Pods in the namespaces. For each namespace an individual SNAT IP will be picked from this pool. 

![](2019-12-18_14-26-55.jpg)

Navigate to "Networking -> IP Address Pools -> IP Address Blocks" and then configure the two IP address blocks shown below. 

![](2019-12-18_14-15-39.jpg)

"K8S-POD-IP-BLOCK" is provisioned as a /16 subnet ("K8S-POD-IP-BLOCK" - 172.31.0.0/16). Whenever a developer creates a new K8S namespace, then this IP Block will be carved out by /24 chunk and a /24 subnet based IP pool will be created in NSX-T automatically. That /24 subnet based IP pool will be assigned to the respective namespace and whenever Pods are created in that namespace, then each POD will pick an IP address from that /24 subnet. (Each namespace which gets an IP Pool out of this "K8S-POD-IP-BLOCK" will be Source NATed to an IP address which gets picked from the "K8S-NAT-POOL" configured earlier)

"K8S-NOSNAT-IP-BLOCK" is also provisioned as a /16 subnet ("K8S-POD-IP-BLOCK" - 10.192.0.0/16) . As the name suggests this IP Block is used for namespaces which will NOT be source NATed. Whenever a developer creates a new K8S namespace with the K8S annonation of " ncp/no_snat: "true" " in the manifest (yaml file), then this IP Block will be used and then this IP Block will be carved out by /24 chunk and a /24 subnet based IP pool will be created in NSX-T automatically. That /24 subnet based IP pool will be assigned to the respective namespace and whenever Pods are created in that namespace, then each POD will pick an IP address from that /24 subnet.

<b>Note :</b> NSX-T supports configuring a persistent SNAT IP per K8S namespace or per K8S service by using native K8S annotations. This provides granular operations for Pods to access a physical database for instance. This is not covered in this demonstration.

## Configuring Firewall Sections

Two new sections will be configured in the NSX-T distributed firewall rule base. Any K8S related firewall rule will be configured between these sections. 

Navigate to "Security -> East-West Security Distributed Firewall -> Category Specific Rules -> Environment" and then click on "+ Add Policy " and then give it a name like "K8S-Begin". Configure one more section by clicking on "+ Add Policy" and then give it a name like "K8S-End". Drag and drop as needed to make sure these sections show up as below. The K8S network policy driven firewall rules will land in between these sections.

![](2019-12-18_14-41-28.jpg)

## Capturing the NSX-T Object Names 

In this step the object UUID of the NSX-T objects which will be consumed by K8S need to be captured and noted. For that what needs to be done is <b>clicking on the three dots on the left of the respective object in the NSX-T UI and select "Copy path to clipboard". </b> This action copies the whole API path of the object in which the object UUID can be found.

Let' s do this for Tier 0 Gateway and also for one of the IP address pools.

![](2019-12-18_14-49-56.jpg)

<b>When "Copy path to clipboard" is selected and the content is pasted to a notepad file , in this lab the output appears like this -> "/infra/tier-0s/T0-OCP-K8S". This means that object UUID is "T0-OCP-K8S".</b> Note that with Policy API, the UUID is the object name itself in most cases. 

<b>Important Note :</b> If the NSX-T object is renamed later on, the object UUID will still stay the same as specified in the object creation time. Hence double checking the UUID as shown above is an important task.

Repeat the above step for all the IP address pools, IP address blocks, firewall sections and make a note of the object UUID, since the data will be used in the NCP config (For firewall section the UUID will be a long string of characters and numbers) 


# NSX Downloadables for K8S
[Back to Table of Contents](https://github.com/dumlutimuralp/k8s-with-nsx-t-2.5.x/tree/master/Part%203#Table-of-Contents)

NSX Container Plug-in for Kubernetes and Cloud Foundry - Installation and Administration Guide, published [here](https://docs.vmware.com/en/VMware-NSX-T-Data-Center/2.5/ncp-kubernetes/GUID-FB641321-319D-41DC-9D16-37D6BA0BC0DE.html), guides the user on the installation steps of the NSX components on a K8S cluster. In this article a more simplified and explanatory view is provided.

Kubernetes related NSX files can be downloaded from NSX-T download page at my.vmware.com (shown below)

![](2019-12-16_17-38-09.jpg)

When "Go To Downloads" is clicked the details of the content can be reviewed in the following page (below)

![](2019-12-16_17-43-07.jpg)

Once the .zip file is downloaded and extracted the content is as below.

![](2019-12-16_22-06-35.jpg)

The breakdown of the content is following :

* Kubernetes : This folder contains the unified manifest file (Yaml) and container image (.tar) for RHEL and Ubuntu operating systems. The same container image is used for NSX Node Agent, NSX Kube Proxy, NSX OVS and NSX Dummy containers. (Note that there is a container image also for Photon operating system which is used on nodes in Pivotal Application Services (PAS) solution)

* Openshift : This folder contains the unified manifest file (.yaml) for Openshift baremetal nodes andd Openshift virtual machine nodes.

* Open vSwitch : This folder contains the Open vSwitch package for various operating systems. 

* PAS : This folder contains the Pivotal Application Services (PAS) specific installation file. 

* Scripts : As the name suggests this folder contains the scripts to clean all the K8S related objects in NSX-T objecct database in a policy API based or management API based NSX-T environment in the event of a K8S cluster decommission process.

Since in this environment Ubuntu is the operating system for K8S nodes, "nsx-ncp-ubuntu-2.5.0.14628220.tar" and "ncp-ubuntu.yaml" will be used. 

Let' s look at the manfiest file "ncp-ubuntu.yaml" in more detail now.


# Content of Manifest File
[Back to Table of Contents](https://github.com/dumlutimuralp/k8s-with-nsx-t-2.5.x/tree/master/Part%203#Table-of-Contents)

As mentioned in [Part 2](https://github.com/dumlutimuralp/k8s-with-nsx-t-2.5.x/blob/master/Part%202/README.md) the manifest file in NCP version 2.5 includes all NSX related definitions and parameters as following:  
- nsx-system namespace specific service account, cluster role, 
- NCP deployment, 
- NSX NCP Bootsratp daemonset, 
- NSX Node Agent daemonset and the respective NCP configmap and NSX Node Agent configmaps

## Edited Parameters in Configmaps

Open up the "ncp-ubuntu.yaml" file with a text editor and follow the guidelines provided below to addd/change the parameters in the file.

### Configmap for NCP

The following <b>three</b> parameters are <b>MANUALLY ADDED</b> in the <b>[nsx_v3]</b> sub-section of the "Configmap for ncp.ini" section of the "ncp-ubuntu.yaml" file. These parameters are mentioned in the official installation document of NCP for K8S.

<b>policy_nsxapi = True :</b> Used to define that NCP should use NSX-T policy API to create objects in NSX-T. 

<b>single_tier_topology = True :</b> This parameter basically defines that Tier1 Gateway level consumption will be used. Meaning that all the K8S cluster related SNAT and Load Balancer configuration will take place on a Tier 1 Gateway.

<b>tier0_gateway = T0-OCP-K8S</b> : Used to define to which T0 gateway the new K8S related Tier 1 gateway (for SNAT and LB) should be connected to.

The existing parameters in the manifest file, which are used in this demonstration, has "#" removed in each line. Below is the list and explanation of each.

<b>nsx_api_managers = 10.190.1.80</b> , <b>nsx_api_user = admin</b> , <b>nsx_api_password = XXXXXXXXXXXXXX</b> : These parameters are for NCP to access/consume the NSX Manager.

<b>insecure = True :</b> NSX Manager server certificate is not verified.

<b>subnet_prefix = 24 :</b> The size of the IP Pools for the namespaces that will be carved out from the main "K8S-POD-IP-BLOCK". Whenever a new K8S namespace is created a /24 IP pool will be allocated from that IP block.

<b>use_native_loadbalancer = True :</b> This setting is to use NSX-T load balancer for K8S Service Type : Load Balancer. Whenever a new K8S service is exposed with the Type : Load Balancer then a VIP will be provisioned on NSX-T LB attached to a Tier 1 Gateway dedicated for LB function.

<b>l4_lb_auto_scaling = True :</b> When the scalability limits of the NSX-T Load Balancer is met then additional NSX-T load balancer instances (meaning additional Tier 1 Gateways) will be automatially provisioned.

<b>default_ingress_class_nsx = True :</b> When the developer does not use ingress class annotation in his/her manifest then the default K8S ingress (which means Layer 7 LB) will be NSX-T Load Balancer.

<b>service_size = SMALL :</b> The size of the NSX-T Load Balancer, that will be instantiated and associated with the Tier 1 Gateway automatially.

<b>container_ip_blocks = K8S-POD-IP-BLOCK :</b> This setting defines from which IP block each K8S namespace will carve its IP Pool/IP address space from. Size of each K8S namespace pool was defined with subnet_prefix parameter above.

<b>no_snat_ip_blocks = K8S-NOSNAT-IP-BLOCK :</b> This setting defines from which IP block each K8S namespace, that <b>will not be Source NATed</b>, is going to carve its IP Pool/IP address space from. 

<b>external_ip_pools = K8S-NAT-POOL :</b> This setting defines from which IP pool each SNAT IP will be allocated from. Whenever a new K8S namespace is created, then a NAT IP will be allocated from this pool for that K8S namespace. 

<b>top_tier_router = None :</b> : This parameter is <b>NOT</b> used in this demonstration. Since because in this example a new Tier 1 Gateway os provisioned by NCP for this K8S cluster. However if an existing Tier 1 Gateway is intended to be used then this parameter can point out to an existing Tier 1 Gateway' s UUID. With this option the K8S cluster can be collapsed down to a single Tier 1 Gateway. However do keep in mind that the existing Tier 1 Gateway should have SR component in place (meaning that it should be associated with an Edge Cluster already)  

<b>external_ip_pools_lb = K8S-LB-Pool :</b> This setting defines from which IP pool each K8S service, which is configured with Type : Load Balancer, will allocate its IP from.

<b>overlay_tz = 26683c3a-928d-4416-8c61-e248ab788cd5 :</b> The UUID of the existing overlay transport zone that will be used for creating new segments for K8S namespaces and Pod networking. 

Note : Transport Zone UUID is pulled by copying the actual value in the "ID" column of the transport zone in the UI.

<b>top_firewall_section_marker = 19d271f0-1623-11ea-93b3-f5b3eacfe64f</b> and <b>bottom_firewall_section_marker = 25a3a710-1623-11ea-93b3-f5b3eacfe64f</b> These parameters are to specify between which sections the K8S orchestrated firewall rules will fall in between. 

The following parameters are used in the <b>"[coe]"</b> sub-section of the configmap for NCP.ini.

<b>cluster = k8s-cluster :</b> Used to identify the NSX-T objects that are provisioned for this K8S cluster. Notice that K8S Node segment ports in "K8S-NodeTransport" are configured with the "k8s-cluster" tag and the "ncp/cluster" scope also with the hostname of Ubuntu node as the tag and "ncp/node_name" scope on NSX-T side.

<b>enable_snat = True :</b> This parameter basically defines that all the K8S Pods in each K8S namespace in this K8S cluster will be Source NATed by default. The Source NAT entries will be automatically configured on the Tier 1 Gateway which will also be automaticcally instantiated for the K8S cluster. The Source NAT IPs will be allocated from IP Pool named "K8S-NAT-POOL" that was configured earlier on NSX-T.

The following parameters are used in the <b>"[k8s]"</b> sub-section of the configmap for NCP.ini.

<b>apiserver_host_ip = 10.190.5.10</b> and <b>apiserver_host_port = 6443</b> : These parameters are to access K8S API.

<b>ingress_mode = nat :</b> This parameter basically defines that NSX-T will use SNAT/DNAT rules for K8S ingress (L7 HTTPS/HTTP load balancing) to access the K8S service at the backend.

### Configmap for NSX Node Agent

<b>ovs_uplink_port = ens192 :</b> The only parameter that needs to be configured in configmap for NSX Node Agent is the uplink port that will be used for Open vSwitch on the K8S Nodes. In this demonstration the second vNIC of each K8S node is used as transport and its ethernet naming is "ens192". 

### Additional Info about the Manifest

Some of the paramaters in the unified manifest can also be used to create new objects in NSX-T, for instance if a given IP Pool is not configured on NSX-T then the respective parameter in the manifest can be populated with the desired IP pool address space and etc. Another example could be assigning a specific name rather than the default name of "br-int" for the OVS oon K8S nodes. 

The manifest file that is used in this demonstration can be found [here](https://raw.githubusercontent.com/dumlutimuralp/k8s-with-nsx-t-2.5.x/master/Yaml/custom-ncp-ubuntu.yaml)

## Next Step

Make sure to copy the recently edited/saved manifest file (yaml) to the K8S master node (from where the NSX components will be deployed) .
