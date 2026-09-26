<div align="center">

# AWS Peering Setup
### Three VPCs · Two Regions · Private connectivity

**Project 01 — AWS Networking**

Connect Web, Application, and Database networks through a full mesh of VPC peering connections.

[Architecture](#architecture) · [Implementation](#implementation) · [Validation](#validation) · [Troubleshooting](#troubleshooting)

</div>

---

## Project overview

This lab documents a network design with two VPCs in **US East (N. Virginia)** and a third in **US East (Ohio)**. Each VPC has a direct connection to both peers, providing routes for private traffic across the three networks.

**Implementation evidence:** Ten supplied console screenshots document the Web/App VPCs, peering request workflow, route configuration, inter-Region acceptance, and successful private-IP ICMP replies. The walkthrough below explains how to reproduce the design; the evidence section distinguishes observed results from remaining checks.

## Architecture

![VPC peering architecture: Web and App VPCs in us-east-1, Database VPC in us-east-2, with three direct peering connections](images/architecture.png)

### Address plan

| Component | Region | VPC CIDR | Lab subnet | Test host private IP |
| --- | --- | --- | --- | --- |
| Web | `us-east-1` | `10.1.0.0/16` | `10.1.1.0/24`¹ | `10.1.1.100` |
| Application | `us-east-1` | `172.16.0.0/16` | `172.16.1.0/24` | `172.16.1.100`¹ |
| Database | `us-east-2` | `192.168.0.0/16` | `192.168.1.0/24` | `192.168.1.100`¹ |

¹ Suggested lab values. The supplied diagram specifies the Web host IP and the App and Database subnets, but does not specify the Web subnet or the other host IPs. Substitute your actual values when implementing.

### Connection plan

| Name | Requester | Accepter | Scope |
| --- | --- | --- | --- |
| `web-app-peer` | Web | App | Same Region |
| `web-db-peer` | Web | Database | Inter-Region |
| `app-db-peer` | App | Database | Inter-Region |

**Why three connections?** Peering is not transitive. Web ↔ App and App ↔ Database do not create a Web ↔ Database path. This topology therefore uses a direct peer for every pair. See [AWS peering behavior and limitations](https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-basics.html).


## Implementation

### 1. Create the three networks

In the VPC console, create the following VPCs and subnets using the address plan above:

1. In `us-east-1`, create `web-vpc` and its Web subnet.
2. In `us-east-1`, create `app-vpc` and its Application subnet.
3. Switch to `us-east-2`, then create `db-vpc` and its Database subnet.

Create a dedicated route table for each lab subnet: `web-rt`, `app-rt`, and `db-rt`. Explicitly associate each subnet with its corresponding table. Record the actual VPC, subnet, and route-table IDs.

An internet gateway or NAT gateway is not needed for the peering traffic itself. Administration, package installation, and service access need their own connectivity design.

### 2. Create and accept the connections

In **VPC → Peering connections**, create each connection from the requester Region using the connection plan.

For `web-app-peer`, select the App VPC in the same Region and accept the request in `us-east-1`.

For `web-db-peer` and `app-db-peer`, choose a peer in another Region, set the accepter Region to `us-east-2`, and enter the Database VPC ID. Switch to `us-east-2` to accept both requests.

Wait until all three connections are **Active** and record their `pcx-...` IDs. AWS documents the requester/accepter Region requirements in [Create a VPC peering connection](https://docs.aws.amazon.com/vpc/latest/peering/create-vpc-peering-connection.html).

### 3. Configure all six routes

Add the following entries to the route tables associated with the lab subnets. Replace connection names with their actual `pcx-...` IDs. Keep each VPC's existing local route.

| Route table | Destination | Target connection |
| --- | --- | --- |
| `web-rt` | `172.16.0.0/16` | `web-app-peer` |
| `web-rt` | `192.168.0.0/16` | `web-db-peer` |
| `app-rt` | `10.1.0.0/16` | `web-app-peer` |
| `app-rt` | `192.168.0.0/16` | `app-db-peer` |
| `db-rt` | `10.1.0.0/16` | `web-db-peer` |
| `db-rt` | `172.16.0.0/16` | `app-db-peer` |

For example, a Web request to `192.168.1.100` follows `web-db-peer`; the response to `10.1.1.100` must use that same connection from `db-rt`. Repeat the routes in any additional route tables whose subnets need this access. See [AWS route-table configuration](https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-routing.html).

### 4. Permit the required traffic

For a temporary Linux connectivity lab, allow **ICMP IPv4 Echo Request** on each test host's security group from the other two test hosts' private `/32` addresses. Permit the corresponding outbound requests if egress is restricted.

For service testing, use narrowly scoped rules such as:

| Destination | Example inbound rule | Purpose |
| --- | --- | --- |
| App test host | TCP `8080` from `10.1.1.100/32` | Web-to-App request |
| DB test host | TCP `5432` from `172.16.1.100/32` | App-to-PostgreSQL request, if PostgreSQL is installed |

These ports are examples, not properties inferred from the diagram. Use the real application and database ports. Do not open every port to the peer CIDRs just to make a test pass.

Use private IP or CIDR rules across Regions; peer security-group references are supported only for same-Region peers. See [AWS peer security-group rules](https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-security-groups.html).

If using custom network ACLs, allow the required traffic in both directions, including return traffic and TCP ephemeral ports where applicable. Also check the operating-system firewall and service binding address.

### 5. Launch the test hosts

Launch one small Linux EC2 test instance in each lab subnet and attach the corresponding security group. Assign the planned private IPs if available, or update the tests and `/32` rules to match the assigned addresses.

The Database test host is an EC2 connectivity target. Deploying an actual managed database requires additional configuration; the diagram's single Database subnet is not a complete RDS deployment design.

Private IP tests below do not depend on DNS. If later using hostnames, configure and validate DNS separately; peering does not automatically share a Route 53 private hosted zone.

## Validation

### Check the control plane

Run these read-only commands with AWS CLI to inspect the peering states in each Region:

```bash
aws ec2 describe-vpc-peering-connections --region us-east-1 --query 'VpcPeeringConnections[].{ID:VpcPeeringConnectionId,Status:Status.Code}' --output table
aws ec2 describe-vpc-peering-connections --region us-east-2 --query 'VpcPeeringConnections[].{ID:VpcPeeringConnectionId,Status:Status.Code}' --output table
```

Match the results to your three recorded connection IDs. An Active state alone does not prove end-to-end connectivity.

### Test the data plane

Run on the **Web host**:

```bash
ping -c 4 172.16.1.100
ping -c 4 192.168.1.100
```

Run on the **App host**:

```bash
ping -c 4 10.1.1.100
ping -c 4 192.168.1.100
```

Run on the **Database test host**:

```bash
ping -c 4 10.1.1.100
ping -c 4 172.16.1.100
```

Expected result after permitting ICMP: replies from each destination's private IP. Record actual output rather than treating this expectation as a completed test.

Once services are running, test the actual ports as well. For example, on the App host, with netcat installed and PostgreSQL listening on the DB host:

```bash
nc -vz -w 5 192.168.1.100 5432
```

A successful TCP connection checks the network path and listener; it does not validate database credentials or application behavior.

### Observed results

The terminal screenshot captures these successful tests:

| Source | Destination | Visible result |
| --- | --- | --- |
| Web `10.1.14.80` | DB `192.168.11.41` | Repeated ICMP replies, roughly 11.8–11.9 ms in the visible sample |
| App `172.16.14.199` | DB `192.168.11.41` | Repeated ICMP replies, mostly 11.1–11.2 ms in the visible sample |
| DB `192.168.11.41` | App `172.16.14.199` | Repeated ICMP replies, roughly 13.3–13.4 ms in the visible sample |

The screenshots do not show final packet-loss summaries, a Web-to-App ping, application-port tests, or every final route table. The visible replies demonstrate connectivity at capture time, not a latency benchmark or a claim that all validation checks were completed.

To repeat the observed checks, run on Web and App:

```bash
ping -c 4 192.168.11.41
```

Then run on DB:

```bash
ping -c 4 172.16.14.199
```

## Implementation screenshots

### 01 · Establish the Web and App networks

The VPC list shows `10.1.0.0/16` and `172.16.0.0/16` in N. Virginia alongside the default VPC. The App VPC ID differs between this early capture and later captures; use the IDs from your current environment rather than copying IDs across screenshots.

![Web and App VPC inventory](images/01-implementation.png)

### 02 · Request same-Region peering

Select the Web VPC as requester and the App VPC as accepter. Both CIDRs are displayed before submission.

![Create Web to App peering](images/02-implementation.png)

### 03 · Accept the request

The newly requested connection is pending acceptance. Use the acceptance action to progress the connection; a later screenshot shows this Web–App connection as Active.

![Peering pending acceptance](images/03-implementation.png)

### 04 · Identify the subnet route tables

The inventory captures identify the named Web and App route tables and their explicit subnet associations. Route entries must be added to the tables actually used by the instances.

![Named route table overview](images/04-implementation.png)

<details>
<summary>View additional route-table association captures</summary>

![Route tables with VPC mappings](images/05-implementation.png)

![Full route table inventory](images/06-implementation.png)

</details>

### 05 · Add the Web-to-App route

The Web route editor targets `172.16.0.0/16` through the peering connection. This screenshot captures the edit form before saving. Its existing default internet route serves a separate purpose; the more specific peer route selects the private path.

![Web route edit](images/07-implementation.png)

### 06 · Add the return route

The App route editor targets `10.1.0.0/16` through the same peering connection. Save the changes and inspect the final route state in the console.

![App return route edit](images/08-implementation.png)

### 07 · Accept inter-Region peering

The acceptance dialog shows a requester in Ohio with CIDR `192.168.0.0/16` and the Web accepter in N. Virginia. The Web–App connection is already Active in the background. A separate background error references a VPC that does not exist; verify current VPC IDs and Regions when reproducing the setup. This dialog itself does not prove that acceptance has completed.

![Ohio to N Virginia peering acceptance](images/09-implementation.png)

### 08 · Verify private connectivity

Three EC2 Instance Connect sessions show successful replies between the deployed private IPs. The source instance details at the bottom of each session identify Web, App, and DB.

![Successful private IP ping tests across Regions](images/10-implementation.png)

### Evidence still useful for a complete audit

- Final Active state of all three peer connections.
- Saved routes for the Database VPC and both cross-Region paths.
- Security-group rules and actual deployed subnet CIDRs.
- Web-to-App connectivity and application-port tests.

## Troubleshooting

| Symptom | What to inspect |
| --- | --- |
| Connection remains pending | Accept from the peer account and the accepter Region. |
| Route shows `blackhole` | Confirm the selected peering connection exists and is Active. |
| Every request times out | Check both subnet route tables, associations, security groups, ACLs, and host firewalls. |
| One pair works but another fails | Verify the failing pair's direct peering connection and its two routes. |
| Ping succeeds but service connection fails | Check the service port, listener, bind address, and TCP rules. |
| Service connects but ping fails | Check ICMP rules; a service can work while ICMP is blocked. |
| Private IP works but hostname fails | Inspect DNS settings and zone associations separately. |

## Cleanup

After collecting evidence, remove only the resources created for this lab:

1. Terminate the test instances and remove any lab database resources.
2. Remove the six peer routes and temporary security-group rules.
3. Delete the three peering connections.
4. Remove lab-only endpoints, NAT gateways, and other dependencies if you created them.
5. Delete the lab subnets, custom route tables, security groups, and VPCs.
6. Check for retained volumes, snapshots, and allocated public IPs.

Resource usage and inter-Region data transfer can incur charges. Review [Amazon VPC pricing](https://aws.amazon.com/vpc/pricing/) before running the lab.

## Key takeaways

The finished lab should demonstrate three direct peer connections, six explicit routes, and verified private traffic between each VPC pair. Routing establishes a path; security controls determine which traffic may use it. Save both the configuration and test evidence so the implementation can be reproduced and reviewed.
