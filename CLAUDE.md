# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an AWS Multi-CDN Control Lab demonstrating a Hybrid Traffic Control Architecture. It solves the latency issues inherent in DNS-based failover by combining Server-Side traffic steering (Route 53) with Client-Side dynamic routing (HTTPDNS/Bootstrap Config).

The lab addresses the "Propagation Lag" problem where traditional DNS failover can take from several minutes to over 24 hours for users to switch from a failed primary CDN to a backup CDN due to TTL caching and ISP behavior.

## Architecture

The project implements a dual-layer approach:
- **Server-Side (Safety Net)**: AWS Route 53 with Application Recovery Controller (ARC) for broad, region-based traffic shaping
- **Client-Side (Speed)**: A "Bootstrap Config" pattern where the App/Web client asks "Where should I go?" before making requests, allowing for instant switching

### Directory Structure
```
aws-multi-cdn-control-lab/
├── infrastructure/    # Pulumi with Go IaC for Route 53, ARC, ALB, and Mock CDNs
├── control-plane/     # EC2 instances for the Config API behind ALB
├── client-app/        # React Web App demonstrating the "Smart Client"
└── simulation/        # Python scripts to break endpoints and flip switches
```

## Prerequisites

- AWS CLI (configured with Administrator access for the lab)
- Pulumi CLI
- Go 1.19+ (for infrastructure code)
- Node.js v16+ (for the client)
- Python 3.9+ (for simulation scripts)
- Domain: Access to zzhe.xyz domain for creating lab subdomains (*.cloudfront.lab.zzhe.xyz)

## Common Commands

### Infrastructure Deployment
```bash
cd infrastructure
pulumi stack init
pulumi up
```

### Client Application
```bash
cd client-app
# Edit .env or src/config.ts with the cloudfront.lab.zzhe.xyz URLs from Pulumi output
npm install
npm run dev
```

### Simulation Scripts
```bash
# Simulate technical failure (make primary return 503 errors)
python3 simulation/break_primary.py

# Manual kill switch via Route 53 ARC
python3 simulation/toggle_arc.py --state OFF
```

### Cleanup
```bash
cd infrastructure
pulumi destroy
```

## Development Workflow

1. Deploy infrastructure first to get the ALB URLs (*.cloudfront.lab.zzhe.xyz)
2. Configure the client app with the URLs from Pulumi output
3. Start the client application
4. Use simulation scripts to test failover scenarios

## Cost Considerations

- Route 53 ARC can be expensive ($195/mo/cluster) - this lab uses standard Route 53 Health Checks + Inverted Logic for cost optimization
- ALB + EC2: Costs for Application Load Balancers and EC2 instances running the mock services
- CloudWatch incurs small costs for alarms and metrics
- Always run `pulumi destroy` when finished to avoid ongoing charges

## Architecture Overview

### Traffic Flow Components

- **Mock Primary CDN**: Simulated via ALB A with EC2 instances running mock Cloudflare service
- **Mock Secondary CDN**: Simulated via ALB B with EC2 instances running mock CloudFront service
- **Control Plane**: Highly available API running on EC2 instances behind ALB that reads Route 53 ARC state
- **Client Application**: React web app that fetches configuration and implements smart failover logic

### Key Infrastructure Components

- **Application Load Balancers (ALB)**: Distribute traffic across EC2 instances for high availability
- **EC2 Instances**: Host the mock CDN services and control plane API
- **Route 53**: DNS management and health checks for server-side failover
- **Route 53 ARC**: Application Recovery Controller for emergency traffic switching
- **CloudWatch**: Monitoring and alerting for system health

## Notes for Future Development

- This repository appears to be related to AWS CDN (Content Delivery Network) management and control
- Consider establishing a clear project structure with appropriate directories for source code, configuration, documentation, and tests
- Add build/deployment scripts and configuration files as the project develops
- Update this CLAUDE.md file as the codebase grows to include specific commands and architectural guidance