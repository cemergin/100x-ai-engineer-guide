<!--
  CHAPTER: 55
  TITLE: Sandboxing & Isolating Agents
  PART: 11 — AI Deployment & Infrastructure
  PHASE: 2 — Become an Expert
  PREREQS: Ch 9 (agent loop), Ch 30 (tool execution & permissions), Ch 48 (security & guardrails), Ch 53 (deployment)
  KEY_TOPICS: agent sandboxing, VPC isolation, egress filtering, proxy restrictions, Docker isolation, file system isolation, permission boundaries, Terraform infrastructure, the OpenClaw pattern, lethal trifecta prevention
  DIFFICULTY: Advanced
  LANGUAGE: TypeScript, Python, Terraform, Docker, YAML
  UPDATED: 2026-04-10
-->

# Chapter 55: Sandboxing & Isolating Agents

> **Part 11 — AI Deployment & Infrastructure** | Phase 2: Become an Expert | Prerequisites: Ch 9, Ch 30, Ch 48, Ch 53 | Difficulty: Advanced | Language: TypeScript/Python

An LLM that generates text is safe. It can hallucinate, say something inappropriate, or produce wrong answers -- but it cannot *do* anything. The text goes to a user who decides what to do with it.

An agent is different. An agent has tools. It can read files, search the web, call APIs, create database records, send emails, and execute code. When you give an LLM hands, you give it the ability to cause real harm. A prompt injection that tricks a chatbot into saying something wrong is embarrassing. A prompt injection that tricks an agent into exfiltrating customer data, deleting production records, or sending unauthorized emails is a security incident.

In Chapter 48, you learned about the **lethal trifecta**: the three conditions that, when combined, create catastrophic risk:

1. **Access to private data** -- the agent can see sensitive information
2. **Ability to take actions** -- the agent can do things with real consequences
3. **Exposure to untrusted input** -- the agent processes data it did not generate

The application-level defenses from Chapter 48 -- input validation, output filtering, permission checks -- are necessary but insufficient. A sufficiently clever prompt injection can bypass application-level controls. The only defense that survives prompt injection is **infrastructure-level isolation**: the agent literally *cannot* reach production systems because the network won't let it. It *cannot* exfiltrate data because egress is blocked. It *cannot* take destructive actions because it runs with read-only permissions in an isolated environment.

This chapter teaches you how to build that infrastructure. The centerpiece is the **OpenClaw pattern** -- how the Nelo engineering team deployed a production agent with zero access to production systems, zero inbound internet access, and egress restricted to an explicit allowlist of domains.

### In This Chapter
- The OpenClaw pattern: a real-world agent sandboxing case study
- AWS account-level isolation for agent workloads
- VPC endpoints: accessing services without public internet
- Proxy-restricted internet access with domain allowlists
- Docker isolation for agent execution
- File system isolation: ephemeral workspaces, read-only mounts, tmpfs
- Network-level controls: security groups, NACLs, egress filtering
- The lethal trifecta revisited: how infrastructure prevents each leg
- Permission boundaries that survive prompt injection
- Complete Terraform examples for sandboxed agent infrastructure
- Executing AI-generated code safely: Vercel Sandbox, E2B, Docker

### Related Chapters
- **Ch 9 (The Agent Loop)** -- the agent pattern you are now isolating
- **Ch 22 (Telemetry & Tracing)** -- spirals back: sandbox audit logs feed the telemetry pipeline
- **Ch 30 (Tool Execution & Permissions)** -- application-level permissions; now infrastructure-level
- **Ch 48 (AI Security & Guardrails)** -- the lethal trifecta; now prevent it with infrastructure
- **Ch 53 (Deploying LLM Applications)** -- you deployed the app; now isolate the dangerous parts
- **Ch 54 (API Gateway)** -- gateway controls complement sandboxing controls
- **Ch 59 (Building Internal AI Tools)** -- sandboxing patterns for platform tools

---

## 1. The OpenClaw Pattern

### 1.1 The Problem

The Nelo engineering team needed to deploy an agent called OpenClaw. This agent could interact with Linear (their project management tool), read and write code, and perform autonomous tasks. It was useful. It was also dangerous.

The risks were concrete:
- The agent could be tricked into reading production data via prompt injection
- A malicious user could use the agent as a proxy to attack internal systems
- A bug in the agent's tool implementation could accidentally modify production databases
- The agent's internet access could be used to exfiltrate sensitive data to an external server

The team needed the agent to be productive -- it had to access Linear, write code, and communicate results -- but it could not have any access to production systems, customer data, or unrestricted internet.

### 1.2 The Architecture

Here is exactly how OpenClaw was deployed:

```
┌─────────────────────────────────────────────────────────┐
│                   AWS Account: Production                │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌─────────────────────┐  │
│  │ Web App  │  │ Database │  │ Customer Data        │  │
│  │ (ECS)    │  │ (RDS)    │  │ (S3)                 │  │
│  └──────────┘  └──────────┘  └─────────────────────┘  │
│                                                         │
│         ▲ ZERO connectivity from agent account          │
│         │                                               │
└─────────┼───────────────────────────────────────────────┘
          │ (no VPC peering, no transit gateway,
          │  no cross-account roles)
          │
          ╳ ← This connection does not exist
          │
┌─────────┼───────────────────────────────────────────────┐
│         │       AWS Account: Agent Sandbox               │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │                 VPC (10.0.0.0/16)                │   │
│  │                                                   │   │
│  │  ┌──────────────────────────────────────────┐    │   │
│  │  │          Private Subnet (10.0.1.0/24)     │    │   │
│  │  │                                           │    │   │
│  │  │  ┌─────────────┐   ┌──────────────────┐  │    │   │
│  │  │  │  OpenClaw    │   │  Egress Proxy    │  │    │   │
│  │  │  │  Agent       │──→│  (Squid)         │  │    │   │
│  │  │  │  (ECS Task)  │   │                  │  │    │   │
│  │  │  └─────────────┘   │  Allowlist:       │  │    │   │
│  │  │         │           │  - linear.app     │  │    │   │
│  │  │         │           │  - api.linear.app │  │    │   │
│  │  │         ▼           │  - slack.com      │  │    │   │
│  │  │  ┌─────────────┐   │  - api.anthropic. │  │    │   │
│  │  │  │  VPC         │   │    com            │  │    │   │
│  │  │  │  Endpoint    │   └──────────────────┘  │    │   │
│  │  │  │  (S3, ECR,   │          │              │    │   │
│  │  │  │   Secrets     │          │              │    │   │
│  │  │  │   Manager)    │          ▼              │    │   │
│  │  │  └─────────────┘   ┌──────────────────┐   │    │   │
│  │  │                    │  NAT Gateway      │   │    │   │
│  │  │                    │  (for proxy only) │   │    │   │
│  │  │                    └──────────────────┘   │    │   │
│  │  └───────────────────────────────────────────┘    │   │
│  │                                                   │   │
│  │  NO public subnets. NO internet gateway.          │   │
│  │  NOT internet-addressable.                        │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 1.3 The Five Isolation Principles

The OpenClaw deployment enforces five principles:

**Principle 1: Separate AWS account.** The agent runs in its own AWS account with no cross-account access to production. There is no VPC peering, no transit gateway connection, no IAM roles that allow cross-account access. The agent literally cannot reach production resources because the network path does not exist.

**Principle 2: VPC endpoint access only.** AWS services (S3, ECR, Secrets Manager) are accessed through VPC endpoints -- private connections that never traverse the public internet. The agent can pull its container image from ECR and read its API keys from Secrets Manager without any internet access.

**Principle 3: Proxy-restricted internet.** The agent needs internet access -- it must call the Anthropic API and the Linear API. But instead of giving it unrestricted internet, all outbound traffic goes through a forward proxy (Squid) that only allows requests to an explicit allowlist of domains.

**Principle 4: Not internet-addressable.** The agent has no public IP, no load balancer, no API Gateway endpoint. It cannot be reached from the internet. It can only be triggered internally (via SQS, EventBridge, or a VPC endpoint from a trusted account).

**Principle 5: Ephemeral execution.** Each agent invocation runs in a fresh container. There is no persistent state between invocations (beyond what is explicitly stored in S3 or a database within the sandbox account). A compromised agent session cannot persist across invocations.

---

## 2. Building the Sandbox Infrastructure

### 2.1 The Sandbox AWS Account (Terraform)

```hcl
# terraform/main.tf — sandbox account infrastructure
# This runs in the AGENT SANDBOX account, NOT production

terraform {
  required_version = ">= 1.5"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket = "agent-sandbox-terraform-state"
    key    = "infrastructure/terraform.tfstate"
    region = "us-east-1"
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project     = "agent-sandbox"
      Environment = var.environment
      ManagedBy   = "terraform"
    }
  }
}

variable "aws_region" {
  default = "us-east-1"
}

variable "environment" {
  default = "production"
}

variable "allowed_egress_domains" {
  type = list(string)
  default = [
    ".linear.app",
    ".api.linear.app",
    ".slack.com",
    ".api.anthropic.com",
  ]
  description = "Domains the agent is allowed to reach via the egress proxy"
}
```

### 2.2 VPC with No Internet Gateway

```hcl
# terraform/vpc.tf — VPC with NO internet gateway
# The agent cannot reach the internet directly

resource "aws_vpc" "agent_sandbox" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "agent-sandbox-vpc"
  }
}

# Private subnets only — NO public subnets
resource "aws_subnet" "private_a" {
  vpc_id                  = aws_vpc.agent_sandbox.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "${var.aws_region}a"
  map_public_ip_on_launch = false  # Critical: no public IPs

  tags = {
    Name = "agent-sandbox-private-a"
  }
}

resource "aws_subnet" "private_b" {
  vpc_id                  = aws_vpc.agent_sandbox.id
  cidr_block              = "10.0.2.0/24"
  availability_zone       = "${var.aws_region}b"
  map_public_ip_on_launch = false

  tags = {
    Name = "agent-sandbox-private-b"
  }
}

# Subnet for the egress proxy — this one gets NAT gateway access
resource "aws_subnet" "proxy" {
  vpc_id                  = aws_vpc.agent_sandbox.id
  cidr_block              = "10.0.10.0/24"
  availability_zone       = "${var.aws_region}a"
  map_public_ip_on_launch = false

  tags = {
    Name = "agent-sandbox-proxy"
  }
}

# Public subnet ONLY for the NAT gateway (proxy egress)
resource "aws_subnet" "nat" {
  vpc_id                  = aws_vpc.agent_sandbox.id
  cidr_block              = "10.0.100.0/24"
  availability_zone       = "${var.aws_region}a"
  map_public_ip_on_launch = false

  tags = {
    Name = "agent-sandbox-nat"
  }
}

# Internet gateway — ONLY for the NAT gateway, NOT for agent subnets
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.agent_sandbox.id

  tags = {
    Name = "agent-sandbox-igw"
  }
}

# NAT gateway — provides internet to the proxy subnet ONLY
resource "aws_eip" "nat" {
  domain = "vpc"
}

resource "aws_nat_gateway" "nat" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.nat.id

  tags = {
    Name = "agent-sandbox-nat"
  }
}

# Route table for the NAT subnet — routes to internet gateway
resource "aws_route_table" "nat" {
  vpc_id = aws_vpc.agent_sandbox.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = {
    Name = "agent-sandbox-nat-rt"
  }
}

resource "aws_route_table_association" "nat" {
  subnet_id      = aws_subnet.nat.id
  route_table_id = aws_route_table.nat.id
}

# Route table for proxy subnet — routes to NAT gateway
resource "aws_route_table" "proxy" {
  vpc_id = aws_vpc.agent_sandbox.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.nat.id
  }

  tags = {
    Name = "agent-sandbox-proxy-rt"
  }
}

resource "aws_route_table_association" "proxy" {
  subnet_id      = aws_subnet.proxy.id
  route_table_id = aws_route_table.proxy.id
}

# Route table for agent subnets — NO route to internet
# Only routes to VPC endpoints and the proxy
resource "aws_route_table" "agent" {
  vpc_id = aws_vpc.agent_sandbox.id

  # No 0.0.0.0/0 route — no internet access
  # VPC endpoint routes are added automatically

  tags = {
    Name = "agent-sandbox-agent-rt"
  }
}

resource "aws_route_table_association" "private_a" {
  subnet_id      = aws_subnet.private_a.id
  route_table_id = aws_route_table.agent.id
}

resource "aws_route_table_association" "private_b" {
  subnet_id      = aws_subnet.private_b.id
  route_table_id = aws_route_table.agent.id
}
```

### 2.3 VPC Endpoints

```hcl
# terraform/vpc-endpoints.tf — private access to AWS services
# VPC endpoints let the agent use AWS services without internet

# S3 Gateway Endpoint (free)
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.agent_sandbox.id
  service_name      = "com.amazonaws.${var.aws_region}.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = [aws_route_table.agent.id]

  tags = {
    Name = "agent-sandbox-s3-endpoint"
  }
}

# ECR Docker endpoint (for pulling container images)
resource "aws_vpc_endpoint" "ecr_dkr" {
  vpc_id              = aws_vpc.agent_sandbox.id
  service_name        = "com.amazonaws.${var.aws_region}.ecr.dkr"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = [aws_subnet.private_a.id, aws_subnet.private_b.id]
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true

  tags = {
    Name = "agent-sandbox-ecr-dkr-endpoint"
  }
}

# ECR API endpoint
resource "aws_vpc_endpoint" "ecr_api" {
  vpc_id              = aws_vpc.agent_sandbox.id
  service_name        = "com.amazonaws.${var.aws_region}.ecr.api"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = [aws_subnet.private_a.id, aws_subnet.private_b.id]
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true

  tags = {
    Name = "agent-sandbox-ecr-api-endpoint"
  }
}

# Secrets Manager endpoint (for API keys)
resource "aws_vpc_endpoint" "secrets_manager" {
  vpc_id              = aws_vpc.agent_sandbox.id
  service_name        = "com.amazonaws.${var.aws_region}.secretsmanager"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = [aws_subnet.private_a.id, aws_subnet.private_b.id]
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true

  tags = {
    Name = "agent-sandbox-secrets-endpoint"
  }
}

# CloudWatch Logs endpoint (for logging)
resource "aws_vpc_endpoint" "logs" {
  vpc_id              = aws_vpc.agent_sandbox.id
  service_name        = "com.amazonaws.${var.aws_region}.logs"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = [aws_subnet.private_a.id, aws_subnet.private_b.id]
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true

  tags = {
    Name = "agent-sandbox-logs-endpoint"
  }
}

# SQS endpoint (for trigger queue)
resource "aws_vpc_endpoint" "sqs" {
  vpc_id              = aws_vpc.agent_sandbox.id
  service_name        = "com.amazonaws.${var.aws_region}.sqs"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = [aws_subnet.private_a.id, aws_subnet.private_b.id]
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true

  tags = {
    Name = "agent-sandbox-sqs-endpoint"
  }
}

# Security group for VPC endpoints
resource "aws_security_group" "vpc_endpoints" {
  name_prefix = "agent-sandbox-vpce-"
  vpc_id      = aws_vpc.agent_sandbox.id

  ingress {
    from_port       = 443
    to_port         = 443
    protocol        = "tcp"
    security_groups = [aws_security_group.agent_task.id]
    description     = "HTTPS from agent tasks"
  }

  tags = {
    Name = "agent-sandbox-vpce-sg"
  }
}
```

### 2.4 The Egress Proxy

```hcl
# terraform/proxy.tf — forward proxy with domain allowlist

resource "aws_security_group" "proxy" {
  name_prefix = "agent-sandbox-proxy-"
  vpc_id      = aws_vpc.agent_sandbox.id

  # Allow inbound from agent tasks on proxy port
  ingress {
    from_port       = 3128
    to_port         = 3128
    protocol        = "tcp"
    security_groups = [aws_security_group.agent_task.id]
    description     = "Proxy traffic from agent tasks"
  }

  # Allow outbound to internet (through NAT gateway)
  egress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTPS to allowed domains"
  }

  # Allow outbound DNS
  egress {
    from_port   = 53
    to_port     = 53
    protocol    = "udp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "DNS resolution"
  }

  tags = {
    Name = "agent-sandbox-proxy-sg"
  }
}

# ECS service for the Squid proxy
resource "aws_ecs_service" "proxy" {
  name            = "agent-proxy"
  cluster         = aws_ecs_cluster.agent_sandbox.id
  task_definition = aws_ecs_task_definition.proxy.arn
  desired_count   = 2  # HA: run 2 proxy instances
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = [aws_subnet.proxy.id]
    security_groups  = [aws_security_group.proxy.id]
    assign_public_ip = false
  }

  service_registries {
    registry_arn = aws_service_discovery_service.proxy.arn
  }
}

# Service discovery so the agent can find the proxy
resource "aws_service_discovery_private_dns_namespace" "agent" {
  name = "agent.internal"
  vpc  = aws_vpc.agent_sandbox.id
}

resource "aws_service_discovery_service" "proxy" {
  name = "proxy"

  dns_config {
    namespace_id = aws_service_discovery_private_dns_namespace.agent.id

    dns_records {
      ttl  = 10
      type = "A"
    }
  }

  health_check_custom_config {
    failure_threshold = 1
  }
}

# Proxy task definition
resource "aws_ecs_task_definition" "proxy" {
  family                   = "agent-proxy"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = 512
  memory                   = 1024
  execution_role_arn       = aws_iam_role.ecs_execution.arn

  container_definitions = jsonencode([
    {
      name  = "squid-proxy"
      image = "${aws_ecr_repository.proxy.repository_url}:latest"
      portMappings = [
        {
          containerPort = 3128
          protocol      = "tcp"
        }
      ]
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.proxy.name
          "awslogs-region"        = var.aws_region
          "awslogs-stream-prefix" = "proxy"
        }
      }
      healthCheck = {
        command     = ["CMD-SHELL", "squid -k check || exit 1"]
        interval    = 30
        timeout     = 5
        retries     = 3
        startPeriod = 10
      }
    }
  ])
}
```

### 2.5 Squid Proxy Configuration

```
# squid/squid.conf — domain allowlist proxy configuration
# This is the ONLY path from the agent to the internet.
# Any domain not on this list is BLOCKED.

# Listening port
http_port 3128

# Access control: define the allowed domains
acl allowed_domains dstdomain .linear.app
acl allowed_domains dstdomain .api.linear.app
acl allowed_domains dstdomain .slack.com
acl allowed_domains dstdomain .api.anthropic.com

# Only allow HTTPS CONNECT to allowed domains
acl SSL_ports port 443
acl CONNECT method CONNECT

# Deny everything except allowed domains over HTTPS
http_access allow CONNECT SSL_ports allowed_domains
http_access deny all

# Security hardening
forwarded_for delete          # Don't leak internal IPs
via off                       # Don't identify as proxy
request_header_access X-Forwarded-For deny all

# Logging for audit trail
access_log /var/log/squid/access.log squid
cache_log /var/log/squid/cache.log

# No caching (we don't want to cache API responses)
cache deny all

# Connection limits
maximum_object_size 10 MB
client_lifetime 5 minutes
connect_timeout 30 seconds
read_timeout 5 minutes
```

```dockerfile
# squid/Dockerfile — the proxy container
FROM ubuntu:22.04

RUN apt-get update && \
    apt-get install -y squid && \
    rm -rf /var/lib/apt/lists/*

COPY squid.conf /etc/squid/squid.conf

# Create log directory
RUN mkdir -p /var/log/squid && \
    chown proxy:proxy /var/log/squid

EXPOSE 3128

# Health check
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD squid -k check || exit 1

CMD ["squid", "-N", "-f", "/etc/squid/squid.conf"]
```

---

## 3. The Agent Container

### 3.1 Security Group: Agent Cannot Reach Internet

```hcl
# terraform/agent.tf — the agent ECS task

resource "aws_security_group" "agent_task" {
  name_prefix = "agent-sandbox-task-"
  vpc_id      = aws_vpc.agent_sandbox.id

  # NO ingress rules — the agent is not internet-addressable
  # It receives work from SQS (via VPC endpoint)

  # Egress: ONLY to the proxy and VPC endpoints
  egress {
    from_port       = 3128
    to_port         = 3128
    protocol        = "tcp"
    security_groups = [aws_security_group.proxy.id]
    description     = "To egress proxy"
  }

  egress {
    from_port       = 443
    to_port         = 443
    protocol        = "tcp"
    security_groups = [aws_security_group.vpc_endpoints.id]
    description     = "To VPC endpoints (AWS services)"
  }

  # DNS resolution within VPC
  egress {
    from_port   = 53
    to_port     = 53
    protocol    = "udp"
    cidr_blocks = [aws_vpc.agent_sandbox.cidr_block]
    description = "DNS within VPC"
  }

  tags = {
    Name = "agent-sandbox-task-sg"
  }
}
```

### 3.2 Agent Task Definition

```hcl
# terraform/agent-task.tf — ECS task definition for the agent

resource "aws_ecs_cluster" "agent_sandbox" {
  name = "agent-sandbox"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }
}

resource "aws_ecs_task_definition" "agent" {
  family                   = "openclaw-agent"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = 1024    # 1 vCPU
  memory                   = 2048    # 2 GB
  execution_role_arn       = aws_iam_role.ecs_execution.arn
  task_role_arn            = aws_iam_role.agent_task.arn

  container_definitions = jsonencode([
    {
      name  = "agent"
      image = "${aws_ecr_repository.agent.repository_url}:latest"

      environment = [
        {
          name  = "HTTP_PROXY"
          value = "http://proxy.agent.internal:3128"
        },
        {
          name  = "HTTPS_PROXY"
          value = "http://proxy.agent.internal:3128"
        },
        {
          name  = "NO_PROXY"
          value = "169.254.169.254,169.254.170.2,.amazonaws.com"
        },
        {
          name  = "NODE_ENV"
          value = "production"
        }
      ]

      secrets = [
        {
          name      = "ANTHROPIC_API_KEY"
          valueFrom = aws_secretsmanager_secret.anthropic_key.arn
        },
        {
          name      = "LINEAR_API_KEY"
          valueFrom = aws_secretsmanager_secret.linear_key.arn
        }
      ]

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.agent.name
          "awslogs-region"        = var.aws_region
          "awslogs-stream-prefix" = "agent"
        }
      }

      # Read-only root filesystem
      readonlyRootFilesystem = true

      # Ephemeral workspace for temporary files
      mountPoints = [
        {
          sourceVolume  = "workspace"
          containerPath = "/tmp/workspace"
          readOnly      = false
        },
        {
          sourceVolume  = "tmp"
          containerPath = "/tmp"
          readOnly      = false
        }
      ]

      # Resource limits
      ulimits = [
        {
          name      = "nofile"
          hardLimit = 1024
          softLimit = 1024
        }
      ]

      # No privileged access
      privileged = false

      # Drop all Linux capabilities
      linuxParameters = {
        capabilities = {
          drop = ["ALL"]
        }
        # Don't allow privilege escalation
        initProcessEnabled = true
      }
    }
  ])

  # Ephemeral volumes (not persistent)
  volume {
    name = "workspace"
  }

  volume {
    name = "tmp"
  }
}

# The agent task's IAM role — minimal permissions
resource "aws_iam_role" "agent_task" {
  name = "agent-sandbox-task-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ecs-tasks.amazonaws.com"
        }
      }
    ]
  })
}

# The agent can ONLY:
# 1. Read/write to its own S3 bucket (for results)
# 2. Send/receive SQS messages (for task queue)
# 3. Write CloudWatch logs
resource "aws_iam_policy" "agent_task" {
  name = "agent-sandbox-task-policy"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:ListBucket",
        ]
        Resource = [
          aws_s3_bucket.agent_results.arn,
          "${aws_s3_bucket.agent_results.arn}/*",
        ]
      },
      {
        Effect = "Allow"
        Action = [
          "sqs:ReceiveMessage",
          "sqs:DeleteMessage",
          "sqs:SendMessage",
          "sqs:GetQueueAttributes",
        ]
        Resource = [
          aws_sqs_queue.agent_tasks.arn,
          aws_sqs_queue.agent_results.arn,
        ]
      },
      {
        Effect = "Allow"
        Action = [
          "logs:CreateLogStream",
          "logs:PutLogEvents",
        ]
        Resource = "${aws_cloudwatch_log_group.agent.arn}:*"
      },
      # DENY everything else — defense in depth
      {
        Effect   = "Deny"
        Action   = "*"
        Resource = "*"
        Condition = {
          StringNotEquals = {
            "aws:RequestedRegion" = var.aws_region
          }
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "agent_task" {
  role       = aws_iam_role.agent_task.name
  policy_arn = aws_iam_policy.agent_task.arn
}
```

### 3.3 The Agent Application

```typescript
// src/agent.ts — the OpenClaw agent running inside the sandbox
import { generateText, tool } from "ai";
import { anthropic } from "@ai-sdk/anthropic";
import { z } from "zod";
import { SQSClient, ReceiveMessageCommand, DeleteMessageCommand } from "@aws-sdk/client-sqs";
import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";

// All HTTP requests go through the proxy (set via environment variables)
// HTTP_PROXY and HTTPS_PROXY are set in the ECS task definition
// The proxy only allows: linear.app, slack.com, api.anthropic.com

const sqs = new SQSClient({ region: process.env.AWS_REGION });
const s3 = new S3Client({ region: process.env.AWS_REGION });

async function processTask(task: any): Promise<string> {
  const result = await generateText({
    model: anthropic("claude-sonnet-4-20250514"),
    system: `You are OpenClaw, an engineering assistant. You can interact with 
Linear to manage issues and projects. You MUST NOT attempt to access any 
systems not explicitly provided as tools. You MUST NOT attempt to bypass 
the proxy or access unauthorized URLs.`,
    prompt: task.instruction,
    tools: {
      linearCreateIssue: tool({
        description: "Create a new issue in Linear",
        parameters: z.object({
          title: z.string().describe("Issue title"),
          description: z.string().describe("Issue description"),
          teamId: z.string().describe("Linear team ID"),
          priority: z.number().min(0).max(4).describe("Priority: 0=none, 1=urgent, 4=low"),
        }),
        execute: async ({ title, description, teamId, priority }) => {
          // This HTTP request goes through the Squid proxy
          // The proxy allows *.api.linear.app — this will succeed
          const response = await fetch("https://api.linear.app/graphql", {
            method: "POST",
            headers: {
              "Content-Type": "application/json",
              Authorization: process.env.LINEAR_API_KEY!,
            },
            body: JSON.stringify({
              query: `mutation { issueCreate(input: { title: "${title}", description: "${description}", teamId: "${teamId}", priority: ${priority} }) { success issue { id identifier url } } }`,
            }),
          });
          return response.json();
        },
      }),

      linearSearchIssues: tool({
        description: "Search for issues in Linear",
        parameters: z.object({
          query: z.string().describe("Search query"),
        }),
        execute: async ({ query }) => {
          const response = await fetch("https://api.linear.app/graphql", {
            method: "POST",
            headers: {
              "Content-Type": "application/json",
              Authorization: process.env.LINEAR_API_KEY!,
            },
            body: JSON.stringify({
              query: `query { issueSearch(query: "${query}") { nodes { id identifier title state { name } priority assignee { name } } } }`,
            }),
          });
          return response.json();
        },
      }),

      // This tool would FAIL because the proxy blocks it
      // Included to illustrate the security boundary
      // fetchArbitraryUrl: tool({
      //   description: "Fetch any URL",
      //   parameters: z.object({ url: z.string() }),
      //   execute: async ({ url }) => {
      //     // If url is not on the proxy allowlist, Squid returns 403
      //     const response = await fetch(url); // BLOCKED by proxy
      //     return response.text();
      //   },
      // }),
    },
    maxSteps: 10,
  });

  return result.text;
}

// Main loop: poll SQS for tasks
async function main() {
  console.log("OpenClaw agent starting...");
  console.log(`Proxy: ${process.env.HTTPS_PROXY}`);

  while (true) {
    try {
      const response = await sqs.send(
        new ReceiveMessageCommand({
          QueueUrl: process.env.TASK_QUEUE_URL!,
          MaxNumberOfMessages: 1,
          WaitTimeSeconds: 20, // Long polling
          VisibilityTimeout: 300, // 5 min to process
        })
      );

      if (!response.Messages || response.Messages.length === 0) {
        continue; // No messages, long poll again
      }

      for (const message of response.Messages) {
        const task = JSON.parse(message.Body!);
        console.log(`Processing task: ${task.id}`);

        try {
          const result = await processTask(task);

          // Store result in S3
          await s3.send(
            new PutObjectCommand({
              Bucket: process.env.RESULTS_BUCKET!,
              Key: `results/${task.id}.json`,
              Body: JSON.stringify({ taskId: task.id, result, completedAt: new Date().toISOString() }),
              ContentType: "application/json",
            })
          );

          // Delete from queue
          await sqs.send(
            new DeleteMessageCommand({
              QueueUrl: process.env.TASK_QUEUE_URL!,
              ReceiptHandle: message.ReceiptHandle!,
            })
          );

          console.log(`Task ${task.id} completed`);
        } catch (error) {
          console.error(`Task ${task.id} failed:`, error);
          // Message returns to queue after visibility timeout
        }
      }
    } catch (error) {
      console.error("Queue polling error:", error);
      await new Promise((r) => setTimeout(r, 5000));
    }
  }
}

main();
```

### 3.4 Agent Dockerfile

```dockerfile
# Dockerfile — the OpenClaw agent container
FROM node:20-slim AS builder

WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY tsconfig.json ./
COPY src/ ./src/
RUN npm run build

FROM node:20-slim AS production

WORKDIR /app

# Security: run as non-root
RUN addgroup --system agent && adduser --system --group agent

# Copy built application
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./

# Create writable tmp directory (root filesystem is read-only)
RUN mkdir -p /tmp/workspace && chown agent:agent /tmp/workspace

USER agent

# No EXPOSE — this container is not internet-addressable
# It pulls tasks from SQS and pushes results to S3

CMD ["node", "dist/agent.js"]
```

---

## 4. The Lethal Trifecta: Infrastructure-Level Prevention

### 4.1 Revisiting the Three Conditions

In Chapter 48, we introduced the lethal trifecta. Now we map each condition to its infrastructure-level defense:

```
LETHAL TRIFECTA                     INFRASTRUCTURE DEFENSE
──────────────                      ──────────────────────

1. Access to               ←───── Separate AWS account
   private data                    No VPC peering to production
                                   No cross-account IAM roles
                                   Agent cannot even ATTEMPT to
                                   reach production databases

2. Ability to               ←───── Read-only filesystem
   take actions                    Minimal IAM permissions
                                   Ephemeral containers
                                   Tool allowlist (app-level)
                                   Egress proxy (infra-level)

3. Exposure to              ←───── Not internet-addressable
   untrusted input                 Input only via SQS queue
                                   Input validation at queue consumer
                                   No public endpoints
```

### 4.2 Breaking Each Leg

**Breaking Leg 1: Access to Private Data**

The strongest defense against data exfiltration is ensuring the agent never has access to the data in the first place.

```hcl
# terraform/network-acl.tf — network-level defense against data access
# Even if the agent's security group is misconfigured,
# the NACL provides a second layer of protection.

resource "aws_network_acl" "agent_subnet" {
  vpc_id     = aws_vpc.agent_sandbox.id
  subnet_ids = [aws_subnet.private_a.id, aws_subnet.private_b.id]

  # Allow outbound to proxy subnet only
  egress {
    rule_no    = 100
    protocol   = "tcp"
    action     = "allow"
    cidr_block = aws_subnet.proxy.cidr_block
    from_port  = 3128
    to_port    = 3128
  }

  # Allow outbound to VPC endpoints (within VPC)
  egress {
    rule_no    = 200
    protocol   = "tcp"
    action     = "allow"
    cidr_block = aws_vpc.agent_sandbox.cidr_block
    from_port  = 443
    to_port    = 443
  }

  # Allow DNS within VPC
  egress {
    rule_no    = 300
    protocol   = "udp"
    action     = "allow"
    cidr_block = aws_vpc.agent_sandbox.cidr_block
    from_port  = 53
    to_port    = 53
  }

  # DENY everything else
  egress {
    rule_no    = 999
    protocol   = "-1"
    action     = "deny"
    cidr_block = "0.0.0.0/0"
    from_port  = 0
    to_port    = 0
  }

  # Allow inbound responses from proxy
  ingress {
    rule_no    = 100
    protocol   = "tcp"
    action     = "allow"
    cidr_block = aws_subnet.proxy.cidr_block
    from_port  = 1024
    to_port    = 65535
  }

  # Allow inbound from VPC (for VPC endpoints)
  ingress {
    rule_no    = 200
    protocol   = "tcp"
    action     = "allow"
    cidr_block = aws_vpc.agent_sandbox.cidr_block
    from_port  = 1024
    to_port    = 65535
  }

  # DENY all other inbound
  ingress {
    rule_no    = 999
    protocol   = "-1"
    action     = "deny"
    cidr_block = "0.0.0.0/0"
    from_port  = 0
    to_port    = 0
  }

  tags = {
    Name = "agent-sandbox-nacl"
  }
}
```

**Breaking Leg 2: Ability to Take Actions**

Even within the sandbox, limit what the agent can do:

```hcl
# terraform/permissions.tf — permission boundary for the agent account
# This is an IAM permission boundary applied to ALL roles in the account.
# Even if someone creates a new role, it cannot exceed these permissions.

resource "aws_iam_policy" "permission_boundary" {
  name = "agent-sandbox-permission-boundary"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      # Allow: basic ECS, S3, SQS, CloudWatch, Secrets Manager
      {
        Effect = "Allow"
        Action = [
          "ecs:*",
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage",
          "ecr:GetAuthorizationToken",
          "s3:GetObject",
          "s3:PutObject",
          "s3:ListBucket",
          "sqs:*",
          "logs:*",
          "secretsmanager:GetSecretValue",
        ]
        Resource = "*"
      },
      # DENY: anything that could affect other accounts
      {
        Effect = "Deny"
        Action = [
          "iam:CreateUser",
          "iam:CreateRole",
          "iam:AttachRolePolicy",
          "iam:PutRolePolicy",
          "organizations:*",
          "sts:AssumeRole",           # Cannot assume roles in other accounts
          "ec2:CreateVpc*",           # Cannot create VPC peering
          "ec2:AcceptVpcPeering*",
          "ec2:CreateTransitGateway*",
        ]
        Resource = "*"
      },
      # DENY: anything in other regions (limit blast radius)
      {
        Effect   = "Deny"
        Action   = "*"
        Resource = "*"
        Condition = {
          StringNotEquals = {
            "aws:RequestedRegion" = var.aws_region
          }
        }
      }
    ]
  })
}
```

**Breaking Leg 3: Exposure to Untrusted Input**

Control what data enters the agent:

```typescript
// src/input-validator.ts — validate and sanitize agent task input
import { z } from "zod";

// Strict schema for task input
const taskSchema = z.object({
  id: z.string().uuid(),
  instruction: z
    .string()
    .max(10_000) // Limit input size
    .refine(
      (text) => !containsSuspiciousPatterns(text),
      "Input contains suspicious patterns"
    ),
  context: z
    .object({
      teamId: z.string().regex(/^[a-zA-Z0-9-]+$/),
      userId: z.string().regex(/^[a-zA-Z0-9-]+$/),
      allowedActions: z.array(
        z.enum(["search_issues", "create_issue", "update_issue"])
      ),
    })
    .strict(), // No additional properties
  metadata: z
    .object({
      source: z.enum(["slack", "api", "scheduled"]),
      timestamp: z.string().datetime(),
    })
    .strict(),
});

function containsSuspiciousPatterns(text: string): boolean {
  const patterns = [
    // Prompt injection attempts
    /ignore\s+(previous|above|all)\s+instructions/i,
    /you\s+are\s+now\s+(a|an)\s+/i,
    /system\s*:\s*/i,
    /\]\s*\(\s*https?:\/\//i, // Markdown link injection
    // Data exfiltration attempts
    /curl\s+/i,
    /wget\s+/i,
    /nc\s+-/i, // netcat
    /\/etc\/passwd/i,
    /\.env/i,
    // Command injection
    /;\s*(rm|cat|ls|chmod|chown)\s+/i,
    /\|\s*(bash|sh|zsh)\s*/i,
  ];

  return patterns.some((p) => p.test(text));
}

export function validateTaskInput(rawInput: unknown): z.infer<typeof taskSchema> {
  return taskSchema.parse(rawInput);
}
```

### 4.3 Defense in Depth Summary

```
Layer           │ What It Prevents                         │ Survives Prompt Injection?
────────────────┼──────────────────────────────────────────┼─────────────────────────
Input validation│ Malformed input, obvious injection       │ Partially (patterns are bypassable)
App permissions │ Tool calls the agent shouldn't make      │ No (LLM can be tricked)
Container       │ File system access, privilege escalation │ Yes (read-only FS, no capabilities)
Security groups │ Network access to unauthorized services  │ Yes (enforced by AWS)
NACLs           │ Backup network restrictions              │ Yes (enforced by AWS)
Egress proxy    │ Data exfiltration to unauthorized domains│ Yes (enforced by Squid)
Separate account│ Access to production resources           │ Yes (no network path exists)
Permission bound│ IAM escalation, cross-account access     │ Yes (enforced by AWS)
```

The bottom five layers all survive prompt injection because they are enforced by infrastructure, not by the application. Even if an attacker completely controls the agent's behavior through prompt injection, the agent still cannot reach production systems, cannot exfiltrate data to unauthorized domains, and cannot escalate its permissions.

---

## 5. Docker Isolation for Local and Development Agents

### 5.1 Docker-Level Sandboxing

Not every agent needs a full AWS sandbox account. For local development and less critical workloads, Docker provides meaningful isolation.

```yaml
# docker-compose.sandbox.yml — sandboxed agent for development
version: "3.8"

services:
  agent:
    build:
      context: .
      dockerfile: Dockerfile
    # Read-only root filesystem
    read_only: true
    # Writable temp directories via tmpfs (in memory, limited size)
    tmpfs:
      - /tmp:size=100M,noexec,nosuid
      - /tmp/workspace:size=500M,noexec,nosuid
    # Drop all capabilities
    cap_drop:
      - ALL
    # No privileged mode
    privileged: false
    # Security options
    security_opt:
      - no-new-privileges:true
    # Resource limits
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 2G
        reservations:
          cpus: "0.5"
          memory: 512M
    # Network: only access the proxy
    networks:
      - agent-internal
    # Environment: route through proxy
    environment:
      - HTTP_PROXY=http://proxy:3128
      - HTTPS_PROXY=http://proxy:3128
      - NO_PROXY=localhost,127.0.0.1
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
    depends_on:
      - proxy

  proxy:
    build:
      context: ./squid
      dockerfile: Dockerfile
    networks:
      - agent-internal
      - external
    ports: []  # No ports exposed to host

networks:
  # Internal network: agent <-> proxy only
  agent-internal:
    internal: true  # No external connectivity
  # External network: proxy <-> internet
  external:
    driver: bridge
```

### 5.2 Seccomp Profiles for Agent Containers

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "comment": "Minimal seccomp profile for AI agent containers",
  "syscalls": [
    {
      "names": [
        "read", "write", "close", "fstat", "lseek", "mmap",
        "mprotect", "munmap", "brk", "rt_sigaction", "rt_sigprocmask",
        "ioctl", "access", "pipe", "select", "sched_yield",
        "mremap", "msync", "mincore", "madvise", "shmget",
        "socket", "connect", "sendto", "recvfrom", "sendmsg",
        "recvmsg", "bind", "listen", "getsockname", "getpeername",
        "setsockopt", "getsockopt", "clone", "execve", "exit",
        "wait4", "uname", "fcntl", "flock", "fsync", "fdatasync",
        "getcwd", "readlink", "stat", "lstat", "poll", "epoll_create",
        "epoll_ctl", "epoll_wait", "openat", "getdents64",
        "set_tid_address", "clock_gettime", "clock_getres",
        "exit_group", "epoll_create1", "pipe2", "eventfd2",
        "getrandom", "futex", "nanosleep", "getpid", "getuid",
        "getgid", "geteuid", "getegid", "sigaltstack",
        "arch_prctl", "set_robust_list", "prlimit64",
        "newfstatat", "rseq"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

```yaml
# In docker-compose, reference the seccomp profile:
services:
  agent:
    security_opt:
      - no-new-privileges:true
      - seccomp=./seccomp-agent.json
```

### 5.3 Vercel Sandbox for AI-Generated Code

When agents generate and execute code (like a coding assistant), you need even stronger isolation. Vercel's Sandbox service provides ephemeral Firecracker microVMs for running untrusted code.

```typescript
// lib/code-sandbox.ts — execute AI-generated code safely
// Using Vercel Sandbox for code execution isolation

interface SandboxResult {
  stdout: string;
  stderr: string;
  exitCode: number;
  timedOut: boolean;
}

async function executeInSandbox(
  code: string,
  language: "javascript" | "python",
  timeoutMs: number = 10_000
): Promise<SandboxResult> {
  // The sandbox runs in a Firecracker microVM:
  // - Isolated kernel
  // - No network access (unless explicitly granted)
  // - Limited CPU and memory
  // - Ephemeral — destroyed after execution
  // - No access to host filesystem

  const response = await fetch(process.env.SANDBOX_API_URL!, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${process.env.SANDBOX_API_KEY}`,
    },
    body: JSON.stringify({
      code,
      language,
      timeout: timeoutMs,
      // Sandbox configuration
      limits: {
        memory: "256MB",
        cpu: "0.5",       // Half a CPU core
        disk: "100MB",
        network: false,    // No network access for code execution
      },
    }),
  });

  if (!response.ok) {
    throw new Error(`Sandbox error: ${response.statusText}`);
  }

  return response.json();
}
```

---

## 6. Network-Level Controls Deep Dive

### 6.1 DNS Restrictions

Even with the egress proxy, the agent could potentially use DNS to exfiltrate data (DNS tunneling). Restrict DNS to the VPC DNS resolver:

```hcl
# terraform/dns.tf — DNS restrictions

# Use Route 53 Resolver to control DNS
resource "aws_route53_resolver_firewall_domain_list" "allowed" {
  name = "agent-sandbox-allowed-domains"
  domains = [
    "linear.app.",
    "*.linear.app.",
    "api.linear.app.",
    "slack.com.",
    "*.slack.com.",
    "api.anthropic.com.",
    # AWS service endpoints
    "*.amazonaws.com.",
  ]
}

resource "aws_route53_resolver_firewall_domain_list" "blocked" {
  name    = "agent-sandbox-blocked-all"
  domains = ["*."]  # Block everything not explicitly allowed
}

resource "aws_route53_resolver_firewall_rule_group" "agent" {
  name = "agent-sandbox-dns-rules"
}

resource "aws_route53_resolver_firewall_rule" "allow_listed" {
  name                    = "allow-listed-domains"
  firewall_rule_group_id  = aws_route53_resolver_firewall_rule_group.agent.id
  firewall_domain_list_id = aws_route53_resolver_firewall_domain_list.allowed.id
  priority                = 100
  action                  = "ALLOW"
}

resource "aws_route53_resolver_firewall_rule" "block_all" {
  name                    = "block-everything-else"
  firewall_rule_group_id  = aws_route53_resolver_firewall_rule_group.agent.id
  firewall_domain_list_id = aws_route53_resolver_firewall_domain_list.blocked.id
  priority                = 200
  action                  = "BLOCK"

  block_response = "NXDOMAIN"
}

resource "aws_route53_resolver_firewall_rule_group_association" "agent" {
  name                   = "agent-sandbox-dns-association"
  firewall_rule_group_id = aws_route53_resolver_firewall_rule_group.agent.id
  vpc_id                 = aws_vpc.agent_sandbox.id
  priority               = 101
}
```

### 6.2 VPC Flow Logs for Auditing

```hcl
# terraform/monitoring.tf — log all network traffic for audit

resource "aws_flow_log" "agent_vpc" {
  vpc_id               = aws_vpc.agent_sandbox.id
  traffic_type         = "ALL"
  log_destination      = aws_cloudwatch_log_group.vpc_flow_logs.arn
  log_destination_type = "cloud-watch-logs"
  iam_role_arn         = aws_iam_role.flow_logs.arn

  tags = {
    Name = "agent-sandbox-flow-logs"
  }
}

resource "aws_cloudwatch_log_group" "vpc_flow_logs" {
  name              = "/agent-sandbox/vpc-flow-logs"
  retention_in_days = 90
}

# Alert on unexpected traffic
resource "aws_cloudwatch_metric_alarm" "unexpected_egress" {
  alarm_name          = "agent-sandbox-unexpected-egress"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "RejectedPackets"
  namespace           = "AgentSandbox"
  period              = 300  # 5 minutes
  statistic           = "Sum"
  threshold           = 10
  alarm_description   = "Agent attempted to reach unauthorized network destinations"
  alarm_actions       = [aws_sns_topic.security_alerts.arn]
}

resource "aws_sns_topic" "security_alerts" {
  name = "agent-sandbox-security-alerts"
}
```

### 6.3 Egress Monitoring and Alerting

```python
# scripts/monitor_egress.py — analyze proxy logs for suspicious patterns
import re
import sys
from collections import Counter
from datetime import datetime


def analyze_squid_log(log_file: str) -> dict:
    """Analyze Squid proxy access logs for security concerns."""
    
    blocked_attempts = []
    allowed_requests = []
    domain_counter = Counter()
    
    with open(log_file) as f:
        for line in f:
            parts = line.strip().split()
            if len(parts) < 10:
                continue
            
            timestamp = float(parts[0])
            status = parts[3]
            url = parts[6]
            
            # Extract domain from URL
            domain_match = re.search(r'https?://([^/:]+)', url)
            if not domain_match:
                # CONNECT method for HTTPS
                domain_match = re.search(r'^([^:]+)', url)
            
            if domain_match:
                domain = domain_match.group(1)
                domain_counter[domain] += 1
            
            if status.startswith("TCP_DENIED") or status.startswith("403"):
                blocked_attempts.append({
                    "timestamp": datetime.fromtimestamp(timestamp).isoformat(),
                    "url": url,
                    "status": status,
                })
            else:
                allowed_requests.append({
                    "timestamp": datetime.fromtimestamp(timestamp).isoformat(),
                    "url": url,
                    "status": status,
                })
    
    return {
        "total_requests": len(allowed_requests) + len(blocked_attempts),
        "allowed": len(allowed_requests),
        "blocked": len(blocked_attempts),
        "blocked_attempts": blocked_attempts[:20],  # First 20
        "top_domains": domain_counter.most_common(10),
        "unique_domains": len(domain_counter),
    }


if __name__ == "__main__":
    report = analyze_squid_log(sys.argv[1])
    
    print(f"Total requests: {report['total_requests']}")
    print(f"Allowed: {report['allowed']}")
    print(f"Blocked: {report['blocked']}")
    
    if report["blocked_attempts"]:
        print("\n--- BLOCKED ATTEMPTS (potential exfiltration) ---")
        for attempt in report["blocked_attempts"]:
            print(f"  {attempt['timestamp']}: {attempt['url']}")
    
    print("\n--- Top Domains ---")
    for domain, count in report["top_domains"]:
        print(f"  {domain}: {count} requests")
```

---

## 7. File System Isolation

### 7.1 Read-Only Root Filesystem

The agent's container runs with a read-only root filesystem. It can only write to explicitly mounted tmpfs volumes.

```dockerfile
# Dockerfile — agent with read-only filesystem support
FROM node:20-slim AS production

WORKDIR /app

# Install application (this becomes read-only at runtime)
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./

# Pre-create directories that Node.js needs to write to
RUN mkdir -p /tmp/workspace /tmp/.npm && \
    addgroup --system agent && \
    adduser --system --group agent && \
    chown -R agent:agent /tmp/workspace /tmp/.npm

USER agent

# At runtime, these are the ONLY writable locations:
# /tmp         — tmpfs, 100MB, noexec (cannot execute binaries)
# /tmp/workspace — tmpfs, 500MB, noexec (agent working directory)
# Everything else is read-only

CMD ["node", "dist/agent.js"]
```

### 7.2 Why File System Isolation Matters

Without file system isolation, a prompt-injected agent could:

```
// Without read-only FS, a compromised agent could:
1. Write a malicious script to disk and execute it
2. Modify application code to change its behavior
3. Write sensitive data to disk and later exfiltrate it
4. Create a reverse shell script
5. Modify /etc/resolv.conf to redirect DNS to an attacker's server

// With read-only FS + noexec tmpfs:
1. Cannot write executable files anywhere
2. Cannot modify application code
3. Can only write to tmpfs (in memory, limited size, wiped on restart)
4. Cannot create or execute scripts
5. Cannot modify system configuration
```

### 7.3 Ephemeral Workspaces

```typescript
// src/workspace.ts — ephemeral workspace manager for agent tasks
import { mkdtemp, rm, writeFile, readFile, readdir, stat } from "fs/promises";
import { join } from "path";
import { tmpdir } from "os";

const WORKSPACE_BASE = "/tmp/workspace";
const MAX_WORKSPACE_SIZE = 100 * 1024 * 1024; // 100MB

export class EphemeralWorkspace {
  private dir: string | null = null;
  private bytesWritten = 0;

  async create(): Promise<string> {
    this.dir = await mkdtemp(join(WORKSPACE_BASE, "task-"));
    this.bytesWritten = 0;
    return this.dir;
  }

  async writeFile(filename: string, content: string): Promise<void> {
    if (!this.dir) throw new Error("Workspace not created");

    // Prevent path traversal
    const safeName = filename.replace(/[^a-zA-Z0-9._-]/g, "_");
    const filePath = join(this.dir, safeName);

    // Verify the resolved path is within the workspace
    if (!filePath.startsWith(this.dir)) {
      throw new Error("Path traversal detected");
    }

    // Check size limit
    const contentSize = Buffer.byteLength(content, "utf-8");
    if (this.bytesWritten + contentSize > MAX_WORKSPACE_SIZE) {
      throw new Error("Workspace size limit exceeded");
    }

    await writeFile(filePath, content, "utf-8");
    this.bytesWritten += contentSize;
  }

  async readFile(filename: string): Promise<string> {
    if (!this.dir) throw new Error("Workspace not created");
    const safeName = filename.replace(/[^a-zA-Z0-9._-]/g, "_");
    const filePath = join(this.dir, safeName);
    if (!filePath.startsWith(this.dir)) {
      throw new Error("Path traversal detected");
    }
    return readFile(filePath, "utf-8");
  }

  async listFiles(): Promise<string[]> {
    if (!this.dir) throw new Error("Workspace not created");
    return readdir(this.dir);
  }

  async destroy(): Promise<void> {
    if (this.dir) {
      await rm(this.dir, { recursive: true, force: true });
      this.dir = null;
      this.bytesWritten = 0;
    }
  }
}
```

---

## 8. Sandboxing Decision Framework

### 8.1 How Much Isolation Do You Need?

```
Agent Capability          Risk Level      Minimum Isolation
──────────────────────    ──────────      ──────────────────
Read-only data access     Low             App-level permissions
  (search, Q&A)

Write access to safe      Medium          App-level permissions +
  systems (create tickets,                 rate limiting +
  send notifications)                      audit logging

Write access to           High            Docker sandbox +
  sensitive systems                        network restrictions +
  (database writes,                        separate service account
  file modifications)

Arbitrary code execution  Critical        Separate AWS account +
  (coding agents,                          VPC isolation +
  shell access)                            egress proxy +
                                           read-only FS +
                                           ephemeral execution

Internet access +         Maximum         Full OpenClaw pattern:
  tool use +                               separate account,
  untrusted input                          proxy allowlist,
                                           not internet-addressable,
                                           ephemeral containers
```

### 8.2 Incremental Sandboxing

You don't have to build the full OpenClaw pattern on day one. Here's a pragmatic adoption path:

**Level 0: No isolation** (prototype only)
- Agent runs in your application process
- Same network, same permissions, same account
- Never use this in production

**Level 1: Application-level isolation** (small risk)
- Tool allowlist, input validation, permission checks
- Same network and account, but limited API calls
- Good for: internal-only tools, limited tool set

**Level 2: Container isolation** (medium risk)
- Agent runs in Docker with read-only FS, dropped capabilities
- Network restrictions via Docker networking
- Good for: agents with file system needs

**Level 3: Network isolation** (high risk)
- Separate VPC or subnet, egress proxy
- Security groups restrict network access
- Good for: agents with internet access needs

**Level 4: Account isolation** (maximum risk — the OpenClaw pattern)
- Separate AWS account, no production connectivity
- VPC endpoints only, proxy-restricted internet
- Not internet-addressable, ephemeral execution
- Good for: agents with tool use + internet + untrusted input

### 8.3 Cost of Each Level

| Level | Monthly Cost Adder | Ops Complexity | Setup Time |
|-------|--------------------|----------------|------------|
| 0 | $0 | None | 0 |
| 1 | $0 | Low | 1-2 days |
| 2 | ~$10-30 | Low | 2-3 days |
| 3 | ~$100-300 (NAT GW, proxy) | Medium | 1-2 weeks |
| 4 | ~$200-500 (NAT GW, proxy, VPC endpoints) | High | 2-4 weeks |

The cost is dominated by the NAT Gateway (~$32/month) and VPC Interface Endpoints (~$7/month each). For production agent workloads with real risk, this is trivial compared to the cost of a security incident.

---

## 9. Common Mistakes

### 9.1 Security Groups Are Not Enough

```
MISTAKE: "I added security groups, so the agent is isolated"

Security groups are stateful — they allow return traffic for any
outbound connection. If the agent can make an outbound connection
to ANY IP on port 443, it can exfiltrate data to any HTTPS server.

FIX: Use an egress proxy with domain allowlisting. Security groups
control which IP addresses can be reached. The proxy controls which
domain names can be reached. You need both.
```

### 9.2 VPC Peering Defeats Account Isolation

```
MISTAKE: "The agent is in a separate account, but we peered the
VPCs so it can access the shared database"

VPC peering creates a network path between accounts. This completely
defeats the purpose of account isolation. If the agent's VPC can
route to the production VPC, a compromised agent can reach
production resources.

FIX: If the agent needs data from production, copy it to the sandbox
account. Never create a direct network path.
```

### 9.3 Environment Variables Are Not Secrets Management

```
MISTAKE: "We put the API key in an environment variable, so it's secure"

Environment variables are visible in:
- ECS task definitions (stored in plaintext)
- Docker inspect output
- Process listings (/proc/*/environ)
- Container logs if accidentally printed

FIX: Use AWS Secrets Manager or Vault. The secret is fetched at
runtime and never stored in the task definition.
```

### 9.4 Allowing All AWS Endpoints

```
MISTAKE: "We set NO_PROXY=*.amazonaws.com so AWS SDK calls bypass the proxy"

This allows the agent to call ANY AWS service, including those in
other accounts (if credentials allow). It could access public S3
buckets, call STS to get temporary credentials, or interact with
services you didn't intend.

FIX: Use VPC endpoints for the specific AWS services needed. Block
DNS resolution for all others. The NO_PROXY setting should only
include the VPC endpoint addresses, not all of amazonaws.com.
```

---

## 10. Monitoring a Sandboxed Agent

### 10.1 What to Monitor

```typescript
// src/monitoring.ts — agent monitoring and alerting
interface AgentMetrics {
  taskId: string;
  // Security metrics
  blockedEgressAttempts: number;     // Proxy denied requests
  dnsBlockedQueries: number;         // DNS firewall blocks
  toolCallsOutsideAllowlist: number; // App-level tool blocks
  // Operational metrics
  executionTimeMs: number;
  tokensUsed: { input: number; output: number };
  toolCallsMade: number;
  // Resource metrics
  memoryPeakMb: number;
  tempDiskUsageMb: number;
}

function alertOnAnomalies(metrics: AgentMetrics) {
  // Alert: any blocked egress attempt is suspicious
  if (metrics.blockedEgressAttempts > 0) {
    sendAlert("SECURITY", `Agent task ${metrics.taskId} attempted ${metrics.blockedEgressAttempts} blocked egress requests`);
  }

  // Alert: high tool call count might indicate a loop
  if (metrics.toolCallsMade > 20) {
    sendAlert("OPERATIONAL", `Agent task ${metrics.taskId} made ${metrics.toolCallsMade} tool calls (possible runaway loop)`);
  }

  // Alert: excessive token usage
  if (metrics.tokensUsed.input + metrics.tokensUsed.output > 100_000) {
    sendAlert("COST", `Agent task ${metrics.taskId} used ${metrics.tokensUsed.input + metrics.tokensUsed.output} tokens`);
  }

  // Alert: approaching resource limits
  if (metrics.tempDiskUsageMb > 400) { // 500MB limit
    sendAlert("RESOURCE", `Agent task ${metrics.taskId} using ${metrics.tempDiskUsageMb}MB of temp disk (limit: 500MB)`);
  }
}

function sendAlert(category: string, message: string) {
  console.error(`[ALERT:${category}] ${message}`);
  // In production: send to Slack, PagerDuty, CloudWatch, etc.
}
```

---

## 11. Executing AI-Generated Code Safely

Everything in Sections 1-10 is about isolating agents that use predefined tools. But there is a harder problem: what happens when the agent *writes* code and you need to *run* it? This is not hypothetical. This is how Ramp's Glass enables "anyone can build anything in 5 minutes" -- the user describes what they want, the AI writes the code, and a sandbox executes it. It is the pattern behind data analysis agents, code interpreters, and no-code builders.

The challenge: you are executing arbitrary code that did not exist until the LLM generated it. You cannot review it in advance. You cannot predict what it will do. You must assume it is hostile.

### 11.1 The Execution Pattern

```
User describes task
        |
        v
LLM generates code (TypeScript, Python, SQL, etc.)
        |
        v
Validation layer (syntax check, blocked patterns, size limits)
        |
        v
Sandbox executes code (isolated from host, network, filesystem)
        |
        v
Capture output (stdout, stderr, return values, generated files)
        |
        v
Return results to user (formatted by LLM if needed)
```

### 11.2 Sandbox Options

There are three serious options for executing AI-generated code. Each has different trade-offs.

**Vercel Sandbox: Ephemeral Firecracker MicroVMs**

Vercel Sandbox runs each code execution in a Firecracker microVM -- the same technology that powers AWS Lambda. Each VM boots in milliseconds, runs the code, and is destroyed. There is no state between executions. The VM has no network access by default and a constrained filesystem.

```typescript
// vercel-sandbox-execution.ts

async function executeInVercelSandbox(code: string): Promise<{
  stdout: string;
  stderr: string;
  exitCode: number;
}> {
  const sandbox = await Sandbox.create({
    // Resource limits
    timeoutMs: 10_000,        // Kill after 10 seconds
    memoryMB: 256,            // 256MB max memory
    // Network policy
    network: "none",           // No network access
    // Filesystem
    readOnlyRootFs: true,      // Cannot modify the base filesystem
    workDir: "/tmp/workspace", // Writable scratch space
  });

  try {
    // Write the generated code to a file in the sandbox
    await sandbox.writeFile("/tmp/workspace/script.ts", code);

    // Execute it
    const result = await sandbox.exec("npx", ["tsx", "/tmp/workspace/script.ts"]);

    return {
      stdout: result.stdout,
      stderr: result.stderr,
      exitCode: result.exitCode,
    };
  } finally {
    // Sandbox is destroyed -- no state persists
    await sandbox.destroy();
  }
}
```

**E2B: Cloud Sandboxes for AI Agents**

E2B provides cloud sandboxes specifically designed for AI-generated code execution. They support multiple languages, persistent filesystem within a session, and fine-grained resource control.

```typescript
// e2b-execution.ts
import { Sandbox } from "e2b";

async function executeWithE2B(code: string, language: "typescript" | "python") {
  const sandbox = await Sandbox.create({
    template: language === "typescript" ? "node" : "python",
    timeoutMs: 30_000,
  });

  try {
    if (language === "typescript") {
      await sandbox.filesystem.write("/home/user/script.ts", code);
      const result = await sandbox.process.start({
        cmd: "npx tsx /home/user/script.ts",
        timeout: 10_000,
      });
      await result.wait();
      return { stdout: result.output.stdout, stderr: result.output.stderr };
    } else {
      await sandbox.filesystem.write("/home/user/script.py", code);
      const result = await sandbox.process.start({
        cmd: "python /home/user/script.py",
        timeout: 10_000,
      });
      await result.wait();
      return { stdout: result.output.stdout, stderr: result.output.stderr };
    }
  } finally {
    await sandbox.close();
  }
}
```

**Docker-Based Sandboxing (Self-Hosted)**

If you need to run this on your own infrastructure, Docker with strict security profiles is the self-hosted option. Note the use of `execFile` rather than `exec` -- this avoids shell injection by passing arguments as an array instead of a string.

```typescript
// docker-sandbox.ts
import { execFile } from "node:child_process";
import { writeFile, mkdtemp, rm } from "node:fs/promises";
import { join } from "node:path";
import { tmpdir } from "node:os";
import { promisify } from "node:util";

const execFileAsync = promisify(execFile);

async function executeInDocker(code: string): Promise<{
  stdout: string;
  stderr: string;
  exitCode: number;
}> {
  // Create a temp directory for the code
  const tempDir = await mkdtemp(join(tmpdir(), "sandbox-"));

  try {
    // Write the code to a file
    await writeFile(join(tempDir, "script.ts"), code);

    // Run in Docker with strict isolation -- execFile prevents shell injection
    const { stdout, stderr } = await execFileAsync("docker", [
      "run",
      "--rm",                              // Remove container after exit
      "--network=none",                    // No network access
      "--read-only",                       // Read-only root filesystem
      "--tmpfs=/tmp:noexec,size=64m",      // Writable /tmp, no execution, 64MB max
      "--memory=256m",                     // 256MB memory limit
      "--cpus=0.5",                        // Half a CPU core
      "--pids-limit=50",                   // Max 50 processes
      "--security-opt=no-new-privileges",  // Cannot escalate privileges
      "--cap-drop=ALL",                    // Drop all Linux capabilities
      `-v=${tempDir}:/workspace:ro`,       // Mount code as read-only
      "node:20-slim",                      // Minimal Node.js image
      "npx", "tsx", "/workspace/script.ts",
    ], {
      timeout: 15_000, // Kill after 15 seconds
    });

    return { stdout, stderr, exitCode: 0 };
  } catch (error: unknown) {
    const err = error as { stdout?: string; stderr?: string; code?: number };
    return {
      stdout: err.stdout ?? "",
      stderr: err.stderr ?? "Execution failed",
      exitCode: err.code ?? 1,
    };
  } finally {
    // Clean up the temp directory
    await rm(tempDir, { recursive: true, force: true });
  }
}
```

### 11.3 Building a Code Execution Tool for an Agent

The real power comes when you wire code execution into an agent's tool set. The agent can write code, run it, inspect the output, fix bugs, and iterate -- all within the sandbox.

```typescript
// code-execution-tool.ts
import Anthropic from "@anthropic-ai/sdk";

const anthropic = new Anthropic();

// Define the code execution tool
const codeExecutionTool: Anthropic.Tool = {
  name: "execute_code",
  description:
    "Execute TypeScript code in an isolated sandbox. " +
    "Use this to perform calculations, data transformations, " +
    "or generate outputs. The sandbox has no network access " +
    "and limited resources. Code should write results to stdout.",
  input_schema: {
    type: "object" as const,
    properties: {
      code: {
        type: "string",
        description: "TypeScript code to execute. Must be self-contained.",
      },
      description: {
        type: "string",
        description: "Brief description of what this code does (for audit logging).",
      },
    },
    required: ["code", "description"],
  },
};

// Pre-execution validation
function validateCode(code: string): { valid: boolean; reason?: string } {
  // Size limit: reject extremely large code
  if (code.length > 50_000) {
    return { valid: false, reason: "Code exceeds 50KB size limit" };
  }

  // Block dangerous patterns
  const blocked = [
    { pattern: /require\s*\(\s*['"]child_process['"]\s*\)/, name: "child_process import" },
    { pattern: /require\s*\(\s*['"]cluster['"]\s*\)/, name: "cluster import" },
    { pattern: /process\.exit/, name: "process.exit" },
    { pattern: /process\.kill/, name: "process.kill" },
    { pattern: /eval\s*\(/, name: "eval()" },
    { pattern: /Function\s*\(/, name: "Function constructor" },
  ];

  for (const { pattern, name } of blocked) {
    if (pattern.test(code)) {
      return { valid: false, reason: `Code contains blocked pattern: ${name}` };
    }
  }

  return { valid: true };
}

// The agent loop with code execution
async function agentWithCodeExecution(userTask: string) {
  const messages: Anthropic.MessageParam[] = [
    { role: "user", content: userTask },
  ];

  const systemPrompt = `You are a helpful assistant that can execute TypeScript code to complete tasks.
When the user asks you to calculate, transform data, or produce output, write TypeScript code and use the execute_code tool.
The sandbox has no network access. Write results to console.log().
If code fails, read the error, fix the code, and try again (up to 3 attempts).`;

  while (true) {
    const response = await anthropic.messages.create({
      model: "claude-sonnet-4-20250514",
      max_tokens: 4096,
      system: systemPrompt,
      tools: [codeExecutionTool],
      messages,
    });

    // If the model is done, return its final message
    if (response.stop_reason === "end_turn") {
      const textBlock = response.content.find(b => b.type === "text");
      return textBlock ? (textBlock as { text: string }).text : "Done.";
    }

    // Process tool calls
    const toolUseBlocks = response.content.filter(
      (b): b is Anthropic.ToolUseBlock => b.type === "tool_use"
    );

    // Add the assistant's response
    messages.push({ role: "assistant", content: response.content });

    for (const toolUse of toolUseBlocks) {
      const { code, description } = toolUse.input as { code: string; description: string };

      // Audit log
      console.log(`[AUDIT] Code execution requested: ${description}`);

      // Validate
      const validation = validateCode(code);
      if (!validation.valid) {
        messages.push({
          role: "user",
          content: [{
            type: "tool_result",
            tool_use_id: toolUse.id,
            content: `Execution blocked: ${validation.reason}. Rewrite without the blocked pattern.`,
            is_error: true,
          }],
        });
        continue;
      }

      // Execute in sandbox (use whichever sandbox from 11.2)
      const result = await executeInDocker(code);

      messages.push({
        role: "user",
        content: [{
          type: "tool_result",
          tool_use_id: toolUse.id,
          content: result.exitCode === 0
            ? result.stdout
            : `Exit code ${result.exitCode}\nStdout: ${result.stdout}\nStderr: ${result.stderr}`,
          is_error: result.exitCode !== 0,
        }],
      });
    }
  }
}

// Usage
const answer = await agentWithCodeExecution(
  "Calculate the compound interest on $10,000 at 7% annual rate over 30 years, " +
  "compounding monthly. Show me a table of the balance at each 5-year mark."
);
console.log(answer);
```

### 11.4 Resource Limits Reference

| Resource | Recommended Limit | Why |
|----------|------------------|-----|
| CPU time | 10-30 seconds | Prevents infinite loops and crypto mining |
| Memory | 256-512MB | Prevents memory bombs |
| Disk | 64-128MB (tmpfs) | Prevents disk filling |
| Network | None (default) | Prevents data exfiltration and attacks |
| Processes | 50 max | Prevents fork bombs |
| File descriptors | 1024 | Prevents descriptor exhaustion |
| Output size | 1MB stdout + 1MB stderr | Prevents memory exhaustion in the host |

For data analysis tasks where the code needs to read input data, pass it as a file mounted read-only into the sandbox. Never give the sandbox network access to fetch data -- fetch the data in your trusted host process and mount it in.

### 11.5 When to Use Each Sandbox

| | Vercel Sandbox | E2B | Docker (Self-Hosted) |
|---|---------------|-----|---------------------|
| **Setup** | Zero (managed) | Zero (managed) | Medium (maintain images) |
| **Boot time** | ~100ms | ~500ms | ~1-2s |
| **Isolation** | Firecracker microVM | Cloud VM | Container (weaker) |
| **Cost** | Pay per execution | Pay per second | Your compute |
| **Best for** | Production code execution | Multi-language, persistent sessions | Self-hosted, air-gapped |
| **Network control** | Yes | Yes | Yes |

---

## 12. What's Next

You now know how to isolate agents at the infrastructure level. The OpenClaw pattern -- separate account, VPC endpoints, proxy-restricted internet, not internet-addressable, ephemeral execution -- provides defense-in-depth that survives prompt injection. You have the Terraform modules to build it and the monitoring to detect breaches. And you know how to safely execute AI-generated code in sandboxes -- from managed solutions like Vercel Sandbox and E2B to self-hosted Docker isolation.

But all this infrastructure is manual. Every deployment is a human process: build the container, push to ECR, update the task definition, verify the deployment. If someone pushes a prompt change that degrades agent quality, you won't know until users complain. If someone introduces a security regression, you won't catch it until the next audit.

In Chapter 56, we will build the **CI/CD pipeline for AI applications**. This includes running eval suites on every PR, canary deployments for prompt changes, A/B testing LLM versions, and feature flags for AI capabilities. The pipeline ensures that every change to your AI system -- code, prompts, models, or tools -- is tested, measured, and safely rolled out.

---

## Key Takeaways

1. **Application-level security does not survive prompt injection.** Only infrastructure-level isolation protects against a fully compromised agent.
2. **The OpenClaw pattern is the gold standard** for agent sandboxing: separate account, VPC endpoints, proxy-restricted internet, not internet-addressable, ephemeral execution.
3. **Defense in depth means layers that don't trust each other.** Security groups, NACLs, egress proxy, DNS restrictions, permission boundaries -- each layer assumes the others have failed.
4. **Egress proxy with domain allowlisting** is the single most important control. Without it, an agent with internet access can exfiltrate data to any server.
5. **File system isolation (read-only root + noexec tmpfs)** prevents an agent from persisting malicious code or writing executable files.
6. **The cost of isolation is trivial** compared to the cost of a security incident. ~$200-500/month for the full OpenClaw pattern.
7. **Adopt incrementally.** Start with application-level controls (Level 1), add container isolation (Level 2), then network and account isolation (Levels 3-4) as your agent's capabilities and risk profile grow.
8. **AI-generated code execution requires the strictest sandboxing.** Use Firecracker microVMs (Vercel Sandbox), cloud sandboxes (E2B), or locked-down Docker containers. Always validate before executing, always kill after a timeout, and never give network access by default.
