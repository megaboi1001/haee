# HMSG WEC Racing – PRD (eu-central-1) AWS Deployment

CloudFormation template to create the production environment for **HMSG (Hyundai Motorsport GmbH)** — WEC Racing — in **eu-central-1 (Frankfurt)**.

This matches the requirements gathered in `Portfolio Collection Template - Assess.xlsx` (e.g. `hmsg-rac-*` naming, 172.200.0.0/24 VPC, m7i.2xlarge MSSQL host, AWS Client VPN + Entra ID SAML, Site-to-Site IPSEC VPN to the racing site).

---

## What this template creates

| Area | Resources | Notes |
|---|---|---|
| **VPC** | `hmsg-rac-prd-vpc-ec1` (172.200.0.0/24) | Flow Logs → S3 (lifecycle 1 month → Glacier, expire 1 year). ⚠️ see [VPC CIDR warning](#-vpc-cidr-warning) |
| **Subnets** | 2 public (ec1a/ec1c) + 2 private (ec1a/ec1c) + 2 DB (ec1a/ec1c), all `/27` | Derived from `VpcCidr` via `Fn::Cidr` — changing `VpcCidr` re-derives them instead of breaking the stack. Dual-AZ per PRD design |
| **Routing** | pub → IGW, pri → NAT, DB → (VPN only, no internet) | VGW propagation on pri/db |
| **NAT** | `hmsg-rac-prd-NAT-pub-ec1` + EIP | In pub-ec1a |
| **EC2** | `hmsg-rac-prd-mssql01-ec2-ec1a` | m7i.2xlarge (8 vCPU/32GiB), Win Server 2022 (EN), root 250GB gp3 + data **4TB gp3 (D:)**, encrypted, EBS-optimised, **IMDSv2 required**, in DB-AZ subnet. **SQL Server is installed by the client** |
| **VPC Endpoints** | Interface endpoints for `secretsmanager`, `ssm`, `ssmmessages`, `ec2messages` in the DB subnet | The DB subnet has no NAT/IGW route by design — these are the *only* way the instance reaches AWS APIs (sysadm password fetch at boot, SSM Session Manager). The instance `DependsOn` all four so it cannot boot before they exist |
| **Security Groups** | `*-common-ec2-sg-ec1` (RDP 3389), `*-mssql-ec2-sg-ec1` (MSSQL 1433), `*-clientvpn-sg-ec1` (443/35001), `*-vpce-sg-ec1` (443, endpoints) | Explicit ingress + egress on every SG; only reachable via Client VPN / on-prem |
| **Client VPN** | `hmsg-rac-prd-clientvpn-ec1`, client CIDR 10.200.0.0/22, split tunnel, SAML (**Entra ID**) auth | **UDP 443 only** (no TCP fallback exists — an endpoint is one transport protocol). Logs → CloudWatch (90d, AWS service-linked role). Target networks: private subnets. Route to whole VPC, authorize all groups |
| **ACM** | `vpn.hmg-racing.com` server certificate (DNS validation) | Or pass an existing issued cert ARN — a new cert is only requested when `ServerCertificateArn` is empty |
| **Site-to-Site VPN** | `*-cgw-ec1` + `*-vgw-ec1` + `*-vpn-conn-ec1` (static routing) | **Optional** — skipped entirely unless *both* `OnPremCidr` and `CustomerGatewayIp` are supplied. See [Deploying without the client's values](#deploying-without-the-clients-values) |
| **Secrets** | `hmsg-rac-prd-sysadm-password` | sysadm password stored in Secrets Manager (read by EC2 at first boot via the Secrets Manager VPC endpoint, not embedded in plaintext user data). `DeletionPolicy: Retain` — matches the instance, so deleting the stack can't strand a retained host without its password |
| **IAM** | SSM role for EC2, SAML provider | S3 flow logs and Client VPN CloudWatch logs are authorized by AWS-managed bucket policy / service-linked role respectively — no custom IAM role needed for either |

## What is NOT in this template (client / 3rd-party responsibility)

- **Microsoft Entra ID Enterprise Application** – created by the **client** (HMGR M365 team). They need these SAML values (from the Q&A sheet):
  - Identifier (Entity ID): `urn:amazon:webservices:clientvpn`
  - Reply URL: `https://self-service.clientvpn.amazonaws.com/api/auth/sso/saml`
  - Sign-on URL: `http://127.0.0.1:35001`
- **Entra ID Federation Metadata XML** – the client generates it from the Entra app; **we apply it** to the IAM SAML provider after deploy via `aws iam update-saml-provider` (see below — it's too large for a CFN parameter).
- ~~**DNS records for `vpn.hmg-racing.com`**~~ – **not required.** Use a self-signed cert imported into ACM; see [Server certificate](#server-certificate). Only needed if you deliberately choose the public-ACM path.
- **SQL Server 2019 installation** on the EC2 host (client does it; port 1433 TSG is pre-opened).
- **Client VPN config distribution** to users (downloaded from console; users authenticate with Entra ID SSO).
- **On-premise VPN appliance** side of the IPSEC tunnel.

---

## Architecture (simplified)

```
Internet
   │
   ├── IGW ── NAT ─────────────────────────────────────────────┐
   │                                                           │
┌──┴───────────────┬───────────────────────────┬────────────────┴──────────────┐
│ pub-ec1a /27     │ pri-ec1a /27  pri-ec1c /27│ db-ec1a /27  db-ec1c /27      │
│ pub-ec1c /27     │  ▲                        │   ┌─────────────────────┐     │
│                  │  │ Client VPN target nets │   │ EC2 mssql01         │     │
│                  │  │                        │   │ m7i.2xlarge          │     │
│                  │  │                        │   │ Windows 2022         │     │
│                  │  │                        │   │ 250G + 4T (D:)       │     │
│                  │  │                        │   └─────────────────────┘     │
└──────────────────┴──┼────────────────────────┴──────────────────────────────┘
                      │   VGW ───── IPSEC Site-to-Site ─────► On-prem race site
                      │
        Up to 10 devs ⇄ AWS Client VPN (UDP 443, SAML/Entra ID)
```

---

## ⚠️ VPC CIDR warning

The requirement sheet specifies **172.200.0.0/24**, and that is the template default — but it is **not RFC1918 private space**. Private ranges are `10.0.0.0/8`, `172.16.0.0/12` (i.e. `172.16.x` – `172.31.x` only), and `192.168.0.0/16`. `172.200.x.x` is publicly allocated address space belonging to someone else on the internet.

Consequences: nothing breaks inside AWS (Route 53 Resolver supports non-RFC1918 VPC CIDRs), but the VPC can **never reach the real internet hosts that own 172.200.0.0/24**, and the range will collide if it is ever routed to a peer, another VPC, or the on-prem network.

**Action:** confirm with the client that this was intentional. If it wasn't, deploy with a private range — the subnets are derived with `Fn::Cidr`, so any `/24` works with no other changes:

```bash
... --parameter-overrides VpcCidr=172.20.0.0/24 ...
```

---

## Prerequisites

1. AWS CLI v2 configured for the account **440027026402** (and logged in).
2. Region: **eu-central-1**.
3. EC2 key pair `hmsg-rac-prd-key-ec2-ec1` in eu-central-1 (create if missing):
   ```bash
   aws ec2 create-key-pair --key-name hmsg-rac-prd-key-ec2-ec1 --region eu-central-1
   ```
4. A **Client VPN server certificate** in ACM, in `eu-central-1`. This does **not** need DNS validation or a domain you own — see [Server certificate](#server-certificate).
5. Choose values:
   - `AdminPassword` – **required**, password for `sysadm` (min 8 chars). **Effectively create-only** — see [Rotating the sysadm password](#rotating-the-sysadm-password)
   - `ServerCertificateArn` – the ACM ARN from step 4 (recommended; otherwise the stack tries to issue a public cert and waits on DNS)
   - `CustomerGatewayIp` + `OnPremCidr` – **optional, all-or-nothing.** Public IP of the on-prem race-site VPN device, and the on-premise network CIDR. Omit both to deploy without the Site-to-Site VPN. If set, `OnPremCidr` **must not overlap `ClientVpnCidr`** (default `10.200.0.0/22`) or `VpcCidr` — e.g. don't use `10.0.0.0/8`, it swallows the Client VPN range.

---

## Server certificate

A Client VPN endpoint requires an ACM server certificate **regardless of the authentication method** — the cert secures the TLS tunnel and is unrelated to Entra ID user sign-in.

It does **not** have to be publicly trusted. AWS's own SAML walkthrough uses *a private certificate imported into ACM*. The `.ovpn` file AWS generates embeds your CA in a `<ca>` block, and users connect to the AWS-assigned endpoint hostname — so **`vpn.hmg-racing.com` never has to exist or resolve, and no DNS validation is needed.** This is the recommended path because it removes the client and the DNS owner from the critical path entirely:

```bash
git clone https://github.com/OpenVPN/easy-rsa.git
cd easy-rsa/easyrsa3
./easyrsa init-pki
./easyrsa build-ca nopass                                     # CN: hmsg-rac-prd-ca
./easyrsa --san=DNS:vpn.hmg-racing.com build-server-full server nopass

aws acm import-certificate \
  --certificate       fileb://pki/issued/server.crt \
  --private-key       fileb://pki/private/server.key \
  --certificate-chain fileb://pki/ca.crt \
  --region eu-central-1 --query CertificateArn --output text
```

> ⚠️ Keep the key at **RSA 2048**. Client VPN supports only 1024- and 2048-bit RSA; a 4096-bit key fails. easy-rsa 3.x defaults to 2048 — don't override it. **Back up `pki/`** — renewal requires the same CA.

Pass the resulting ARN as `ServerCertificateArn` and neither ACM resource in the template is created.

<details>
<summary>Alternative: public ACM certificate (only if you control <code>hmg-racing.com</code>)</summary>

Omit `ServerCertificateArn` and the template requests a public cert for `VpnDomainName` with DNS validation.

- **Hosted zone in this account:** pass `HostedZoneId=Z0XXXXXXXX` and ACM writes the validation record itself.
- **DNS managed elsewhere:** the stack **blocks** at the certificate resource until someone adds the CNAME. Retrieve it with:
  ```bash
  aws acm describe-certificate --region eu-central-1 \
    --certificate-arn "$(aws cloudformation describe-stacks --stack-name hmsg-rac-prd \
        --query 'Stacks[0].Outputs[?OutputKey==`ServerCertificateArn`].OutputValue' --output text)"
  ```
  Hand `ResourceRecord.Name` / `.Value` to the DNS owner; the stack resumes once ACM sees it.
</details>

---

## Deploy

### Deploying without the client's values

`CustomerGatewayIp` and `OnPremCidr` are **optional and all-or-nothing**. Leave both empty and the stack skips the Site-to-Site VPN (customer gateway, VGW + attachment, VPN connection, static route, both route propagations) *and* the two on-premise security-group ingress rules — 9 resources in total. Everything else (VPC, EC2, VPC endpoints, Client VPN) deploys normally, so you are not blocked waiting on the racing-site team.

```bash
cd /Users/dda/Downloads/haee

aws cloudformation deploy \
  --template-file hmsg-rac-prd.yaml \
  --stack-name hmsg-rac-prd \
  --region eu-central-1 \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    AdminPassword='<real-password>' \
    ServerCertificateArn=arn:aws:acm:eu-central-1:<acct>:certificate/<id>
```

Check what you got: the `SiteToSiteVpnDeployed` output reports `true`/`false`.

### Adding the Site-to-Site VPN later

When the client supplies both values, re-deploy with **only those two overrides**. Every parameter you omit is sent as `UsePreviousValue: true`, so `AdminPassword` and `ServerCertificateArn` keep the values already stored in the stack — you do not need to (and should not) re-type them:

```bash
aws cloudformation deploy \
  --template-file hmsg-rac-prd.yaml \
  --stack-name hmsg-rac-prd \
  --region eu-central-1 \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    CustomerGatewayIp=203.0.113.10 \
    OnPremCidr=10.50.0.0/16
```

This is an **add-only** update — 9 new resources, 0 modified, 0 replaced:

| Added | |
|---|---|
| `CustomerGateway` | on-premise device (`CustomerGatewayIp`, BGP ASN 65000) |
| `VpnGateway` + `VpcVgwAttachment` | VGW attached to the VPC |
| `VpnConnection` + `VpnConnectionRouteOnPrem` | static-route tunnel to `OnPremCidr` |
| `VpnPropagationPri`, `VpnPropagationDb` | route propagation onto the private + db route tables |
| `CommonSgOnPremRdp`, `MssqlSgOnPremMssql` | on-premise RDP 3389 / MSSQL 1433 ingress |

The on-premise SG rules are **standalone `AWS::EC2::SecurityGroupIngress` resources**, not `Fn::If` entries inside `CommonSg`/`MssqlSg` — that is what keeps the two security groups out of the change set. Route tables are likewise untouched: propagation is its own resource. Existing routes stay in place; the on-premise route simply appears alongside them.

Prove it before applying, with `--no-execute-changeset`:

```bash
aws cloudformation deploy ... --no-execute-changeset   # prints a change-set ARN

aws cloudformation describe-change-set \
  --change-set-name <arn-from-above> --region eu-central-1 \
  --query 'Changes[].ResourceChange.[Action,LogicalResourceId,Replacement]' --output table
```

Expect 9 rows, all `Add`, all `Replacement: None`. Then re-run without `--no-execute-changeset`.

> **Warnings**
> - Do **not** pass a different `AdminPassword` on an update — it rewrites the secret without changing the Windows account. See [Rotating the sysadm password](#rotating-the-sysadm-password).
> - Changing `CustomerGatewayIp` **later** (racing site moves) is *not* add-only: it replaces `CustomerGateway` and `VpnConnection`, which issues new tunnel outside-IPs and new pre-shared keys. The on-premise device must be reconfigured from the freshly downloaded tunnel config.
> - `VpnConnection` takes several minutes. If it fails, the update rolls back by deleting only these 9 resources; the VPC, EC2 instance, secret and Client VPN endpoint are unaffected.
> - Use a real customer-gateway IP, not the `203.0.113.x` documentation range.

Supplying only one of the two values silently skips the VPN rather than half-building it. If the `SiteToSiteVpnDeployed` output says `false` when you expected `true`, one of the two is missing.

---

## Post-deployment steps

1. **Client VPN config**: VPC console → Client VPN → *Client VPN Endpoints* → select `hmsg-rac-prd-clientvpn-ec1` → **Download Client Configuration**. Distribute the `.ovpn` to the ≤10 users. They sign in via **Entra ID SSO** (MFA enforced by client's tenant policies).

2. **SAML metadata – apply the real XML** (required before anyone can log in):
   The stack deploys with an inline **placeholder** SAML provider. Real Entra ID Federation Metadata XML is ~13 KB — larger than the 4096-character CloudFormation parameter limit — so it is applied **out-of-band** to the IAM provider created by the stack:
   ```bash
   SAML_ARN=$(aws cloudformation describe-stacks --stack-name hmsg-rac-prd --region eu-central-1 \
     --query 'Stacks[0].Outputs[?OutputKey==`SamlProviderArn`].OutputValue' --output text)

   aws iam update-saml-provider \
     --saml-provider-arn "$SAML_ARN" \
     --saml-metadata-document file:///path/to/client-metadata.xml
   ```
   The placeholder is re-applied only if the inline value in the template changes — a normal stack update leaves your real metadata untouched.

3. **Site-to-site tunnel** *(only once `CustomerGatewayIp`/`OnPremCidr` are deployed)*: VPC console → Site-to-Site VPN → `hmsg-rac-prd-vpn-conn-ec1` → **Download Configuration** (vendor: *Generic*, platform: *Generic*). Give the tunnel 1/2 config (IPs + pre-shared keys) to the on-prem team to configure their firewall.

4. **SQL Server install (client)**: RDP to the `MssqlPrivateIp` stack output (in the `db-ec1a` /27) via Client VPN using `sysadm`. Put SQL data/log on **D: drive** (already formatted NTFS, labelled `SQLData`). Port 1433 TCP is opened on Windows Firewall and the security group.

5. **Entra ID app (client)**: client creates the *AWS ClientVPN* Enterprise Application using the SAML values above, assigns the VPN user group, and generates the Federation Metadata XML → send to us (item 2).

6. **Reach on-prem from VPC**: route table `*-rtb-db-ec1` and `*-rtb-pri-ec1` receive the static on-prem route automatically via VGW propagation (verified via `VPNConnectionRouteOnPrem`).

---

## Access details

- **RDP to DB host**: `sysadm` / password from Secrets Manager (`admin/secret value`), or built-in `Administrator` password decrypted with the key pair (EC2 console → *Get Windows password*).
- **Private IP**: `MssqlPrivateIp` output (the 5th /27 of `VpcCidr`, DB-AZ a).
- **SSM**: EC2 has `AmazonSSMManagedInstanceCore` — Session Manager works as a backup via the `ssm`/`ssmmessages`/`ec2messages` VPC interface endpoints (no internet needed, but the endpoints are required — this doesn't work without them).

---

## Update notes / operational items

- **Race site moves** → the customer-gateway public IP changes. Update the stack:
  ```bash
  aws cloudformation deploy ... --parameter-overrides CustomerGatewayIp=<new-ip>
  ```
  (replaces CGW + VPN connection; tunnel config must be re-shared with on-prem).
- **DB instance has no internet egress** (by design, matches the Q&A sheet) — only reaches Secrets Manager/SSM via the VPC interface endpoints in that subnet, nothing else. SQL media must be staged via VPN. If Windows Update / external downloads are ever needed, add `0.0.0.0/0 → NAT` to the DB route table.
- **Instance protected** from accidental deletion (`DeletionPolicy: Retain`). `SysadmSecret` is retained too, so a retained host never loses the only copy of its admin password.
- **Flow log bucket is retained** (`DeletionPolicy: Retain`) — CloudFormation cannot delete an S3 bucket that still holds objects, so a stack teardown leaves `hmsg-rac-prd-flowlog-<acct>` behind with its logs. Empty it with `aws s3 rm s3://<bucket> --recursive` afterward (treat it as the flow-log archive).

### Rotating the sysadm password

UserData runs **only at first boot**. Passing a new `AdminPassword` on a stack update therefore rewrites the Secrets Manager value but **does not change the Windows account** — the stored secret would silently stop working. Rotate in this order instead:

```bash
# 1. Change it on the host first (RDP or Session Manager):
#    net user sysadm '<new-password>'
# 2. Then bring the secret into line:
aws secretsmanager put-secret-value \
  --secret-id hmsg-rac-prd-sysadm-password \
  --secret-string '<new-password>' \
  --region eu-central-1
```

Keep passing the *original* `AdminPassword` on subsequent `deploy` runs (or let `deploy` retain it) so CloudFormation doesn't overwrite the rotated secret. If the account and secret ever do drift apart, recover with the built-in `Administrator` (EC2 console → *Get Windows password*, decrypted with the key pair) and reset `sysadm` from there.

---

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Stack stuck at ACM certificate | You're on the public-ACM path and the DNS CNAME isn't added yet. Better: cancel, import a self-signed cert, and pass `ServerCertificateArn` (see [Server certificate](#server-certificate)). |
| `SiteToSiteVpnDeployed` output is `false` | `CustomerGatewayIp` and `OnPremCidr` are all-or-nothing — one of them is empty. |
| On-prem can't reach the DB after adding the VPN | Confirm the tunnel is `UP` on both ends, then check the `*-rtb-pri-ec1`/`*-rtb-db-ec1` route tables received the propagated on-prem route. |
| `ClientVpnRoute`/`ClientVpnEndpoint` failure | Known CFN race with target-network association. Retry the stack update; if it persists, remove `ClientVpnRouteVpc`, deploy, then re-add via update. |
| Users can't sign in | Real SAML metadata not applied yet (placeholder). Run `aws iam update-saml-provider` with the real XML (see *Post-deployment*). Also confirm the Entra app Identifier/Reply URL values from the Q&A sheet. |
| Can't reach DB over Client VPN | Confirm split-tunnel on; user connected; SG allows 1433 from 10.200.0.0/22; DB host firewall allows 1433. Client is UDP 443 — there is no TCP fallback. |
| `sysadm` login fails on a fresh instance | UserData retries the Secrets Manager read 10× / 30s then throws. Check `C:\ProgramData\Amazon\EC2Launch\log\agent.log` (via `Administrator` + key pair, or Session Manager) and confirm the four VPC interface endpoints are `available`. |
| Change set fails `AWS::EarlyValidation::ResourceExistenceCheck` on a redeploy | Usually the S3 flow-log bucket name is still taken by an orphaned bucket (or an external ref is missing: key pair / ACM cert / SSM AMI param). Purge the old bucket — delete all objects *and* delete markers, then `delete-bucket` — and retry. After a torn-down stack, check `aws s3api head-bucket --bucket hmsg-rac-prd-flowlog-<acct>`; if it responds, the name is not free yet. |
| `ClientVpnEndpoint` fails "Security Groups cannot be specified without a VPC ID" | Template regression — the endpoint needs `VpcId` whenever `SecurityGroupIds` is set. Use the current template (fixed). |
| `sysadm` password from the secret stops working | Credential drift — `AdminPassword` was changed on a stack update, which does not re-run UserData. See *Rotating the sysadm password*. |
| Secret name in use | Stack was deleted and re-created within ~7 days (Secrets Manager soft-delete). Use a different `AdminSecretName` or purge the old secret with `aws secretsmanager restore-secret`. |
| EC2 shows in `pending` | AMI resolving — check `WindowsAmi` SSM parameter name is valid, key pair exists in region. |

---

## Cost notes (approx. inputs, per month)

- EC2 m7i.2xlarge + 4.25 TB gp3 EBS (~€0.01/GiB-mo → ~€43)
- Client VPN endpoint (hourly) + Client VPN logs (CloudWatch ingestion)
- Site-to-site VPN (hourly, 2 tunnels — **not billed if you deploy without `CustomerGatewayIp`/`OnPremCidr`**) + NAT gateway + transit data
- VPC Flow Logs → S3 (encrypted, lifecycle: Glacier @ 30d, expire @ 365d)
- 4x VPC interface endpoints (hourly + per-GB), single-AZ (`DbSubnetA` only)

---

## Validation

`cfn-lint` (region `eu-central-1`): **0 errors / 0 warnings.**

`cfn-guard 3.2.0` against the `aws-guard-rules-registry` EC2 / S3 / ACM / Secrets Manager / IAM / CloudWatch rule sets (64 rule files): **14 distinct rules report findings.** None is a deployment blocker; each is either a rule limitation, inherent to the design, or optional hardening that has been deliberately deferred. Full list:

| Finding | Resource | Why it's accepted |
|---|---|---|
| `EC2_SECURITY_GROUP_INGRESS_OPEN_TO_WORLD_RULE` | `ClientVpnSg` | A Client VPN endpoint must be reachable from any user's home/hotel IP. UDP 443 only. |
| `SECURITY_GROUP_INGRESS_CIDR_NON_32_RULE` | `ClientVpnSg` | Same rule, same cause — remote users have no fixed /32. |
| `RESTRICTED_INCOMING_TRAFFIC` | `CommonSg` | Flags port 3389 unconditionally, regardless of source. Ingress here is limited to `ClientVpnCidr` + `OnPremCidr`, never `0.0.0.0/0`. |
| `NO_UNRESTRICTED_ROUTE_TO_IGW` | `PubRouteInternet` | The public subnets exist solely to host the NAT gateway, which requires a default route to the IGW. |
| `IAM_NO_INLINE_POLICY_CHECK` | `Ec2Role` | One tightly-scoped inline policy (`secretsmanager:GetSecretValue` on a single secret ARN). A managed policy would be no more restrictive. |
| `S3_BUCKET_VERSIONING_ENABLED` | `FlowLogBucket` | Versioning is `Suspended` — flow logs are write-once and lifecycle-expired at 365d; versioning would only add cost. |
| `S3_BUCKET_REPLICATION_ENABLED` | `FlowLogBucket` | No cross-region DR requirement stated for flow logs. |
| `S3_BUCKET_DEFAULT_LOCK_ENABLED` | `FlowLogBucket` | Object Lock conflicts with the 365-day expiry lifecycle rule. |
| `S3_BUCKET_LOGGING_ENABLED` | `FlowLogBucket` | Server access logging on a log bucket needs a second bucket; not requested. |
| `S3_BUCKET_NO_PUBLIC_RW_ACL` | `FlowLogBucket` | **Rule limitation** — it looks for an explicit `AccessControl` property. All four `PublicAccessBlockConfiguration` settings are `true`, so the bucket is fully private. |
| `SECRETSMANAGER_USING_CMK` | `SysadmSecret` | Uses the AWS-managed key. Switch to a CMK if your compliance baseline requires it. |
| `SECRETSMANAGER_ROTATION_ENABLED_CHECK` | `SysadmSecret` | Automatic rotation would desync from the Windows local account (see *Rotating the sysadm password*). Manual rotation is documented instead. |
| `CLOUDWATCH_LOG_GROUP_ENCRYPTED` | `ClientVpnLogGroup` | Uses default CloudWatch encryption; add a KMS key if required. |
| `EC2_INSTANCE_DETAILED_MONITORING_ENABLED` | `MssqlInstance` | 1-minute metrics cost extra and weren't requested. Cheap to enable later. |

Resolved during review: `EBS_OPTIMIZED_INSTANCE` and `SUBNET_AUTO_ASSIGN_PUBLIC_IP_DISABLED` both now pass.

**Not verified:** no stack has been deployed, so layer-3 pre-deployment validation (`aws cloudformation describe-events --filters FailedEvents=true`) has not been run. Runtime concerns it would catch — AMI availability, key-pair existence, service quotas, IAM permissions of the deploying principal — remain unchecked.