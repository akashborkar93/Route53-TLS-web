# Route 53 + ACM TLS for an ALB-backed app

Setup notes for serving `https://www.akashcloud.online` and `https://akashcloud.online` from 2 VMs behind an
Application Load Balancer (ALB), with DNS in Route 53 and a free ACM certificate.

> No credentials, account IDs or resource ARNs are recorded here. Placeholders
> like `<ALB_NAME>` and `<ACCOUNT_ID>` stand in for real values.

## Architecture

```
Browser --HTTPS:443--> Route 53 (www alias) --> ALB (TLS terminates, ACM cert)
                                                   |
                                                   +--HTTP:80--> VM 1
                                                   +--HTTP:80--> VM 2
```

- Domain registrar: Namecheap (DNS delegated to Route 53)
- Region: `us-east-1`
- Load balancer: Application Load Balancer, internet-facing
- Certificate: ACM cert for `akashcloud.online` + `*.akashcloud.online` (covers apex and `www`)

## Prerequisites / pre-checks

```bash
aws sts get-caller-identity                 # confirm the right account/user
aws configure get region                    # should be us-east-1 (same as the ALB)
aws elbv2 describe-load-balancers \
  --query "LoadBalancers[].[LoadBalancerName,Type,Scheme,State.Code]" --output text
aws elbv2 describe-target-health --target-group-arn <TG_ARN>   # both targets healthy
```

ALB security group must allow inbound 80/443 from `0.0.0.0/0`. VM security groups
should allow the app port from the ALB security group.

## Steps

### 1. Create the Route 53 hosted zone
Route 53 -> Hosted zones -> Create hosted zone
- Name: `akashcloud.online` (no `www`, no `http://`)
- Type: Public

Note the 4 NS records generated.

### 2. Delegate the domain to Route 53 (Namecheap)
Namecheap -> Domain List -> Manage -> Nameservers -> **Custom DNS**, enter the 4
Route 53 NS values.

Copy any existing Namecheap records (MX, TXT, etc.) into the Route 53 zone
BEFORE switching, otherwise email/other services break.

Verify (use a public resolver, local ones cache):
```bash
nslookup -type=NS akashcloud.online 8.8.8.8
```

### 3. Request the ACM certificate
Certificate Manager (**same region as the ALB**, us-east-1) -> Request public cert
- Domains: `akashcloud.online` AND `*.akashcloud.online` (a wildcard alone does not cover the apex)
- Validation: DNS
- Click **Create records in Route 53**

Wait for status **Issued**. Pending for a long time usually means step 2 is
incomplete. Do not delete the validation CNAME (`_xxxx.akashcloud.online`) -
ACM needs it for auto-renewal.

### 4. Add the HTTPS:443 listener on the ALB
Console route failed in the KodeKloud lab:
`AccessDenied ... elasticloadbalancing:AddListenerCertificates`
(the console calls that action when choosing a cert; the lab IAM user lacks it).

Workaround - `CreateListener` via CLI works because it takes the cert directly:
```bash
aws elbv2 create-listener \
  --load-balancer-arn <ALB_ARN> \
  --protocol HTTPS --port 443 \
  --certificates CertificateArn=<ACM_CERT_ARN> \
  --ssl-policy ELBSecurityPolicy-TLS13-1-2-2021-06 \
  --default-actions Type=forward,TargetGroupArn=<TG_ARN>
```

Verify:
```bash
aws elbv2 describe-listeners --load-balancer-arn <ALB_ARN> \
  --query "Listeners[].[Port,Protocol,SslPolicy]" --output text
curl -sk -o /dev/null -w "%{http_code}\n" https://<ALB_DNS_NAME>   # 200
```

### 5. Create the DNS alias records
Route 53 -> hosted zone -> Create record
- Name: `www`
- Type: A, **Alias** on
- Target: Alias to Application and Classic Load Balancer -> us-east-1 -> the ALB
- Evaluate target health: Yes

Repeat with a **blank name** for the apex (`akashcloud.online`), same ALB target.
CLI equivalent (apex):
```bash
aws route53 change-resource-record-sets --hosted-zone-id <ZONE_ID> --change-batch '{
  "Changes":[{"Action":"UPSERT","ResourceRecordSet":{
    "Name":"akashcloud.online.","Type":"A",
    "AliasTarget":{"HostedZoneId":"<ALB_CANONICAL_ZONE_ID>",
                   "DNSName":"dualstack.<ALB_DNS_NAME>","EvaluateTargetHealth":true}}}]}'
```
(`<ALB_CANONICAL_ZONE_ID>` comes from `aws elbv2 describe-load-balancers`
-> `CanonicalHostedZoneId`.)

Use an Alias (not CNAME): free, and the only option at the apex.

### 6. Apex HTTPS (added later)
The first certificate was wildcard-only, so the apex failed TLS. Fix:
1. Request a new cert with both names (same DNS validation CNAME as before, so
   it validates automatically and becomes Issued within a minute or two):
   ```bash
   aws acm request-certificate --domain-name akashcloud.online      --subject-alternative-names "*.akashcloud.online" --validation-method DNS
   ```
2. Swap the listener's certificate. `add-listener-certificates` is denied in the
   lab, but `modify-listener` works:
   ```bash
   aws elbv2 modify-listener --listener-arn <HTTPS_LISTENER_ARN>      --certificates CertificateArn=<NEW_ACM_CERT_ARN>
   ```
3. Add the apex alias record (step 5). The old wildcard-only cert can be deleted.

### 7. HTTP:80 listener
Plan was HTTP -> HTTPS redirect (Edit listener -> Redirect to URL, HTTPS, 443,
301). Instead the HTTP:80 listener was **deleted** (console: Listeners ->
select HTTP:80 -> Manage listener -> Delete listener).

Consequence: `http://` gets no response (connection error). Only `https://`
works. Do not delete the HTTP listener until HTTPS is confirmed working.
Preferable long-term: keep it with a 301 redirect.

### 8. Final validation
```bash
curl -I https://www.akashcloud.online      # HTTP/1.1 200 OK, valid certificate
curl -I https://akashcloud.online          # apex, same
nslookup www.akashcloud.online 8.8.8.8     # resolves to ALB IPs
aws elbv2 describe-target-health --target-group-arn <TG_ARN>   # healthy
```

## Final state
| Item | Value |
|---|---|
| Hosted zone | `akashcloud.online` (public) |
| Records | NS, SOA, ACM validation CNAME, `www` and apex A aliases -> ALB |
| Certificate | `akashcloud.online` + `*.akashcloud.online`, Issued, DNS-validated |
| ALB listeners | HTTPS:443 only (ACM cert, TLS 1.3 policy) |
| Targets | 2 VMs healthy on port 80 |

## Gotchas / lessons learned
- ACM cert must be in the **same region** as the ALB.
- `route53domains:*` was blocked by an SCP in the lab; hosted zones/records
  still worked. Registration is at Namecheap anyway.
- Wildcard `*.domain` does **not** cover the bare `domain`. Request the cert with
  both names from the start, and create an apex alias record.
- Lab IAM denies `AddListenerCertificates`; use `create-listener` /
  `modify-listener` (certificate passed directly) instead.
- On Windows, the AWS CLI output has CRLF: pipe through `tr -d ''` before
  embedding values in JSON, and avoid `/tmp` paths for `file://` arguments.
- Local DNS resolvers cache old NS records; check with `8.8.8.8`.
- App behind the ALB sees HTTP; use `X-Forwarded-Proto` for scheme detection.
- Copy old registrar DNS records before changing name servers.

## Troubleshooting
- Cert stuck Pending: NS delegation not done or validation CNAME missing.
- 502/503: targets unhealthy, wrong health-check path/port.
- Timeout on 443: ALB security group missing inbound 443.
- Works on `www` but not apex: no apex alias record, or cert lacks apex name.
