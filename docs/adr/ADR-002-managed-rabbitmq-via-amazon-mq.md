# ADR-002: Use Amazon MQ (managed RabbitMQ) for the Milestone 2 message broker

## Status
Accepted

## Context
Milestone 2 turns the Trade API from a synchronous request/response service
into the entry point of an event-driven pipeline: it publishes each accepted
trade to a queue instead of computing anything itself, and a new Risk Engine
consumer reads from that queue to compute risk. The roadmap commits to
RabbitMQ specifically, over AMQP, rather than an AWS-proprietary queue — so
the open question for M2 is not which broker, but who runs it.

As with ADR-001, this is a solo build in a personal AWS account under a fixed
budget, and the project has an established pattern of trading a managed-service
cost premium for less operational surface when the milestone's real value is
elsewhere (Fargate over self-managed EC2 hosts). For M2, the actual point of
the milestone is the event-driven boundary and the risk-computation logic
behind it — not learning to operate a message broker. Self-hosting RabbitMQ on
ECS Fargate was seriously considered: it costs roughly the same at this scale,
and it would force early use of AWS Cloud Map for service discovery (the
Trade API and Risk Engine need a stable internal address for a broker that has
no business being internet-facing), a pattern this project doesn't have yet
and will need again once M3 adds Postgres/Redis. But it also means owning
RabbitMQ's version, upgrades, and message durability (Fargate's local storage
is ephemeral, so durable queues need an EFS mount — another new resource, more
Terraform, more surface) before the Risk Engine itself exists. Given the
choice between spending M2's time on broker infrastructure or on the actual
event-driven design, speed and staying inside the project's existing "let AWS
own the undifferentiated part" pattern won out.

## Decision
Milestone 2's broker is Amazon MQ for RabbitMQ, deployed as a single-instance
broker (no HA) on the smallest supported instance class (`mq.t3.micro`),
placed in the private subnets already provisioned by `infra/vpc`. Credentials
are generated with Terraform's `random_password` and stored in AWS Secrets
Manager rather than committed to any `.tf` file or CI variable. This follows
the existing module pattern: a new `infra/messaging` root module with its own
S3 backend key, pulling `vpc_id` and private subnet IDs from `infra/vpc`'s
remote state the same way `infra/ecs` does. The module is added to the
`infra.yml` matrix, and `rtrp-ci-permissions` gets a new statement granting
the `mq:*` actions the CI role doesn't currently have.

## Alternatives considered
- **Self-hosted RabbitMQ on ECS Fargate** — not rejected outright, deferred.
  Cost is comparable to Amazon MQ at this scale, and the Cloud Map
  service-discovery pattern it would force is genuinely useful groundwork for
  M5's EKS migration (a Cloud Map-backed internal DNS name is conceptually the
  same thing a Kubernetes `Service` provides). But it bundles three new
  concerns — service discovery, durable storage via EFS, and RabbitMQ
  version/upgrade ownership — into a milestone whose real subject is the
  message contract and the Risk Engine, not the broker. The same
  service-discovery lesson will very likely come up again in M3 regardless of
  this decision, so nothing here is lost for good — only resequenced.
- **Amazon SQS** — rejected. It's the cheapest, lowest-ops option and would
  need no new Terraform pattern beyond a single `aws_sqs_queue`, but it's
  AWS-proprietary: no exchanges, no routing keys, no AMQP semantics. The
  roadmap already commits to RabbitMQ for this milestone, and swapping it for
  SQS would mean designing the M2 event boundary around a different messaging
  model than planned, not just a different host.
- **Amazon MQ, active/standby (HA)** — rejected for M2. Roughly doubles the
  broker cost for a failover guarantee this milestone doesn't need — a single
  Trade API instance and a single Risk Engine consumer have no HA requirement
  to protect yet. Same logic ADR-001 used to defer EKS: pay for resilience
  when something depends on it, not before.

## Consequences
- Ships M2's event boundary faster: no service-discovery setup, no EFS, no
  RabbitMQ upgrade path to own — effort goes into the message contract (what
  the Trade API publishes) and the Risk Engine's consumer logic, which is the
  actual point of this milestone.
- Cost: adds roughly $26-32/month (eu-north-1, single-instance `mq.t3.micro`,
  approximate — verify against the AWS Pricing Calculator before applying) on
  top of the ~$60-70/month M1 already runs. Not meaningfully cheaper than
  self-hosting would have been; this decision was about time and operational
  surface, not cost.
- New skill/surface introduced deliberately: this is the project's first use
  of Secrets Manager, needed because Amazon MQ requires broker credentials at
  creation time and hardcoding them was never on the table.
- New CI gap to close on day one of implementation: `rtrp-ci-permissions` has
  no `mq:*` actions yet, so the first `terraform plan` against
  `infra/messaging` will fail authorization the same way `ecr` and `dns` did
  when the CI matrix first covered them — expected, not a surprise, given the
  pattern already seen twice this project.
- Deferred, not eliminated: the Cloud Map / internal-service-discovery lesson
  this choice avoids for M2 is still on the roadmap — M3's Postgres/Redis (if
  self-hosted rather than RDS/ElastiCache) or a future self-hosted broker
  migration would reintroduce it.
- Vendor lock-in is partial, not total: Amazon MQ's control plane is
  AWS-specific, but the data plane speaks standard AMQP/RabbitMQ, so the
  Trade API and Risk Engine's client code (`pika`/`aio-pika`) is portable.
  Migrating off Amazon MQ later is a Terraform and connection-string change,
  not an application rewrite — a smaller switching cost than ADR-001's
  ECS-to-EKS migration carries.
