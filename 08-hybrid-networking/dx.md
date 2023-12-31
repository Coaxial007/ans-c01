# Direct Connect

## Concepts

- It is a physical connection into an AWS region
- It can be 1, 10 or 100 Gbps connection
- The connection is between the business premisses => DX Location => AWS Region
- When we order a DX connection, actually we order a port allocation at a DX location. It does not provide a connection of any kind by itself, it is just a physical port. It is up to us to connect to this directly or arrange a connection to be extended via a third party comms provider
- The cost for DX includes a hourly cost + cost for the outbound data transfer. Inbound data transfer is free of charge
- To be taken in consideration:
    - Provisioning time: AWS will take time to allocate a port + arrange connection to the premises (weeks/months)
    - No builtin resilience by default
    - Provides low and consistent latency + high speeds
    - DX can be used to access AWS Private Services (running in VPCs) and AWS Public Services

## Physical Connection Architecture

- A direct connect is a physical port allocated at DX location, this physical port provides either 1, 10 or 100 Gbps connection speed
- Port allocated at the DX location requires the use of single-mode fibre, we can't connect using copper connection
- Physical layer connection standards:
    - 1Gbps connection: 1000BASE-LX (1310nm) Transceiver
    - 10Gbps connection: 10GBASE-LR (1310nm) Transceiver
    - 100Gbps connection: 100GBASE-LR4 Transceiver
- In terms of the configuration for the DX ports:
    - We have to make sure that Auto-Negotiation is disabled
    - We configure the port speed
    - Full-duplex has to be manually set on the network connection
- The router on the DX location should support the BGP protocol and BGP MD5 Authentication
- Optional configurations:
    - MACsec
    - Bidirectional Forwarding Detection (BFD)

## Direct Connect MACsec

- Partially improves the long standing problem with DX: lack of built-in encryption
- MACsec is standard, allows Layer 2 frames to be encrypted. It extends standard Ethernet
- MACsec provides a hop by hop encryption architecture. 2 devices need to be next to each other
- MACsec provides the following higher level features:
    - Confidentiality (strong encryption at L2)
    - Data Integrity (data can not be modified in transit without detection)
    - Data origin authenticity
    - Replay protection
- MACsec does not replace IPSec over DX, MACSec is not end-to-end
- MACsec is designed to allow super high speeds
- Key components:
    - **Secure Channel**: base component, each MACsec participant creates a secure channel used to send traffic to other participant (uni-directional traffic)
    - Secure Channels are assigned an identifier (SCI)
    - **Secure Associations**: communication that occurs at each secure channel, takes places as a series of transient sessions. Each secure channel generally has 1 secure association, exception when this associations are being replaced
    - MACsec modified Ethernet frames by inserting a **16bytes MACsec** tag and also adds a 16 bytes **Integrity Check Value (ICV)**
    - **MACsec Key Agreement** protocol: manages peer discovery, authentication and the generation of encryption keys
    - **Cipher Suite**: how data is encrypted, controls the algorithm, packets per key, key rotation, etc.

    ![NO MACsec](images/DXMACSec.png)

    ![MACsec 101](images/DXMacSec2.png)

- MACsec architecture:

    ![MACsec Architecture](images/DXMACsec3.png)

    ![MACsec Architecture Extended](images/DXMACSec4.png)

- MACsec is not a substitute of IPSEC encryption!

## DX Connection Process

- Letter of Authorization Customer Facility Access (LOA-CFA): it is a form, which gives the authorization to one customer to get the data center staff to connect to the equipment of another customer
- This form is able to be downloaded by a customer once a DX port has be provisioned within a DX location

## DX Virtual Interfaces BGP Session and VLAN

- DX connections are layer 2 connections (Data Link)
- We need a way to connect to multiple types of layer 3 (IP) networks (VPC and public zone) over the DX connection
- VIFs allows us to run multiple layer 3 networks over layer 2 DX connections
- A VIF is BGP Peering Session isolated within a VLAN
- VLAN isolates the different layer 3 networks using VLAN tagging
- BGP exchanges prefixes, each end nodes knows about networks at each side
- With BGP MD5 authentication we can ensure that only authenticated data is accepted at each side
- VIFs can have 3 types: Public, Private and Transit
- Public VIFs are used to connect to public AWS services or services from VPCs with public IP addressing
- Private VIFs are used to connect to private VPC resources
- Transit VIFs allow integration between DX and Transit Gateways
- A single DX connection can have at total 50 Public/Private VIFs and 1 Transit VIF, for hosted connection we can have 1 VIF
 
    ![DX VIFs + VLAN](images/DXBGPSessonVLAN.png)

- VIF internal components:
    - VLAN for isolation of traffic
    - BGP for exchanging prefixes
    - MD5 authentication

## Private VIFs

- They are used to access the resources inside 1 AWS VPC using private IP addresses
- Resources can be accessed with their private IP using private VIFs, public IPs and Elastic IPs wont work
- Private VIFs are associated with a Virtual Private Gateway (VGW) which can be associated to 1 VPC. This has to be in the same region where the DX location connection terminates
- 1 Private VIF = 1 VGW = 1 VPC (there are ways around this using Transit VIFs)
- There is no encryption on private VIFs, DX is not adding encryption and neither is the private VIFs (there are ways around this, example using HTTPS)
- With private VIFs we can use normal or Jumbo Frames (MTU of 1500 or 9001)
- Using VGW, route propagation is enabled by default
- Creating private VIFs:
    - Pick the connection the VIF will run over
    - Chose VGW (default) or Direct Connect Gateway
    - Chose who owns the interface (this account or another account)
    - Choose a VLAN id - 802.1Q which needs to match the customer config
    - We need to enter the BGP ASN of on-premises (public or private). If private use 64512 to 65535
    - We can choose IPs or auto generate them
    - AWS will advertise the VPC CIDR range and the BGP Peer IPs (`/30`)
    - We can advertise default or specific corporate prefixes (**max 100** - this is HARD limit, the interface will go into an idle state)
- Private VIFs architecture:

    ![Private VIFs Architecture](images/DXPrivateVIFS.png)
 
 - Key learning objectives:
    - Private VIFs are used to access private AWS services
    - Private VIF => 1 VGW => 1 VPC
    - VPC needs to be in the same region as the DX location
    - VGW has an AWS assigned
    - Over the private VIF runs the BGP with IPv4 or IPv6 (separate BPG peering connections)
    - We configure our own AS on the VIF, which can be private ASN or public ASN

## Public VIFs

- Are used to access AWS public zone services: both public services and services which have a public Elastic IP
- They offer no direct access to private VPC services
- We can access all public zone regions with one public VIF across AWS global network
- AWS advertises all AWS public IP ranges to us, all traffic to AWS services will go over the AWS global network
- We can advertise any public IPs we own over BGP, in case we don't have public IPs, we can work with AWS support to allocate some to us
- Public VIFs support bi-directional BGP communities
- Advertised prefixes are not transitive, our prefixes don't leave AWS
- Create a public VIF:
    - We pick the connection the VIF will run over
    - We chose the interface owner (this account or another)
    - Chose VLAN - 802.1Q, which needs to match the customer configuration
    - Chose the customer side BGP ASN (ideally this is public ANS for full functionality)
    - Configure authentication and select optional peering IP addresses
    - We have to select which prefixes we want to advertise

- Private VIFs architecture:

    ![Public VIFs Architecture](images/DXPublicVIFS.png)

## Direct Connect Public VIF + VPN

- Using a VPN gives us an encrypted and authenticated tunnel
- Architecturally, having a VPN over DX uses a Public VIF + VGW/TGW public endpoints
- With a VPN we connect to public IPs which belong to a VGW or TGW
- A VPN is transit agnostic: we can connect using a VPN to VGW or a TGW over the internet or over DX
- A VPN is end-to-end encryption between a Customer Gateway (CGW) and TGW/VGW, while MACsec is single hop based
- VPNs have wide vendor support
- VPNs have more cryptographic overhead compared to MACsec
- A VPN can be provided immediately, can be used while DX is provisioned and/or as a DX backup

    ![VPN over DX](images/DXPublicVIFVPN.png)

# BFD - Bidirectional Forwarding Detection with Direct Connect

- BFD is used to improve failover times
- It reduces the time to require to failover between VGWs
- Without BFD, the failover works as following: BGP sends `Keep-alives` every 30 seconds and has a concept of a hold-down timer (starts from 0 and counts to 90). Whenever a `Keep-alive` is received, the timer is reset. If it reaches 90, the VIF is considered unreachable
- With BFD, failover can occur in less than a second. BFD has a concept of liveness detection interval (300ms)
- BFD also has a concept of multiplier (default is 3) => failover can occur in 900ms
- BFD is enabled on VIFs by default, but in oder to work, it has to be enabled on the customer side using BGP options configuration

## BGP Communities

- BGP communities are tags attached to those prefixes which are advertised by BGP
- Extra metadata sent with routes advertised giving some extra information or context about a route
- Well known, predefined ones:
    - `NO_EXPORT`: don't advertise to EXTERNAL BGP peers (AWS uses this for incoming)
    - `NO_ADVERTISE`: do not advertise to ANY peers
- Regular BGP communities: 32 bit values, split into 2 * 16 portions
- Format of these is: `AS_NUMBER:OPERATOR_ASSIGNED_VALUE`. Example: `7224:9100`
- BGP operators act on advertisements based on communities
- BGP communities are used to provide some level of visibility of the location of route advertisement
- We can use local preference BGP community tags to achieve load balancing and route preference for incoming traffic to our network. For each prefix that we advertise over a BGP session, we can apply a community tag to indicate the priority of the associated path for returning traffic:
    - `7224:7100` - Low preference
    - `7224:7200` - Medium preference
    - `7224:7300` - High preference
- Control scope of how for to propagate our prefixes:
    - `7224:9100` - Local AWS Region
    - `7224:9200` - All AWS Regions for a continent
        - North America–wide
        - Asia Pacific
        - Europe, the Middle East, and Africa
    - `7224:9300` - Global (all public AWS Regions)
- The communities `7224:1` – `7224:65535` are reserved by AWS Direct Connect. AWS Direct Connect applies the following BGP communities to its advertised routes:
    - `7224:8100` - Routes that originate from the same AWS Region in which the AWS Direct Connect point of presence is associated
    - `7224:8200` - Routes that originate from the same continent with which the AWS Direct Connect point of presence is associated
    - No tag - Global (all public AWS Regions)
- Summary:
    - BGP communities control how far AWS advertise our routes
    - Allow BGP administrators to define rules for how to handle incoming prefix advertisements
    - Other use: local preference

## Direct Connect Gateways

- Direct Connect is a regional service
- Once a DX connection is up, we can use public VIFs to access all AWS Public Services in all AWS regions
- Private VIFs can only access VPCs in the same AWS regions via VGWs
- Direct Connect Gateway is a global network device: it is accessible in all regions
- We integrate with it on the on-premises side by creating a private VIF and associate this with a DX Gateway instead of the Virtual Private Gateway (VGW). This integrates the on-premises router with the DX Gateway
- On the AWS side we create VGW associations in any VPC in any AWS regions
- DX gateways allow to route through them to the on-premises environments and vice-versa. They don't allow VPCs connected to the gateway to communicate with each other
- We can have 10 VGW attachments per DX gateway
- 1 DX connection can have up to 50 private VIFs, each of which support 1 DX gateway and 1 DX gateway supports 10 VGW association => we can connect up to 500 VPCs
- DX gateway don't have a cost, we have cost for data transit only
- Cross-account DX Gateways: multiple account can create association proposal for a DX gateway

## Direct Connect Transit VIFs and TGW

- A DX Gateway does not route between the associated VPCs to that gateway, it only routes from on-premises to AWS side or vice-versa
- Transit Gateways are regionals, it is possible to peer TGWs allowing connections between regions
- Transit Gateways are hub-and-spoke architecture, anything associated with a TGW is able to communicate with anything other associated to that TGW
- This architecture also works within peered TGWs
- DX-TGW Architecture:

   ![DX-TGW Architecture](images/DXGateway4.png)

- A DX supports up to 50 public and private VIFs and only 1 Transit VIF
- We can connect up to 3 Transit Gateways connected to a Direct Connect connection
- An individual DX gateway can be used with VPCs and private VIFs or with Transit Gateways and transit VIFs, **NOT BOTH** at the same time!
- Consideration:
    - DX gateway does not route between its attachments, this is why the peering connection between TGWs is required
    - Each TGW can be attached up to 20 DX gateways
    - Each TGW supports up 5000 attachments, up to 50 peering attachments
- DX Gateway routing problems:
    - DX gateway only allows communications from a private VIF to any associated virtual private gateways
    - With a transit gateway we can solve this, if we connect the DX gateway to a transit gateway (works only in one region)

## Direct Connect Resilience and HA

- To improve resilience:
    - Order 2 DX ports instead of one => 2 cross connects, 2 customer DX routes connecting to 2 on-premises routes
    - Connect to 2 DX locations, have to customer routers and 2 on-premises routers in different buildings (geographically separated)
- Not resilient DX architecture:

    ![DX resilience NONE](images/DirectConnectResilience1.png)

- Resilient DX architecture:

    ![DX resilience OK](images/DirectConnectResilience2.png)

- Improved resilient DX architecture:

    ![DX resilience BETTER](images/DirectConnectResilience3.png)

- Extreme resilient DX architecture:

    ![DX resilience GREAT](images/DirectConnectResilience4.png)

## Direct Connect Link Aggregation Groups (LAG)

- LAG: allows to take multiple physical connections and configure them to act as one
- From speed perspective: the speed increases linearly depending on the number of connections
- LAG do provide resilience, although AWS does not market them as such. They do not provide any resilience regarding hardware failure or the failure of entire location
- LAGs use an Active/Active architecture, maximum 4 connection can be part of the LAG
- All connections must have the same speed and terminate at the same DX location
- `MinimumLinks`: the LAG is active as long as the number of working connections is greater or equal to this value

    ![DX LAG](images/DirectConnectLAG.png)