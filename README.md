# TuesdayPf2e — EC2 Server Control

Manually triggered GitHub Actions workflows to start and stop an AWS EC2 game server, and keep a Route 53 A record pointed at its public IP.

## Workflows

| Workflow | File | What it does |
|----------|------|--------------|
| **Shut Down Server** | [`.github/workflows/shutdown-server.yml`](.github/workflows/shutdown-server.yml) | Stops the EC2 instance. If it is already stopped (or stopping), the job exits successfully with that message. |
| **Startup Server** | [`.github/workflows/startup-server.yml`](.github/workflows/startup-server.yml) | Starts the EC2 instance (if needed), reads its public IP, and UPSERTs the Route 53 A record. If the instance is already running, start is skipped and DNS is still synced. |

### How to run

1. Open the repository on GitHub.
2. Go to **Actions**.
3. Select **Shut Down Server** or **Startup Server**.
4. Click **Run workflow**.

## Required GitHub configuration

Configure these under **Settings → Secrets and variables → Actions**.

### Secrets

| Name | Description |
|------|-------------|
| `AWS_ACCESS_KEY_ID` | Access key for an IAM user with the permissions below |
| `AWS_SECRET_ACCESS_KEY` | Secret key for that IAM user |

### Variables

| Name | Example | Description |
|------|---------|-------------|
| `AWS_REGION` | `us-east-1` | AWS region of the EC2 instance |
| `EC2_INSTANCE_ID` | `i-0123456789abcdef0` | Target EC2 instance ID |
| `ROUTE53_HOSTED_ZONE_ID` | `Z1234567890ABC` | Route 53 hosted zone ID |
| `ROUTE53_RECORD_NAME` | `game.example.com` | FQDN for the A record (trailing dot optional) |
| `ROUTE53_TTL` | `300` | TTL in seconds (optional; defaults to `300` if unset) |

## IAM permissions

Attach a policy like this to the IAM user used by the workflows. Replace `REGION`, `ACCOUNT_ID`, `INSTANCE_ID`, and `HOSTED_ZONE_ID`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EC2Describe",
      "Effect": "Allow",
      "Action": ["ec2:DescribeInstances"],
      "Resource": "*"
    },
    {
      "Sid": "EC2StartStop",
      "Effect": "Allow",
      "Action": ["ec2:StartInstances", "ec2:StopInstances"],
      "Resource": "arn:aws:ec2:REGION:ACCOUNT_ID:instance/INSTANCE_ID"
    },
    {
      "Sid": "Route53Update",
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets",
        "route53:GetChange"
      ],
      "Resource": [
        "arn:aws:route53:::hostedzone/HOSTED_ZONE_ID",
        "arn:aws:route53:::change/*"
      ]
    }
  ]
}
```

## Behavior notes

- **Already stopped:** Shut Down Server reports that the instance is already stopped/stopping and succeeds without calling stop again.
- **Already running:** Startup Server skips `StartInstances`, still waits for / reads the public IP, and UPSERTs Route 53 so DNS stays correct.
- **Public IP required:** The instance must be in a public subnet (or otherwise receive a public IPv4 address). Without a public IP, Startup Server fails after polling.
- **DNS:** Startup always UPSERTs the A record to the current public IP, which covers IP changes after stop/start when Elastic IP is not used.
