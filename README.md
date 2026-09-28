# 30 Days, 30 AWS Projects

### One day. One AWS project. Build practical cloud experience.

A 30-day hands-on AWS challenge, documenting architecture, implementation, screenshots, and lessons learned.

## Project journal

| Day | Date | Project | Topics | Evidence |
| --- | --- | --- | --- | --- |
| 01 | 2026-09-26 | [Day 01 – AWS Peering Setup](day-01-aws-peering-setup/README.md) | VPC, routing, same-Region and inter-Region peering, EC2 | Architecture + 10 implementation screenshots |
| 02 | 2026-09-27 | [Day 02 – VPC Flow Logs to S3](day-02-vpc-flow-logs-to-s3/README.md) | EC2, Nginx, VPC Flow Logs, S3 | 8 selected implementation screenshots |
| 03 | 2026-09-27 | [Day 03 – S3 Gateway VPC Endpoint](day-03-s3-gateway-vpc-endpoint/README.md) | Private subnets, S3 Gateway Endpoint | Architecture + 6-step test evidence |

## Repository structure

```text
30-days-30-aws-projects/
├── README.md
├── PROJECT-TEMPLATE.md
├── day-01-aws-peering-setup/
│   ├── README.md
│   └── images/
├── day-02-vpc-flow-logs-to-s3/
│   ├── README.md
│   └── images/
└── day-03-s3-gateway-vpc-endpoint/
    ├── README.md
    └── images/
```

Each new project gets its own descriptive folder. The journal records the day number and date so folders remain easy to browse by topic.

## Add the next daily project

1. Create a folder named for the project, using lowercase words separated by hyphens.
2. Copy [PROJECT-TEMPLATE.md](PROJECT-TEMPLATE.md) into that folder as `README.md`.
3. Put architecture diagrams and screenshots in the project's `images/` folder.
4. Document the steps, actual results, lessons learned, and cleanup.
5. Add a row to the journal above with the next day number, date, and project link.

Use relative image links such as `![Route table](images/route-table.png)` so the documentation renders directly on GitHub. Report only results supported by your test output or screenshots.

## Learning areas

Networking · Compute · Storage · Identity and access · Databases · Monitoring · Automation

New projects are added as the practice journey continues.
