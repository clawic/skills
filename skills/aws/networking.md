# Networking — VPC, Reachability, and the Bill Hiding in the Route Table

Mental model: a VPC is a private address range; a subnet is that range sliced per Availability Zone; a **route table decides what a subnet can reach**; a security group is a stateful allow-list on an interface; a NACL is a stateless allow/deny on a subnet. "Public" and "private" are not settings — a subnet is public if and only if its route table has a default route to an internet gateway.

## Address Plan (decided once, never resized)

- The VPC's primary CIDR is immutable. Secondary CIDR blocks can be added; a subnet can never be resized or moved. Plan for the largest thing you will run, not the first.
- Default sizing that survives growth: `/16` VPC, `/20` subnets (4,091 usable addresses each) — three AZs × (public, private, data) = nine subnets and room left over.
- AWS reserves 5 addresses in every subnet (network, VPC router, DNS, future use, broadcast). A `/28` has 11 usable, not 16 — this is what makes tiny subnets fail under autoscaling.
- Never reuse `10.0.0.0/16` across accounts or environments you might ever peer or connect to a corporate network. Overlapping CIDRs cannot be peered, and the fix is a re-IP migration.
- One `awsvpc`-mode ECS task or Lambda ENI consumes a subnet IP. A `/24` data subnet plus a scaling Fargate service runs out of addresses and the error reads like a capacity problem.

## Subnet Tiers

| Tier | Route to | Contains |
|---|---|---|
| Public | Internet gateway (0.0.0.0/0 → igw) | ALB/NLB, NAT gateway, bastion (if any) |
| Private (app) | NAT gateway (0.0.0.0/0 → nat) | EC2, ECS tasks, Lambda ENIs |
| Data | No default route at all | RDS, ElastiCache, anything holding state |

A database that has no route to the internet cannot be misconfigured into exposure by a later security-group edit. That is the entire argument for the third tier.

## NAT vs VPC Endpoints — Do the Arithmetic

NAT Gateway costs $0.045/hr (~$33/mo) plus $0.045/GB processed, per AZ. Traffic to AWS services does not have to go through it.

- **Gateway endpoints (S3, DynamoDB): free.** There is no scenario where a private subnet talking to S3 through NAT is correct. Create them on every route table that needs them.
- **Interface endpoints (PrivateLink) for everything else**: ~$0.01/hr per AZ (~$7.30/mo) plus $0.01/GB processed.
- Break-even when the NAT stays for other traffic: `endpoint monthly cost ÷ (NAT per-GB − endpoint per-GB)` = `7.30 ÷ (0.045 − 0.01)` ≈ **210 GB/month per endpoint per AZ**. Below that, the endpoint is a security and latency decision, not a cost one. If the endpoint lets you delete the NAT entirely, it wins immediately.
- Fargate tasks in private subnets need `ecr.api`, `ecr.dkr`, `logs`, and the **S3 gateway endpoint** (image layers are stored in S3) — miss the S3 one and pulls fail with `CannotPullContainerError` even though the ECR endpoints exist.
- One NAT gateway per AZ is the availability answer; one NAT total is the cost answer and makes an AZ failure a total outage. Choose deliberately, and remember cross-AZ traffic to a single NAT bills at $0.01/GB each way on top.

## Security Groups vs NACLs

| | Security group | NACL |
|---|---|---|
| Scope | Network interface | Subnet |
| State | Stateful — return traffic is automatic | Stateless — return traffic needs its own rule |
| Rules | Allow only | Allow and deny, evaluated in numbered order |
| Default | Deny inbound, allow all outbound | Allow all both ways |

- Reference security groups, not CIDRs: `--source-group sg-alb` survives IP churn and documents intent. A CIDR is a comment that goes stale.
- Restrict egress too. Default allow-all outbound is the exfiltration path nobody monitors, and it is also what lets a compromised container reach the metadata endpoint.
- NACL rules need ephemeral-port returns (1024-65535). Missing one produces intermittent failures that look like packet loss. Standard practice is to leave NACLs at default and do all control in security groups, using NACLs only for a subnet-level deny of a known-bad CIDR.
- Quotas that bite: 60 inbound + 60 outbound rules per security group, 5 security groups per network interface (raisable to 16, and the rule quota scales down when you raise it).

## DNS Inside the VPC

- `enableDnsSupport` and `enableDnsHostnames` both matter. With hostnames off, an RDS endpoint resolves to its *public* address from inside the VPC, and the traffic leaves and comes back — slow, billed as egress, and blocked by a private-only security group.
- The VPC resolver lives at the VPC base + 2 (`10.0.0.2` for `10.0.0.0/16`) and answers at `169.254.169.253` from any subnet. It throttles at 1,024 packets per second per network interface — a chatty service with no DNS caching hits this and sees random resolution failures.
- Route 53 private hosted zones must be *associated* with the VPC; a zone that exists but is unassociated resolves from nowhere.
- Hybrid DNS (corporate names from inside the VPC, VPC names from on-prem) needs Route 53 Resolver endpoints — inbound for on-prem→AWS, outbound with forwarding rules for AWS→on-prem. There is no way to do this with `/etc/resolv.conf` alone that survives an instance replacement.

## Connecting VPCs and Networks

| Need | Use | Watch out |
|---|---|---|
| Two VPCs, few connections | VPC peering | Not transitive; CIDRs must not overlap; route tables on both sides |
| Many VPCs / hub-and-spoke | Transit Gateway | ~$0.05/hr per attachment plus per-GB — cheap per VPC, adds up at scale |
| Expose one service to other accounts | PrivateLink endpoint service | Consumer never gets network access to your VPC — the safest cross-account shape |
| On-prem, low cost | Site-to-Site VPN | Over the internet; throughput ~1.25 Gbps per tunnel |
| On-prem, predictable | Direct Connect | Weeks of lead time; pair with a VPN for failover |

Overlapping CIDRs are the wall you hit here, and the only fixes are re-IP or a NAT layer between them. This is why the address plan is the first decision.

## Load Balancer Selection and Timeouts

- ALB: HTTP/HTTPS, WebSocket, gRPC, path/host routing, per-request pricing via LCUs, ~$16/mo floor. Idle timeout 60s by default.
- NLB: raw TCP/UDP, static IPs per AZ, source IP preservation at L4, extreme connection counts. Idle timeout on TCP/UDP listeners **defaults to 350s and is configurable from 60s to 6000s** via the listener attribute `tcp.idle_timeout.seconds` (shipped September 2024; verified 2026-07). **TLS listeners are still fixed at 350s.** So for long-lived idle connections (database proxies, message brokers): raise the attribute first, and fall back to TCP keepalive under 350s only on TLS listeners or past the 6000s ceiling.
- Gateway Load Balancer: only for inserting third-party network appliances inline.
- Cross-zone load balancing is on by default for ALB (free) and off by default for NLB (and billed as cross-AZ traffic when enabled). An NLB with uneven targets per AZ distributes unevenly until you turn it on.
- Health check defaults on an ALB target group: 30s interval, 5s timeout, 5 healthy / 2 unhealthy thresholds — so a new target is healthy after ~150s and a failing one is out after ~60s. Deregistration delay defaults to 300s, which is why a deploy appears to hang after the new tasks are already serving.

## Reachability Debugging

```bash
# The tool that replaces an hour of guessing: it names the blocking component
aws ec2 create-network-insights-path --source i-source --destination i-target \
  --protocol tcp --destination-port 5432
aws ec2 start-network-insights-analysis --network-insights-path-id nip-xxx
# Who is actually talking, and what is being rejected
aws ec2 create-flow-logs --resource-type VPC --resource-ids vpc-xxx \
  --traffic-type ALL --log-destination-type cloud-watch-logs --log-group-name /vpc/flowlogs
```

VPC Reachability Analyzer answers "can A reach B" statically, including the component that blocks it, without sending a packet. Flow logs answer "did it try, and was it accepted or rejected" — the `REJECT` records name the security group or NACL that dropped it. When neither is available, walk the six hops in order and stop at the first failure: SG inbound → SG outbound on the *source* → route table → NACL (allow inbound needs the matching ephemeral-port allow outbound) → subnet type → DNS resolution.

## Metadata Endpoint

`169.254.169.254` is reachable from any instance, which is why an SSRF bug in an application becomes credential theft. IMDSv2 (`--http-tokens required`) turns a one-line SSRF into a multi-step attack that most payloads do not implement. On ECS container instances, also stop containers from borrowing the *host's* instance role: set the agent's `ECS_AWSVPC_BLOCK_IMDS=true` for awsvpc tasks, and give the task its own task role instead.
