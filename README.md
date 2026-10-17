# Azure Network Path Audit

## Overview

In this hands-on Azure lab, I traced how a workload reaches a Storage account through a Private Endpoint. I followed the path across five configuration layers: the VNet and subnet, Network Security Group (NSG), route table, Private Endpoint, and private DNS record.

The goal was to understand the configured path from the workload to storage, identify which controls affect that path, and explain why traffic to the Private Endpoint does not follow the subnet's default route through the firewall appliance.

> **Environment note:** This is a lab investigation. The screenshots and findings document the configuration I inspected; they do not represent a production client environment.

## What I investigated

1. **VNet and subnet** — identified the workload subnet and its address range.
2. **Network Security Group** — reviewed the inbound HTTPS rule and the subnet's security-group association.
3. **Route table** — inspected the user-defined default route and its virtual-appliance next hop.
4. **Private Endpoint** — reviewed the endpoint's DNS configuration and private IP assignment.
5. **Private DNS** — checked the A record used to resolve the Storage account's private-link hostname.

## 1. VNet and workload subnet

I started with the VNet's **Subnets** blade and identified `snet-workload`. The subnet is associated with both an NSG and a route table, so I continued by examining those controls.

![Azure portal showing the workload subnet and its NSG and route-table associations](images/chapter5/01-workload-subnet.png)

**What this establishes:** the subnet is the workload's network segment, and its attached security and routing controls influence traffic to and from resources in that subnet.

## 2. Network Security Group

I inspected the inbound security rules on the workload NSG. The custom rules include an HTTPS allow rule restricted to a specific source and a separate inbound deny rule, followed by Azure's default inbound rules.

![Redacted Azure portal screenshot showing workload NSG inbound rules](images/chapter5/02-nsg-inbound-rules.png)

**What this establishes:** NSG rules filter traffic based on rule priority, source, destination, port, protocol, and action. An allow rule for TCP 443 applies to traffic matching that rule; it does not by itself mean every endpoint can successfully complete an application-level connection.

## 3. Route table and forced egress

I inspected the route table associated with the workload subnet. It contains a user-defined route for `0.0.0.0/0` with a virtual appliance as the next hop.

![Redacted Azure portal screenshot showing the workload route table](images/chapter5/03-route-table.png)

**What this establishes:** the default route directs traffic that has no more-specific matching route toward the virtual appliance. The actual next hop is selected using Azure's route-selection behavior, including longest-prefix matching.

## 4. Private Endpoint

I reviewed the Private Endpoint's DNS configuration and confirmed that it has a private IP address associated with the Storage service's blob endpoint.

![Redacted Azure portal screenshot showing Private Endpoint DNS configuration](images/chapter5/04-private-endpoint-dns.png)

**What this establishes:** the Private Endpoint provides a private IP in the VNet that maps to the target PaaS service through Azure Private Link. The endpoint connection must also be approved for the intended connection to work.

## 5. Private DNS record

I inspected the private DNS zone `privatelink.blob.core.windows.net` and its A record. The record maps the Storage account's private-link hostname to the Private Endpoint's private IP.

![Redacted Azure portal screenshot showing the private DNS A record](images/chapter5/05-private-dns-record.png)

**What this establishes:** private DNS resolution lets the application continue using the Storage account's hostname while resolving it to the Private Endpoint's private IP from a network linked to the private DNS zone.

## Packet path summary

For the workload in the VNet with the private DNS zone linked, the path is:

1. The application requests the Storage account by hostname.
2. DNS resolution returns the Private Endpoint's private IP.
3. The workload's outbound NSG rules are evaluated.
4. Azure evaluates the effective routes for that destination.
5. The more-specific Private Endpoint `/32` system route wins over the broader `0.0.0.0/0` user-defined route.
6. Traffic reaches the Private Endpoint and proceeds to the Storage service over Azure Private Link.

In this lab configuration, the `/32` route means traffic to the Private Endpoint bypasses the firewall-directed default route. That is a routing outcome to validate against the intended security design, not something to assume is always desirable.

## Ticket response: Why can the application reach Storage?

I traced the network path from the workload subnet to the Storage account to understand how the application establishes connectivity.

The application connects to the Storage account using its hostname. Azure DNS resolves the hostname through the private DNS zone linked to the VNet, returning the private IP address assigned to the Private Endpoint.

The workload subnet has an associated Network Security Group (NSG) and route table. The NSG's outbound rules allow the traffic, while the route table contains a user-defined default route (`0.0.0.0/0`) that directs otherwise-unmatched traffic to the firewall appliance.

However, the Private Endpoint introduces a more-specific `/32` system route for its private IP address. Because Azure uses longest-prefix matching, this route takes precedence over the default route. As a result, traffic destined for the Private Endpoint bypasses the firewall appliance and reaches the Storage account through Azure Private Link over Microsoft's backbone.

The Storage account has public network access disabled, and the Private Endpoint connection is approved. This allows the application to access the Storage account through the private connection while preventing access through its public endpoint.

### Finding

The application reaches Storage through private connectivity because DNS resolves the hostname to the Private Endpoint's private IP address. The `/32` system route takes precedence over the firewall-directed default route, so the firewall does not inspect this traffic.

### Recommendation

Verify that this routing behavior aligns with the intended network security policy. If Storage-bound traffic must pass through the firewall for inspection, review the routing design and determine an appropriate way to enforce that requirement.

## Private Endpoint DNS scenarios

The packet-trace simulator illustrated three different outcomes:

| Scenario | DNS result | Expected path and outcome |
|---|---|---|
| Workload in VNet; private DNS zone linked | Private Endpoint IP | Private path through Azure Private Link; Storage accepts the approved private-endpoint connection. |
| Workload in VNet; private DNS zone not linked | Public endpoint IP | Traffic follows the public-destination route toward the firewall; Storage rejects the request because public network access is disabled. |
| Laptop outside the VNet | Public endpoint IP | The laptop cannot use the VNet-linked private DNS zone and Storage rejects the public-endpoint request. |

The key lesson is that **DNS determines the destination IP address**, while routing and the Storage account's network-access configuration determine where the traffic goes and whether the service accepts it. A publicly resolvable hostname does not mean public network access is enabled.

## Key takeaways

- NSGs filter traffic; route tables determine the next hop.
- A `0.0.0.0/0` user-defined route is a catch-all, but more-specific routes can take precedence.
- Private Endpoints provide private connectivity to supported PaaS services.
- Private DNS must be configured and linked appropriately for clients to resolve the service hostname to the private IP.
- Enabling a Private Endpoint does not, by itself, disable the PaaS service's public endpoint.
- Validate effective routing rather than assuming all traffic follows the firewall route.

## Screenshot and publication checklist

Before making this repository public:

- Replace the screenshot files in `images/chapter5/` with reviewed, redacted copies.
- Remove subscription IDs, tenant identifiers, account-specific details, public source IPs, NIC identifiers, and any credentials or tokens that are not needed to explain the finding.
- Keep screenshots focused on the Azure portal blade and relevant configuration evidence.
- Confirm that no lab flags or secrets are included.
- Verify every relative screenshot path in this README matches the uploaded file names.
