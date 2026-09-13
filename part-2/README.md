# Part 2 — AWS ingestion and Tableau architecture

The design separates public source acquisition from private data processing and private analysis. S3 is the target store. This is an architecture proposal; no AWS resources have been deployed.

## Diagrams

- [Single integrated architecture PNG](hdb_aws_architecture_integrated.png) — the end-to-end submission overview
- [Single-page editable draw.io diagram](hdb_aws_architecture_integrated.drawio) — the matching integrated view


Open the `.drawio` file in diagrams.net or the draw.io desktop application. Service boxes, labels, boundaries, and connectors are editable. AWS icons are embedded, so the diagram does not need to fetch them online.

The integrated view covers all four requirements on one canvas and includes the revised HA, retention, transfer-tuning, and audit design. Use it for submission. The assignment does not require separate diagrams.

| Requirement | Design response |
| --- | --- |
| 1. Batch ingestion, including files over 100 MB | Scheduled Fargate jobs stream downloads into S3 multipart uploads, with bounded memory, retries, and a completion manifest. |
| 2. Public source and private HDB VPC | Private ingestion workers reach approved public hosts through an egress proxy and NAT. Processing workers have private AWS endpoint access. |
| 3. Tableau integration through the Athena driver | Tableau Server uses its supported Athena JDBC driver, a dedicated workgroup, the Glue catalog, and an S3 result location. |
| 4. Private exploitation traffic | The Tableau VPC has local Athena and Glue interface endpoints and its own S3 gateway endpoint. The query path needs no internet route or VPC peering. |

## Assumptions

1. Both VPCs and the S3 lake are in one AWS account in `ap-southeast-1`. The example CIDRs are `10.10.0.0/16` and `10.20.0.0/16`. Actual addresses are allocated through the platform network plan.
2. A daily batch meets the freshness requirement. Backfills use the same workflow with bounded concurrency. The five source files are the initial scope; the ingestion design also supports larger files.
3. The data VPC spans two Availability Zones, with a proxy and NAT gateway per AZ and local routes. Tableau has a three-node cluster across three AZs, an internal ALB, and service endpoint ENIs in each AZ. Batch tasks can be rescheduled after an AZ failure. The assumed budget and Tableau licence support this HA topology.
4. Tableau Server runs on private EC2 instances behind an internal HTTPS ALB. Corporate VPN or Direct Connect already exists. The deployed connector must support the selected temporary-credential provider; confirm this against the installed Tableau release and driver before implementation.
5. Source commencement dates remain year-only. Production processing retains the Part 1 assumptions and validation decisions, including quarantine of potential price anomalies. BI users see accepted data; quarantine access requires a separate role.
6. Bucket names in the diagram are examples. Production names must be globally unique. Separate buckets or tightly scoped prefixes isolate raw, accepted, quarantine, audit, and query-result data.

## 1. Batch ingestion and recovery

EventBridge Scheduler starts a Step Functions Standard workflow. The workflow acquires a conditional DynamoDB lock, runs an ingestion task, runs the containerised Part 1 processing task, and publishes the completed dataset location. Step Functions carries dataset IDs, object keys, and manifest references rather than CSV contents. Its ECS integration waits for task completion. [AWS ECS integration](https://docs.aws.amazon.com/step-functions/latest/dg/connect-ecs.html)

The ingestion task discovers dataset IDs, initiates each download, polls for readiness, and retrieves the supplied download URL. API requests use rate-aware backoff and bounded retries. Signed URLs are short-lived credentials and are excluded from logs. Approved file-host domains are checked separately from API hosts; redirects must also pass the allowlist. [data.gov.sg download API](https://guide.data.gov.sg/developer-guide/dataset-apis/download-dataset)

The worker uploads bounded chunks while reading the HTTP response. It does not load the whole file into memory or send file contents through the orchestrator. Multipart uploads permit part-level retries. Part size is chosen for expected file size and adjusted to stay within S3's part-count limit. Unfinished uploads are aborted, with a lifecycle rule as a backstop. [S3 multipart uploads](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html)

The run records the source version or HTTP validators when available, object version, byte count, and SHA-256 digest. A locally computed digest records integrity; it verifies source integrity only when an authoritative source checksum is available for comparison. Multipart ETags are not treated as MD5 checksums. Resume a source download only when the host supports HTTP Range and validators establish that the file is unchanged; otherwise refresh the URL and restart the transfer.

Only completed objects enter the manifest and run ledger. Reruns skip a matching completed source version, rather than assuming that any existing object is valid. Raw objects retain their original bytes. Processing writes to a new run prefix, checks row reconciliation and quality outcomes, and publishes catalog locations only after the output is complete. A failed run leaves the previous published snapshot available.

## 2. Network and system segmentation

Ingestion tasks have no public IP. Their security group permits the egress proxy on TCP 3128, approved interface endpoints on TCP 443, and S3/DynamoDB service prefix lists on TCP 443. The proxy accepts requests only from the ingestion role's worker security group and permits HTTPS CONNECT only to approved source hosts on port 443. The client validates source TLS certificates. The proxy is in a private egress subnet with a default route through the same-AZ NAT gateway in a public subnet; that subnet routes outbound internet traffic through the VPC internet gateway. NAT provides outbound translation and does not expose an inbound worker listener.

The proxy subnet deliberately has no S3 gateway endpoint route. This keeps publicly supplied S3 download URLs on the controlled source egress path rather than subjecting them to an endpoint policy restricted to HDB buckets. Worker subnet route tables have local S3 and DynamoDB gateway routes. ETL workers cannot reach the proxy and have no internet default route. Interface endpoints provide ECR API/DKR, CloudWatch Logs, Secrets Manager, and Glue access. The S3 endpoint policy also permits the ECR image-layer bucket required by Fargate. Fargate does not require an ECS interface endpoint for this configuration. [Fargate endpoint requirements](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/vpc-endpoints.html)

Separate execution, ingestion, processing, and Tableau roles restrict access to their own operations and prefixes. S3 uses Block Public Access, TLS-only access, versioning where recovery is required, and SSE-KMS. Key policies permit the relevant roles and S3 use of the keys. Secrets Manager holds any future source credentials; the current public source needs no API key. Logs omit credentials and signed URLs.

## 3. Tableau and Athena

Install the Athena JDBC driver supported by the deployed Tableau version on every node that executes queries or refreshes extracts. Configure the regional Athena server, the `hdb_tableau` workgroup, and the dedicated S3 staging location. Tableau's connector documents port 443 for API access and port 444 for result streaming. Both ports must be permitted from the Tableau security group to the Athena endpoint security group. [Tableau Athena connector](https://help.tableau.com/current/pro/desktop/en-us/examples_amazonathena.htm), [AWS JDBC requirements](https://docs.aws.amazon.com/athena/latest/ug/jdbc-v3-driver.html)

Use a connector-supported instance-profile credential provider with temporary credentials and IMDSv2. Support must be verified with the actual Tableau/driver combination; do not assume every release exposes the same connection properties. The role permits workgroup-scoped query submission and management, `athena:GetQueryResultsStream`, Glue metadata reads, accepted-data reads, result-prefix reads/writes, and the necessary KMS operations. Avoid permanent access keys in workbook connections.

The workgroup enforces the result location and encryption and applies scan budgets. Query timeouts and concurrency are managed through Athena settings and application controls within service quotas. Result objects have a short, explicit retention period. Dataset access is read-only for Tableau; raw and quarantine data are outside its permitted scope.

## 4. Private queries, metadata, and results

The Tableau VPC has an Athena interface endpoint with private DNS enabled. The standard regional hostname resolves to local private endpoint ENIs, keeping client API and streaming connections inside AWS networking. A Glue interface endpoint supports direct catalog calls made by the driver or application. [Athena PrivateLink](https://docs.aws.amazon.com/athena/latest/ug/interface-vpc-endpoint.html)

The Tableau VPC also has its own S3 gateway endpoint. Drivers may fetch query results directly from S3, so an Athena endpoint alone is insufficient for a fully private result path. S3 gateway endpoints are associated with local route tables and cannot be shared across VPC peering. Each VPC therefore accesses the same authorised lake through its own endpoint; no route between the two VPCs is needed. [JDBC result-fetch behaviour](https://docs.aws.amazon.com/athena/latest/ug/jdbc-v3-driver-advanced-connection-parameters.html), [S3 gateway endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-s3.html)

Athena, Glue, and S3 are managed regional services outside the customer VPCs. Athena reads S3 and writes results through AWS service networking with TLS; these calls do not pass through the Tableau VPC's S3 gateway. [Athena traffic privacy](https://docs.aws.amazon.com/athena/latest/ug/internetwork-traffic-privacy.html)

Bucket policies must distinguish approved direct endpoint access from authorised Athena-mediated requests. Use narrowly scoped principals and the appropriate `aws:CalledVia` / `aws:ViaAWSService` conditions for forward access sessions. A blanket deny for requests lacking a particular `aws:SourceVpce` would block Athena's own S3 access. The managed-service allowance must not become a general permission for other callers. [Athena forward access sessions](https://docs.aws.amazon.com/athena/latest/ug/security-iam-athena-calledvia.html)

## Operating the design

### Transfer configuration and service selection

Start with a 100 MiB multipart threshold, 16 MiB upload parts, four concurrent upload requests per file, two source-file workers, and an SDK connection pool of 16. Explicitly bound the number of buffered and in-flight parts. Increase part size for very large objects to respect S3's 10,000-part limit. Keep the source HTTP stream sequential unless its host supports safe range requests. Reuse HTTPS sessions where the source permits keep-alive, and set connection/read timeouts. Tune using measured throughput, memory, retries, and source throttling; these values are planning defaults, not benchmark results.

The source is a public HTTP API followed by generated download URLs. DataSync's supported storage locations do not provide a generic HTTP API ingestion workflow. Transfer Family's managed file-transfer protocols and browser transfers also do not replace the required metadata/initiate/poll sequence. They would become candidates if the source offered a supported storage location or transfer protocol. [DataSync locations](https://docs.aws.amazon.com/datasync/latest/userguide/working-with-locations.html), [Transfer Family protocols](https://docs.aws.amazon.com/transfer/latest/userguide/what-is-aws-transfer-family.html)

S3 Batch Operations acts on objects already stored in S3, so it is useful for future bulk copy, tagging, or object maintenance rather than fetching this API source. File-level Fargate concurrency handles the present ingestion workload. Step Functions orchestrates execution; Fargate performs the ETL. Lambda could handle short control tasks, but its 15-minute execution limit adds an unnecessary constraint to large downloads and the existing Python processing stage. [S3 Batch Operations](https://docs.aws.amazon.com/AmazonS3/latest/userguide/batch-ops.html), [Lambda timeout](https://docs.aws.amazon.com/lambda/latest/dg/configuration-timeout.html)

### Credentials, metadata, and result reuse

The authentication flow is **EC2 instance profile → IMDSv2 temporary credentials → supported JDBC credential provider → signed Athena/Glue/S3 requests**. The driver refreshes credentials through that provider. This single-account design does not require a driver-issued STS AssumeRole call; an assumed cross-account role would require its own trust policy, permissions, and a regional private STS endpoint. IAM credentials are not stored as permanent keys in Secrets Manager. Secrets Manager rotation applies only if a future authenticated external source introduces a secret.

Every query node reaches the local Athena endpoint. The diagram shows the Glue metadata path through the local Glue endpoint and the direct S3 fetch path through the Tableau VPC gateway. Endpoint security groups permit the cluster's worker security groups, not only one node.

Request Athena result reuse with a maximum age of 60 minutes only when the Tableau-supported driver exposes that option. Reuse is per query within the same workgroup and can return stale data after a source update. Use a versioned analytical view or include the published snapshot identity in the query so a new publication changes the reuse context; if the connector cannot preserve that context, disable reuse. Daily extracts refresh after successful publication and provide the fallback cache. Seven-day S3 result retention supports retrieval; retention alone does not enable caching. [Athena result reuse](https://docs.aws.amazon.com/athena/latest/ug/reusing-query-results.html)

### Availability and recovery

The integrated view now shows three Tableau EC2 nodes in separate AZs. All three run Coordination Service and Client File Service. Repository instances on AZ A and AZ B provide active/passive redundancy; File Store and query/gateway processes are replicated across nodes. The ALB routes private HTTPS traffic to healthy gateways. Configure inter-node access using the deployed Tableau release's documented port matrix. This is a proposed cluster topology, not three independent single-node installations. [Tableau HA requirements](https://help.tableau.com/current/server-linux/en-us/distrib_ha_install_3node.htm)

The planning targets are an RTO of 15 minutes for a node/AZ incident and an RTO of four hours with an RPO of 24 hours for full Tableau recovery from nightly backups. Validate these targets with repository failover, coordination quorum, client reconnection, initial-node recovery, and restore drills. The ALB alone does not ensure application HA. Save both `.tsbak` data backups and exported configuration/topology, together with the installer version and recovery runbook. Keep 35 days of online backups in S3. [Tableau full backup and restore](https://help.tableau.com/current/server-linux/en-us/backup_restore.htm)

All persisted workload, audit, result, and backup data is assumed to remain in Singapore. Same-region versioning and backups support accidental deletion and application recovery; they do not provide recovery from a complete regional outage. No bounded regional-outage RTO is claimed. Cross-region replication and a standby platform require an explicit residency and business-continuity decision and are outside this baseline.

### Retention, partitioning, and cost

These are proposed retention values for the assessment, not statutory requirements:

| Data | Proposed policy |
| --- | --- |
| Latest raw snapshot | Keep online in S3 Standard for replay; retain the latest successfully published source even if older than one year. |
| Retired raw snapshots | Tag after a newer snapshot is published. Transition eligible objects to Standard-IA at 30 days and Glacier Flexible Retrieval at 90 days of object age; expire at 365 days. Apply equivalent noncurrent-version rules. Restore delays apply to archived copies. |
| Active analytical data | Keep published Parquet online. Organise as `run_id=<snapshot>/year=YYYY/month=MM/`; register only published partitions. Remove retired processed versions after 90 days. |
| Quarantine and profiling audit | Retain 90 days with restricted access. |
| Athena query results | Expire after seven days; apply noncurrent-version expiry if versioning is enabled. |
| Tableau backups | Retain 35 days online. |
| Security and query execution audit | Retain 365 days in a separate restricted store. |
| Incomplete multipart uploads | Abort after one day as a lifecycle backstop. |

Do not transition tiny metadata files blindly. Lifecycle object-size filters, transition fees, minimum storage-duration charges, and archive retrieval delays affect the policy. Current queryable Parquet and the latest recovery inputs stay online. [S3 lifecycle considerations](https://docs.aws.amazon.com/AmazonS3/latest/userguide/lifecycle-transition-general-considerations.html)

Planning capacity is five source datasets, a daily refresh, and a provisional ceiling of 10 GB of source downloads per day, with a tenfold growth scenario reviewed over two years. These are sizing assumptions; the Part 1 row counts are not a throughput benchmark. Begin with on-demand Fargate and Athena. Consider EC2 Savings Plans or reservations for the always-on Tableau cluster after measuring usage; Athena provisioned capacity is a later option if sustained concurrency justifies it. Intelligent-Tiering is an alternative for larger objects with uncertain access patterns, subject to monitoring fees; it is not an additional blanket policy on top of the defined raw archive schedule.

### Logging and residency evidence

Enable VPC Flow Logs for both VPCs, CloudTrail management events, and selected S3 object-level data events. Configure S3 server access logs to a separate Singapore landing bucket; classic S3 server-log delivery uses SSE-S3, so this landing bucket is an explicit exception to the main SSE-KMS data/audit buckets. Export or copy logs to the SSE-KMS audit store if required. Server logs are delivered on a best-effort basis and supplement CloudTrail rather than providing complete audit evidence on their own. Exclude the log destination from recursive server logging. [S3 logging options](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ServerLogs.html)

Capture Athena query-state events through EventBridge and use an audit collector, implemented as a scheduled/event-triggered container task, to persist query execution ID, workgroup, caller correlation, SQL, state, timing, scan bytes, and reuse outcome to the separate audit store. Reconcile events against execution history so missed notifications do not silently lose records. SQL can contain sensitive literals and requires restricted access and appropriate redaction. [Athena query events](https://docs.aws.amazon.com/athena/latest/ug/athena-events.html)

Use regional buckets, endpoints, log destinations, and backup locations in `ap-southeast-1`. Enforce permitted regions through platform policy where applicable and verify resource locations and replication settings during deployment. Singapore residency is a design assumption to confirm with HDB, not a claim of legal compliance or protection from every AWS global control-plane operation.

| Concern | Decision |
| --- | --- |
| Scalability | Scale download and ETL tasks independently. Cap download concurrency to respect source limits. Size ETL memory for the complete processing stage; move to distributed processing if growth exceeds a practical single-task limit. |
| Performance | Publish compressed Parquet partitioned by transaction month. Tableau filters should prune months. Compact small files as volume grows; budget live queries and extract refreshes separately. |
| Maintainability | Provision with infrastructure as code. Pin container image digests and driver versions. Version schemas and manifests. Deploy changes through a controlled release process. |
| Recovery | Bounded retries, task timeouts, conditional locks, immutable run prefixes, and publication after validation. Retain raw versions for replay. |
| Observability | CloudWatch alarms for failures and freshness; structured metrics for transfer bytes, duration, quarantine counts, and reconciliation. Use CloudTrail and VPC Flow Logs for access and network evidence. |
| Cost | NAT gateways, per-AZ proxies, interface endpoints, and Tableau instances create a baseline cost. Accept that cost for isolation and availability; control Athena scans, result retention, and idle batch capacity. |

Before production, verify DNS resolves privately, submit and poll a query, fetch results through both streaming and direct S3 paths, browse metadata, and renew temporary credentials. Repeat with no NAT or internet default route in Tableau subnets. Test denial from unauthorised roles and endpoints, source-download interruption, overlapping scheduled runs, and failed ETL publication. These are acceptance checks for the proposed implementation, not claims of tests already performed.

## Icon attribution

Diagrams use the official [AWS Architecture Icons](https://aws.amazon.com/architecture/icons/), July 2026 package. Service/resource artwork is embedded in the editable file; selected original assets are retained in `assets/aws/`. AWS icons identify AWS components. The Tableau component uses the EC2 icon to identify its proposed hosting service.
