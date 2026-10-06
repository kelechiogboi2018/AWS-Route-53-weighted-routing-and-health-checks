# AWS Route 53 Weighted Routing with Health Checks

## Project Overview
This project demonstrates DNS-based traffic distribution and health-aware routing for a web workload hosted on two Amazon EC2 instances in the AWS London Region (`eu-west-2`). Both instances run Apache HTTP Server and publish distinct pages so DNS routing can be verified visually and with `nslookup`.

![AWS Route 53 architecture](aws-route53-weighted-routing-architecture.png)

## Problem We Solved
A web application hosted on only one server creates a single point of failure: if Apache or the EC2 endpoint becomes unavailable, users cannot reach the application. The project solves this by placing two web servers behind Amazon Route 53 weighted DNS records and associating endpoint health checks with the records. When both endpoints are healthy, DNS answers are distributed according to their weights. When one endpoint fails its health check, Route 53 can stop returning that unhealthy record and direct DNS answers to the healthy endpoint.

This also demonstrates an important distinction: Route 53 is performing **DNS routing**, not proxying every HTTP request like an Application Load Balancer.

## Architecture
- AWS Region: Europe (London), `eu-west-2`
- VPC: `10.0.0.0/16`
- Two public subnets / Availability Zones used for the two EC2 web servers
- EC2-1 public endpoint used in testing: `18.132.203.16`
- EC2-2 public endpoint used in testing: `13.40.222.71`
- Apache HTTP Server (`httpd`) listening on TCP port 80
- Route 53 public hosted zone for `kelechicloud.com`
- DNS name: `weighted.kelechicloud.com`
- Two weighted A records with equal weights (50 / 50)
- TTL: 60 seconds
- One HTTP Route 53 health check per EC2 endpoint

> Public IPs shown here are lab/test values and may change if instances are recreated without Elastic IP addresses.

## How Weighted Routing Works
Route 53 weighted routing lets multiple records share the same DNS name and record type. Each record receives a numerical weight. With two healthy records both configured with weight 50, each record has an equal relative probability of being selected for a DNS response.

A 50/50 configuration does **not** mean DNS answers must alternate perfectly. Selection is probabilistic, and recursive DNS caching can cause the same result to appear repeatedly.

Weighted routing is useful for controlled traffic distribution, canary releases, blue/green migrations, A/B-style infrastructure testing, and gradual traffic shifts.

## DNS TTL and Caching
The records used a TTL of **60 seconds**. TTL tells recursive DNS resolvers approximately how long they may cache a DNS answer before requesting fresh information.

Caching explains why repeated queries can temporarily return the same EC2 address even when both records are healthy. During testing, queries were sent explicitly to Cloudflare's resolver (`1.1.1.1`) to make the resolver being tested clear.

```powershell
nslookup weighted.kelechicloud.com 1.1.1.1
```

Successful testing returned both backend addresses across queries:

```text
18.132.203.16
13.40.222.71
```

## EC2 and Apache Configuration
Amazon Linux 2023 was used on the EC2 instances. Apache was installed, started, and enabled so that the service starts automatically after boot.

```bash
sudo yum update -y
sudo yum install -y httpd
sudo systemctl start httpd
sudo systemctl enable httpd
sudo systemctl status httpd
```

The individual pages identified which server answered the request, for example **London Web Server 1 / EC2-1** and **London Web Server 2 / EC2-2**. This made routing behaviour easy to verify in a browser.

## Bash Setup Script
The core Bash automation used to solve the server-configuration part of the problem can be represented as follows. Run an appropriately customised copy on each EC2 instance and change the server/AZ labels for that instance.

```bash
#!/bin/bash
set -e

# Update packages and install Apache
sudo yum update -y
sudo yum install -y httpd

# Start Apache now and at boot
sudo systemctl enable --now httpd

# Create a recognisable web page for this backend
sudo tee /var/www/html/index.html > /dev/null <<'HTML'
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>KelechiCloud - London Web Server 1</title>
</head>
<body>
  <h1>KelechiCloud</h1>
  <h2>London Web Server 1</h2>
  <p>Route 53 DNS Routing Project</p>
  <p>Region: Europe (London) - eu-west-2</p>
  <p>Availability Zone: eu-west-2a</p>
  <p>Server: EC2-1</p>
</body>
</html>
HTML

sudo systemctl restart httpd
sudo systemctl status httpd --no-pager
```

For EC2-2, the page was changed to identify **London Web Server 2**, its Availability Zone, and **EC2-2**.

## Route 53 Configuration
Two A records were created with the same hostname:

| Record | Destination | Policy | Weight | TTL |
|---|---|---|---:|---:|
| `weighted.kelechicloud.com` | EC2-1 | Weighted | 50 | 60 s |
| `weighted.kelechicloud.com` | EC2-2 | Weighted | 50 | 60 s |

Each record was given a unique Record ID and associated with the health check for its corresponding EC2 endpoint.

## Health Checks
Route 53 HTTP endpoint health checks monitored port 80 on each server. During normal operation, both health checks displayed **Healthy**.

The lab used the normal failure threshold of three consecutive failed checks. This prevents a single transient failure from immediately declaring an endpoint unhealthy.

The health-check logic is:

```text
Route 53 health checker
        |
        | HTTP request :80
        v
EC2 / Apache endpoint
        |
        +-- successful response --> Healthy
        |
        +-- repeated failures ----> Unhealthy
```

## Failure and Recovery Test
A controlled failure was introduced by stopping Apache on one EC2 instance:

```bash
sudo systemctl stop httpd
sudo systemctl status httpd
```

After the required failed health checks, Route 53 showed that endpoint as **Unhealthy**, while the second endpoint remained **Healthy**. DNS testing then showed the healthy endpoint being returned instead of the failed endpoint.

Apache was restored with:

```bash
sudo systemctl start httpd
sudo systemctl status httpd
```

Route 53 subsequently detected recovery and both health checks returned to **Healthy**. Repeating DNS queries then demonstrated that both EC2 IP addresses could again be returned.

## Commands Used

### SSH to an EC2 instance
```powershell
ssh -i "Devops-TGW-London-Key-Key.pem" ec2-user@<EC2-PUBLIC-IP>
```

### Apache service management
```bash
sudo systemctl status httpd
sudo systemctl stop httpd
sudo systemctl start httpd
sudo systemctl restart httpd
sudo systemctl enable httpd
```

### DNS testing
```powershell
nslookup weighted.kelechicloud.com 1.1.1.1
```

### Exit SSH
```bash
exit
```

## Testing Evidence
The implementation was validated at several layers:

1. Each EC2 public IP successfully displayed its own Apache web page.
2. `weighted.kelechicloud.com` resolved to both backend IP addresses during healthy operation.
3. Both Route 53 health checks reported **Healthy** initially.
4. Apache was stopped on one endpoint to simulate application failure.
5. The corresponding health check changed to **Unhealthy** while the other remained healthy.
6. DNS queries returned the remaining healthy endpoint during the failure test.
7. Apache was restarted and the failed health check recovered to **Healthy**.
8. Subsequent DNS queries again returned both backend addresses.

## Security Decisions
- HTTP port 80 was opened so users and Route 53 health checkers could reach the web service.
- SSH port 22 should be restricted to a trusted administrator IP/CIDR rather than exposed to the whole internet.
- The EC2 key pair remains local and must never be committed to GitHub.
- No AWS access keys, passwords, tokens, or other secrets are stored in the website or repository.
- For a production application, HTTPS/TLS, least-privilege IAM, monitoring/logging, patch management, and tighter network controls should be added.

## Troubleshooting and Lessons Learned
### Repeated DNS answers
Repeated `nslookup` results do not prove weighted routing is broken. DNS caching and probabilistic weighted selection can produce the same answer several times. The project therefore used repeated tests and an explicitly selected resolver.

### Health check remains unhealthy
Verify that Apache is running and listening on port 80:

```bash
sudo systemctl status httpd
```

Also confirm that the EC2 security group permits the health-check traffic required by the design and that the endpoint returns a successful HTTP response.

### Website unavailable by public IP
Check the instance state, public IP, route to the Internet Gateway, subnet configuration, security-group HTTP rule, and Apache service status.

### SSH connection fails
Check the correct `.pem` key, username (`ec2-user` for this Amazon Linux configuration), instance public IP, and the security group's SSH source rule.

## Skills Demonstrated
- Amazon Route 53 public DNS
- Weighted routing policies
- DNS TTL and caching behaviour
- Route 53 endpoint health checks
- Failure detection and recovery testing
- Amazon EC2 administration
- Amazon Linux 2023
- Apache HTTP Server
- VPC, subnet and security-group fundamentals
- SSH and Linux service management with `systemctl`
- Bash automation
- DNS troubleshooting with `nslookup`
- High-availability concepts
- Practical incident simulation and validation
- Technical documentation and architecture communication

## What This Project Proves
The project demonstrates more than simply launching EC2 instances. It shows the ability to design, configure, test and troubleshoot a health-aware DNS architecture. A backend failure was deliberately introduced, observed through Route 53 health checks, handled through DNS routing, and then recovered and verified.

## Production Improvements
A production version could add HTTPS with ACM, an Application Load Balancer where request-level load balancing is required, Auto Scaling, CloudWatch alarms/logging, infrastructure as code with Terraform, CI/CD, and Elastic IPs or load-balancer targets to avoid dependence on changing EC2 public IP addresses.

## Conclusion
The lab successfully demonstrated **Amazon Route 53 weighted routing combined with health checks** across two Apache EC2 web servers. Under normal conditions, both healthy endpoints participated in DNS routing. During a simulated Apache failure, the affected endpoint became unhealthy and the healthy backend continued serving as the valid DNS destination. After recovery, both endpoints returned to service.

This provides hands-on evidence of DNS routing, availability engineering, health monitoring, Linux administration, failure testing and troubleshooting in AWS.
