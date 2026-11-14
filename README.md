# AWS VPC Watchdog
A two-tier VPC with a public web server, an isolated private server,
and a serverless uptime monitor. Built by hand in the AWS console.

## Architecture
![Resource map](screenshots/resource-map.png)

## What I built
- Custom VPC (10.0.0.0/16), 4 subnets across 2 AZs, IGW, public and private route tables
- EC2 web server (Apache via user data) in a public subnet
- EC2 private server reachable only through the web server's security group
- Lambda health check every 5 minutes via EventBridge, logs in CloudWatch

## Key decisions
- No NAT Gateway: private server needs no internet, which keeps cost near $0
- Lambda outside the VPC: tests the site the way a real user would
- Security-group referencing instead of IP rules for east-west traffic

## Failure testing
### Test 1: Stopped Apache

#### Break it: sudo systemctl stop httpd

{
  "check": "DOWN",
  "url": "http://3.147.57.157/health",
  "status_code": null,
  "latency_ms": 15,
  "detail": "<urlopen error [Errno 111] Connection refused>"
}

#### Fix it: sudo systemctl start httpd

{
  "check": "UP",
  "url": "http://3.147.57.157/health",
  "status_code": 200,
  "latency_ms": 16,
  "detail": "OK"
}

### Test 2: Remove Security Group Rule

#### Break it: Delete rule (ec2 security group) -- HTTP/80/0.0.0.0/0

{
  "check": "DOWN",
  "url": "http://3.147.57.157/health",
  "status_code": null,
  "latency_ms": 5025,
  "detail": "<urlopen error timed out>"
}

#### Fix it: Add rule (ec2 security group) -- HTTP/80/0.0.0.0/0

{
  "check": "UP",
  "url": "http://3.147.57.157/health",
  "status_code": 200,
  "latency_ms": 30,
  "detail": "OK"
}

### Test 3: Broke Route Table; Cut the Route to the Internet

#### Break it: Move wd-public-a route table off the public one

{
  "check": "DOWN",
  "url": "http://3.147.57.157/health",
  "status_code": null,
  "latency_ms": 5005,
  "detail": "<urlopen error timed out>"
}

#### Fix it: Move wd-public-a route table back to public

{
  "check": "UP",
  "url": "http://3.147.57.157/health",
  "status_code": 200,
  "latency_ms": 5,
  "detail": "OK"
}
