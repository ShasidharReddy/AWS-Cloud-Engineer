# AWS CI/CD & DevOps Guide

A comprehensive reference for AWS-native and AWS-adjacent CI/CD, DevOps, infrastructure as code, artifact, and deployment patterns.

## Table of Contents
- [AWS CodePipeline](#aws-codepipeline)
- [AWS CodeBuild](#aws-codebuild)
- [AWS CodeDeploy](#aws-codedeploy)
- [AWS CodeCommit](#aws-codecommit)
- [AWS CodeArtifact](#aws-codeartifact)
- [Amazon ECR](#amazon-ecr)
- [AWS CloudFormation](#aws-cloudformation)
- [AWS CDK](#aws-cdk)
- [AWS SAM](#aws-sam)
- [Terraform on AWS](#terraform-on-aws)
- [GitOps with AWS](#gitops-with-aws)
- [Deployment Strategies](#deployment-strategies)
- [IaC Comparison](#iac-comparison)
- [CI/CD Pipeline Architectures](#cicd-pipeline-architectures)
- [Appendix A - Shared AWS CLI Setup](#appendix-a---shared-aws-cli-setup)
- [Appendix B - Monorepo Reference Diagram](#appendix-b---monorepo-reference-diagram)
- [Appendix C - Microservices Reference Diagram](#appendix-c---microservices-reference-diagram)
- [Appendix D - Multi-Account Promotion Checklist](#appendix-d---multi-account-promotion-checklist)
- [Appendix E - Container Pipeline Reference Diagram](#appendix-e---container-pipeline-reference-diagram)
- [Appendix F - Operational Best Practices](#appendix-f---operational-best-practices)
- [Appendix G - Troubleshooting Commands](#appendix-g---troubleshooting-commands)
- [Appendix H - Governance Summary](#appendix-h---governance-summary)

## Conventions Used in This Document

- AWS CLI examples assume a configured shell profile or role-based access.
- Replace placeholders such as `$ACCOUNT_ID`, `$AWS_REGION`, and resource names with real values.
- Mermaid diagrams use the requested AWS-inspired palette: `fill:#FF9900,color:#232F3E`, `fill:#232F3E,color:#fff`, and `fill:#527FFF,color:#fff`.
- Best practices favor immutable artifacts, least privilege, and automated verification.
- Production releases should be auditable, observable, and reversible.

## AWS CodePipeline

### Why it matters

- AWS CodePipeline orchestrates end-to-end software delivery from source retrieval to production deployment.
- It standardizes stage transitions so teams can enforce repeatable gates across application, infrastructure, and container workflows.
- Artifacts flow between stages by using S3 artifact stores, which enables traceability and deterministic releases.
- Manual approval actions provide auditable change control for production or regulated environments.
- Cross-region and cross-account actions let one central pipeline deploy to shared services, test accounts, and production accounts.

### Mermaid diagram

```mermaid
flowchart LR
    Dev[Developer Commit] --> Source[Source Stage
CodeCommit or GitHub]
    Source --> Build[Build Stage
CodeBuild]
    Build --> Test[Test Stage
Security and Integration]
    Test --> Approve[Manual Approval]
    Approve --> Deploy1[Deploy Stage
Dev Account]
    Approve --> Deploy2[Deploy Stage
Prod Account]
    Artifact[(Artifact Store
Amazon S3)] --> Source
    Artifact --> Build
    Artifact --> Deploy1
    Artifact --> Deploy2
    Region2[Cross-Region Artifact Store] --> Deploy2
    Role[Cross-Account IAM Role] --> Deploy2
    classDef awsOrange fill:#FF9900,color:#232F3E,stroke:#232F3E,stroke-width:1px;
    classDef awsDark fill:#232F3E,color:#fff,stroke:#232F3E,stroke-width:1px;
    classDef awsBlue fill:#527FFF,color:#fff,stroke:#232F3E,stroke-width:1px;
    class Dev,Approve awsOrange;
    class Source,Build,Test,Deploy1,Deploy2 awsBlue;
    class Artifact,Region2,Role awsDark;
```

### Explanation

1. A pipeline contains stages, and each stage contains one or more actions that run in sequence or parallel.
2. Source actions ingest versioned input from repositories or object storage and emit a named output artifact.
3. Build actions compile, package, test, and publish artifacts such as ZIP files, containers, or CloudFormation templates.
4. Approval actions pause execution until an authorized operator approves or rejects the release.
5. Deploy actions target services such as CloudFormation, CodeDeploy, ECS, or custom integrations.
6. For cross-account delivery, the pipeline account assumes roles in target accounts and uses KMS plus S3 policies to protect artifacts.

### Core elements

| Element | Details |
| --- | --- |
| Pipeline | Top-level workflow definition that owns stages, triggers, artifact stores, and execution history. |
| Stage | Logical phase such as source, build, test, approve, or deploy. |
| Action | Unit of work within a stage, for example CodeBuild, CodeDeploy, Lambda invoke, or manual approval. |
| Artifact store | S3 bucket where input and output artifacts are versioned and transferred between actions. |
| Execution | A single run of the pipeline, visible with status, timings, and per-action details. |
| Cross-region support | Separate artifact buckets are required when deploy actions run in secondary regions. |
| Cross-account support | IAM roles and KMS permissions allow a centralized delivery account to deploy elsewhere. |

### AWS CLI commands

```bash
aws codepipeline list-pipelines
aws codepipeline get-pipeline --name my-app-pipeline
aws codepipeline get-pipeline-state --name my-app-pipeline
aws codepipeline start-pipeline-execution --name my-app-pipeline
aws codepipeline list-action-executions --pipeline-name my-app-pipeline
aws codepipeline stop-pipeline-execution --pipeline-name my-app-pipeline --pipeline-execution-id 1111-2222-3333 --abandon false
aws codepipeline put-approval-result --pipeline-name my-app-pipeline --stage-name ApproveProd --action-name ManualApproval --result summary="Approved via CLI",status=Approved --token TOKEN_VALUE
aws s3 mb s3://central-pipeline-artifacts-$ACCOUNT_ID-$AWS_REGION
aws kms create-key --description "KMS key for CodePipeline artifacts"
aws sts assume-role --role-arn arn:aws:iam::222222222222:role/CrossAccountDeployRole --role-session-name codepipeline-deploy
aws codepipeline retry-stage-execution --pipeline-name my-app-pipeline --pipeline-execution-id 1111-2222-3333 --stage-name DeployProd --retry-mode FAILED_ACTIONS
```

### Example pipeline flow notes

```yaml
version: 1
pipeline:
  name: my-app-pipeline
  stages:
    - name: Source
      actions:
        - provider: CodeStarSourceConnection
          output_artifacts: [SourceOutput]
    - name: Build
      actions:
        - provider: CodeBuild
          input_artifacts: [SourceOutput]
          output_artifacts: [BuildOutput]
    - name: Approve
      actions:
        - provider: Manual
    - name: Deploy
      actions:
        - provider: CodeDeployToECS
          input_artifacts: [BuildOutput]
```

### Best practices

- Use separate pipelines or branch filters for trunk, release, and hotfix flows instead of overloading one definition.
- Encrypt artifact buckets with KMS and block public access on both the pipeline bucket and any replicated regional buckets.
- Use manual approvals only where they add clear risk reduction; automate non-production gates aggressively.
- Model environments as distinct stages so the same build artifact is promoted instead of rebuilt.
- Prefer event-driven triggers from repository integrations rather than frequent polling.
- Use least-privilege service roles per action provider and isolate production deployment permissions.
- Capture change metadata in approval messages or SNS notifications for better auditability.
- Export stage and action metrics to CloudWatch alarms for failed executions and long approval wait times.

### Operational metrics to watch

- Pipeline success rate by branch and environment.
- Lead time from source action to production deploy action.
- Average manual approval latency for critical stages.
- Rate of rollback or retry-stage-execution events.
- Artifact promotion time between regions or accounts.
- Number of failed actions grouped by provider type.

### Common pitfalls

- Rebuilding in every environment can produce non-identical artifacts and drift between test and prod.
- Missing KMS grants commonly breaks cross-account artifact decryption.
- Long-running approval steps can cause stale releases and deployment contention.
- Hard-coding region-specific resources in one pipeline makes multi-region delivery brittle.
- Excessive monolithic pipelines slow change throughput and complicate blast-radius control.

### Review checklist

- [ ] Artifact bucket is encrypted, versioned, and lifecycle-managed.
- [ ] Cross-account roles are assumable only by the pipeline service role.
- [ ] Every stage emits a clearly named output artifact.
- [ ] Production stages use approvals, alarms, or automatic rollback controls.
- [ ] Execution notifications are wired to chat, email, or incident systems.
- [ ] Pipeline changes are managed as code through CloudFormation, CDK, or Terraform.

## AWS CodeBuild

### Why it matters

- AWS CodeBuild is a managed build service that compiles source, runs tests, and packages deployable artifacts.
- It supports ephemeral build containers so teams do not manage build servers or patch operating systems.
- Buildspec files define phases, environment variables, reports, and artifact outputs as code.
- Managed compute types let teams balance speed and cost for general workloads, Arm builds, and GPU or Lambda builds.
- Native Docker support makes CodeBuild useful for container images, SBOM generation, and signed build workflows.

### Mermaid diagram

```mermaid
flowchart LR
    Repo[Source Repo] --> Build[CodeBuild Project]
    Build --> Cache[S3 or Local Cache]
    Build --> Docker[Docker Build Engine]
    Build --> Reports[Test Reports]
    Build --> Artifact[Build Artifact
S3 or CodePipeline]
    Docker --> ECR[Amazon ECR]
    classDef awsOrange fill:#FF9900,color:#232F3E,stroke:#232F3E,stroke-width:1px;
    classDef awsDark fill:#232F3E,color:#fff,stroke:#232F3E,stroke-width:1px;
    classDef awsBlue fill:#527FFF,color:#fff,stroke:#232F3E,stroke-width:1px;
    class Repo,Docker awsOrange;
    class Build,Reports,Artifact,ECR awsBlue;
    class Cache awsDark;
```

### Explanation

1. A CodeBuild project defines the source type, environment image, compute type, service role, and artifact destination.
2. The buildspec.yml file controls install, pre_build, build, post_build, reports, and cache behavior.
3. For Docker builds, privileged mode is typically enabled so the build container can run Docker-in-Docker.
4. Caching can be local for fast dependency reuse or S3-based for more portable cache sharing across builds.
5. Report groups store JUnit, Cucumber, or coverage outputs for quality dashboards and pipeline gating.
6. Batch builds and matrix builds help teams test multiple runtimes, regions, or architectures in parallel.

### Core elements

| Element | Details |
| --- | --- |
| Project | Reusable build configuration that defines the runtime image, role, environment variables, and artifacts. |
| buildspec.yml | Source-controlled build definition containing commands, reports, cache settings, and artifact paths. |
| Compute type | Sizing choice such as BUILD_GENERAL1_SMALL, MEDIUM, LARGE, or ARM variants. |
| Environment image | AWS managed or custom container image with the required language toolchain. |
| Cache | Local source, Docker layer, or S3 dependency cache that reduces repeated download time. |
| Reports | Structured test and coverage outputs that surface pass/fail and trend data. |
| Artifacts | ZIP files, templates, binaries, or image metadata emitted to downstream systems. |

### AWS CLI commands

```bash
aws codebuild list-projects
aws codebuild batch-get-projects --names my-app-build
aws codebuild create-project --cli-input-json file://codebuild-project.json
aws codebuild start-build --project-name my-app-build
aws codebuild batch-get-builds --ids my-app-build:1111-2222-3333
aws codebuild create-report-group --name my-app-junit --type TEST --export-config exportConfigType=NO_EXPORT
aws codebuild list-reports-for-report-group --report-group-arn arn:aws:codebuild:$AWS_REGION:$ACCOUNT_ID:report-group/my-app-junit
aws codebuild invalidate-project-cache --project-name my-app-build
aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
aws logs tail /aws/codebuild/my-app-build --follow
aws codebuild stop-build --id my-app-build:1111-2222-3333
```

### Example buildspec.yml

```yaml
version: 0.2
env:
  variables:
    IMAGE_REPO_NAME: my-app
phases:
  install:
    runtime-versions:
      nodejs: 20
    commands:
      - npm ci
  pre_build:
    commands:
      - npm test -- --ci
      - aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
  build:
    commands:
      - docker build -t $IMAGE_REPO_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION .
      - docker tag $IMAGE_REPO_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$IMAGE_REPO_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION
  post_build:
    commands:
      - docker push $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$IMAGE_REPO_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION
reports:
  unit-tests:
    files:
      - junit.xml
cache:
  paths:
    - '/root/.npm/**/*'
artifacts:
  files:
    - imagedefinitions.json
```

### Best practices

- Pin runtime images and toolchain versions so builds remain reproducible over time.
- Use batch builds for matrix testing instead of creating many nearly identical projects.
- Choose the smallest compute type that satisfies runtime and performance targets; scale only when evidence demands it.
- Enable build timeouts, VPC settings, and restrictive service roles to contain risk.
- Store secrets in Secrets Manager or Parameter Store rather than plain environment variables.
- Publish machine-readable reports for unit tests, security scans, and coverage.
- Use S3 cache for dependencies that are expensive to download and local Docker layer cache for image-heavy builds.
- Separate build and deploy responsibilities so one project does not hold unnecessary production credentials.

### Operational metrics to watch

- Median build duration by branch or buildspec version.
- Cache hit ratio for package managers and Docker layers.
- Build failure distribution by phase: install, pre_build, build, or post_build.
- Image push duration and ECR upload throughput.
- Test report pass rate and coverage trends.
- Cost per build minute across compute types.

### Common pitfalls

- Using privileged mode unnecessarily expands the attack surface.
- Very large build containers can hide inefficient dependency management.
- Missing report exports force teams to parse console logs manually.
- Embedding secrets in buildspec files leads to exposure in source control.
- Unbounded caches can become stale, oversized, and difficult to invalidate.

### Review checklist

- [ ] Buildspec is source controlled and peer reviewed.
- [ ] Reports are published for tests and security scans.
- [ ] Cache strategy is explicit and periodically invalidated.
- [ ] Service role has only artifact, log, and registry permissions required.
- [ ] Docker builds use ECR login and immutable tags.
- [ ] CloudWatch log retention is configured for the project.

## AWS CodeDeploy

### Why it matters

- AWS CodeDeploy automates deployments to EC2, on-premises instances, ECS services, and Lambda functions.
- It supports in-place and blue/green strategies so teams can trade speed, cost, and rollback safety.
- Deployment groups define instance targets, alarms, triggers, and rollback behavior.
- The appspec.yml file maps files and lifecycle hooks so custom scripts run in a predictable order.
- Automatic rollback integrates with deployment failures and CloudWatch alarms to protect production systems.

### Mermaid diagram

```mermaid
flowchart LR
    Pipeline[CodePipeline or CLI] --> Deploy[CodeDeploy Application]
    Deploy --> Group[Deployment Group]
    Group --> InPlace[In-Place Deployment
EC2 or On-Prem]
    Group --> BlueGreen[Blue/Green Deployment
EC2 or ECS]
    InPlace --> Hooks[Lifecycle Hooks]
    BlueGreen --> Hooks
    Hooks --> Alarm[CloudWatch Alarms]
    Alarm --> Rollback[Automatic Rollback]
    classDef awsOrange fill:#FF9900,color:#232F3E,stroke:#232F3E,stroke-width:1px;
    classDef awsDark fill:#232F3E,color:#fff,stroke:#232F3E,stroke-width:1px;
    classDef awsBlue fill:#527FFF,color:#fff,stroke:#232F3E,stroke-width:1px;
    class Pipeline,Hooks awsOrange;
    class Deploy,Group,InPlace,BlueGreen,Rollback awsBlue;
    class Alarm awsDark;
```

### Explanation

1. A CodeDeploy application is the logical container for one or more deployment groups and target platforms.
2. Deployment groups select targets by Auto Scaling group, tags, ECS service, or Lambda alias and define health constraints.
3. In-place deployments update existing hosts sequentially or in batches, which is simpler but carries more service disruption risk.
4. Blue/green deployments create or shift traffic to a replacement environment, enabling safer validation and faster rollback.
5. Lifecycle hooks like BeforeInstall, AfterInstall, ApplicationStart, and ValidateService allow deterministic automation around the deployment.
6. Rollback can be triggered automatically when a deployment fails, stops, or trips an attached CloudWatch alarm.

### Core elements

| Element | Details |
| --- | --- |
| Application | Top-level CodeDeploy entity representing a deployable workload. |
| Deployment group | Target selection plus deployment config, alarms, and rollback preferences. |
| Deployment config | Traffic or host update rate such as one at a time, half at a time, or custom minimum healthy hosts. |
| appspec.yml | Revision manifest describing file mappings, permissions, and lifecycle hooks. |
| Lifecycle hooks | Scripts or Lambda functions executed before, during, or after traffic shifting. |
| Blue/green routing | Traffic switch via ELB, ECS task sets, or Lambda aliases for safer cutover. |
| Rollback | Automated restoration to the prior known-good revision when health checks fail. |

### AWS CLI commands

```bash
aws deploy create-application --application-name my-app --compute-platform Server
aws deploy create-deployment-group --application-name my-app --deployment-group-name prod --service-role-arn arn:aws:iam::$ACCOUNT_ID:role/CodeDeployServiceRole --ec2-tag-filters Key=Environment,Value=prod,Type=KEY_AND_VALUE --deployment-config-name CodeDeployDefault.HalfAtATime
aws deploy register-application-revision --application-name my-app --revision revisionType=S3,s3Location={bucket=my-app-revisions,key=releases/app.zip,bundleType=zip}
aws deploy create-deployment --application-name my-app --deployment-group-name prod --s3-location bucket=my-app-revisions,key=releases/app.zip,bundleType=zip --ignore-application-stop-failures
aws deploy get-deployment --deployment-id d-ABCDEF123
aws deploy batch-get-deployments --deployment-ids d-ABCDEF123 d-GHIJKL456
aws deploy list-deployment-groups --application-name my-app
aws deploy stop-deployment --deployment-id d-ABCDEF123 --auto-rollback-enabled
aws deploy get-deployment-config --deployment-config-name CodeDeployDefault.AllAtOnce
aws logs tail /aws/codedeploy-agent/codedeploy-agent.log --follow
```

### Example appspec.yml for EC2

```yaml
version: 0.0
os: linux
files:
  - source: /
    destination: /opt/my-app
hooks:
  ApplicationStop:
    - location: scripts/stop.sh
      timeout: 180
  BeforeInstall:
    - location: scripts/before_install.sh
      timeout: 180
  AfterInstall:
    - location: scripts/after_install.sh
      timeout: 180
  ApplicationStart:
    - location: scripts/start.sh
      timeout: 180
  ValidateService:
    - location: scripts/validate.sh
      timeout: 180
```

### Best practices

- Prefer blue/green for user-facing workloads where rollback time matters more than temporary duplicate capacity cost.
- Make hook scripts idempotent so reruns or retries do not corrupt host state.
- Attach CloudWatch alarms for latency, 5xx errors, and host health to enable automated rollback.
- Store revisions in versioned S3 buckets and keep deployment metadata traceable to a commit SHA.
- Use immutable AMIs or containers when possible; minimize mutable host-level configuration.
- Separate deployment groups by environment and blast radius rather than reusing one massive target set.
- Validate load balancer health checks and deregistration delays before production cutovers.
- Keep CodeDeploy agent health and version current on EC2-based deployments.

### Operational metrics to watch

- Deployment success rate by environment and deployment config.
- Mean time to rollback after a failed release.
- Hook execution duration and failure frequency.
- Traffic shift completion time for ECS or Lambda blue/green releases.
- Host health degradation during in-place updates.
- Alarm-triggered rollback count per application.

### Common pitfalls

- Non-idempotent scripts often break retries and leave instances in inconsistent states.
- Skipping ValidateService removes the last automated guard before customer traffic arrives.
- Using all-at-once on fragile stateful services can cause an avoidable outage.
- Forgetting to update IAM roles breaks lifecycle hooks that call other AWS services.
- Retaining old target groups too long can complicate blue/green cleanup.

### Review checklist

- [ ] Deployment config matches the service risk profile.
- [ ] appspec.yml is versioned with the application revision.
- [ ] Rollback is enabled on failure and on alarm thresholds.
- [ ] Lifecycle scripts are tested outside of CodeDeploy.
- [ ] Target groups and health checks are validated before cutover.
- [ ] Deployment history is visible in CloudWatch or dashboards.

## AWS CodeCommit

### Why it matters

- AWS CodeCommit is a managed Git repository service for teams already invested in AWS-native source control workflows.
- Repositories integrate with IAM, EventBridge, Lambda, and CodePipeline for simple event-driven automation.
- Triggers can start builds, compliance checks, or notifications when branches change.
- Pull request and approval rule features exist for legacy users, but many organizations now prefer GitHub or GitLab for richer SCM ecosystems.
- Because AWS has closed CodeCommit to new customers, migration planning is important for long-term platform strategy.

### Mermaid diagram

```mermaid
flowchart LR
    Dev[Developer Push] --> Repo[CodeCommit Repository]
    Repo --> Trigger[Repository Trigger
SNS or Lambda]
    Repo --> PR[Pull Request Review]
    Trigger --> Pipeline[CodePipeline]
    PR --> Approvals[Approval Rules
Legacy Workflow]
    Repo --> Migration[Migration Path
GitHub or GitLab]
    classDef awsOrange fill:#FF9900,color:#232F3E,stroke:#232F3E,stroke-width:1px;
    classDef awsDark fill:#232F3E,color:#fff,stroke:#232F3E,stroke-width:1px;
    classDef awsBlue fill:#527FFF,color:#fff,stroke:#232F3E,stroke-width:1px;
    class Dev,Approvals awsOrange;
    class Repo,Trigger,PR,Pipeline awsBlue;
    class Migration awsDark;
```

### Explanation

1. A CodeCommit repository behaves like a standard Git remote while inheriting AWS IAM-based authentication and authorization.
2. Repository events can publish to SNS, invoke Lambda, or drive EventBridge rules that start downstream automation.
3. Pull requests offer branch-based review workflows, although ecosystem depth is shallower than GitHub or GitLab.
4. Approval rule templates helped enforce minimum approver counts, but many teams treat them as a legacy pattern and are migrating away.
5. A pragmatic migration plan preserves commit history, branch protections, and CI integrations while retiring CodeCommit safely.
6. For existing environments, CodeCommit can still act as a source stage in CodePipeline until migration is complete.

### Core elements

| Element | Details |
| --- | --- |
| Repository | Git remote hosted by AWS with IAM-backed access control. |
| Branch | Named line of development used for feature, release, and hotfix workflows. |
| Trigger | Event-driven integration that notifies SNS or Lambda on repository changes. |
| Pull request | Review object for merging source and destination branches. |
| Approval rule | Legacy review control that enforces reviewer counts or templates. |
| Credential helper | Git helper that signs HTTPS requests with AWS credentials. |
| Migration target | GitHub or GitLab repository used for future-state source management. |

### AWS CLI commands

```bash
aws codecommit list-repositories
aws codecommit create-repository --repository-name platform-infra --repository-description "Legacy AWS native repo"
aws codecommit get-repository --repository-name platform-infra
aws codecommit put-file --repository-name platform-infra --branch-name main --file-path README.md --file-content fileb://README.md --commit-message "Seed repository"
aws codecommit create-branch --repository-name platform-infra --branch-name release --commit-id COMMIT_SHA
aws codecommit get-branch --repository-name platform-infra --branch-name main
aws codecommit create-pull-request --title "Release merge" --targets repositoryName=platform-infra,sourceReference=release,destinationReference=main
aws codecommit put-repository-triggers --repository-name platform-infra --triggers file://triggers.json
aws codecommit get-pull-request --pull-request-id 1
git remote add github git@github.com:example/platform-infra.git && git push --mirror github
```

### Example trigger definition

```json
[
  {
    "name": "start-pipeline-on-main",
    "destinationArn": "arn:aws:sns:us-east-1:123456789012:repo-events",
    "branches": ["main"],
    "events": ["updateReference"],
    "customData": "main branch updated"
  }
]
```

### Best practices

- Treat CodeCommit as a maintenance or transition platform, not a greenfield default, because service adoption is narrowing.
- Mirror repositories regularly to GitHub or GitLab to preserve optionality and simplify migration rehearsals.
- Use EventBridge or SNS to decouple repository events from downstream build logic.
- Apply least-privilege IAM access for humans, automation roles, and federated identities.
- Standardize branch naming and merge policies before migration so the target SCM inherits clean practices.
- Document approval-rule behavior and map it to branch protection or CODEOWNERS in the destination platform.
- Retain immutable backup bundles or mirrors before making cutover changes.
- Audit webhook, trigger, and PAT usage during the migration inventory.

### Operational metrics to watch

- Push and merge frequency per repository.
- Lead time from pull request creation to merge.
- Number of repositories mirrored to the target SCM.
- Trigger delivery failures to SNS or Lambda.
- Branch divergence between CodeCommit and migration targets.
- Count of repositories still using approval rules.

### Common pitfalls

- Ignoring migration until late can create a large backlog of hidden integrations.
- Custom credential helper setups are often forgotten during developer onboarding.
- Repository triggers that directly invoke brittle scripts are hard to migrate safely.
- Approval rules do not map one-to-one to richer branch protection systems.
- Teams sometimes overlook large-file or LFS behaviors when mirroring history.

### Review checklist

- [ ] Repository access is controlled through IAM roles or federation.
- [ ] Triggers are documented and tested.
- [ ] Migration target and cutover plan are identified.
- [ ] Pull request policies are mapped to GitHub or GitLab equivalents.
- [ ] Repository mirrors are validated periodically.
- [ ] Commit history backups exist before final migration.

## AWS CodeArtifact

### Why it matters

- AWS CodeArtifact is a managed artifact repository for software packages and internal dependency distribution.
- Domains provide a security and governance boundary that can hold multiple repositories.
- Repositories can connect to upstreams and external package ecosystems to create curated dependency flows.
- It supports common formats such as npm, Maven, pip, NuGet, and generic package types through native client integrations.
- Central package controls reduce supply-chain drift and improve provenance across CI systems.

### Mermaid diagram

```mermaid
flowchart LR
    Domain[CodeArtifact Domain] --> RepoA[Team Repository]
    Domain --> RepoB[Shared Base Repository]
    RepoB --> Upstream[Upstream Repository]
    Upstream --> Public[Public Registry]
    RepoA --> NPM[npm Clients]
    RepoA --> Maven[Maven Clients]
    RepoA --> Pip[pip Clients]
    classDef awsOrange fill:#FF9900,color:#232F3E,stroke:#232F3E,stroke-width:1px;
    classDef awsDark fill:#232F3E,color:#fff,stroke:#232F3E,stroke-width:1px;
    classDef awsBlue fill:#527FFF,color:#fff,stroke:#232F3E,stroke-width:1px;
    class Public,Upstream awsOrange;
    class Domain,RepoA,RepoB awsBlue;
    class NPM,Maven,Pip awsDark;
```

### Explanation

1. A domain is the top-level ownership boundary where encryption, sharing, and repository management are coordinated.
2. Repositories store packages directly or proxy upstream sources so build systems can fetch approved dependencies from one place.
3. Upstream repositories help central platform teams control which external packages are cached and allowed downstream.
4. Auth tokens are short-lived, so CI jobs typically run login commands immediately before dependency installation or publish steps.
5. Multiple package formats can coexist in the same domain while following separate repository and permission models.
6. Cross-account domain sharing is useful when a central platform team serves packages to many application accounts.

### Core elements

| Element | Details |
| --- | --- |
| Domain | Security and billing boundary containing one or more repositories. |
| Repository | Concrete package endpoint for publish and install operations. |
| Upstream | Linked repository that allows package retrieval and caching from another source. |
| External connection | AWS-managed connection to public ecosystems such as npmjs or Maven Central. |
| Auth token | Temporary bearer credential used by package managers and CI pipelines. |
| Package format | npm, Maven, pip, NuGet, or generic package type. |
| Policy model | Resource policies plus IAM permissions governing read, publish, and administration. |

### AWS CLI commands

```bash
aws codeartifact create-domain --domain shared-artifacts
aws codeartifact create-repository --domain shared-artifacts --repository team-npm
aws codeartifact create-repository --domain shared-artifacts --repository shared-upstream
aws codeartifact associate-external-connection --domain shared-artifacts --repository shared-upstream --external-connection public:npmjs
aws codeartifact update-repository --domain shared-artifacts --repository team-npm --upstreams repositoryName=shared-upstream
aws codeartifact login --tool npm --domain shared-artifacts --repository team-npm
aws codeartifact login --tool pip --domain shared-artifacts --repository py-internal
aws codeartifact get-repository-endpoint --domain shared-artifacts --repository team-npm --format npm
aws codeartifact list-packages --domain shared-artifacts --repository team-npm
aws codeartifact put-package-origin-configuration --domain shared-artifacts --repository team-npm --format npm --package left-pad --restrictions publish=ALLOW,upstream=BLOCK
```

### Example client setup steps

```bash
export DOMAIN=shared-artifacts
export REPO=team-npm
export ACCOUNT_ID=123456789012
aws codeartifact get-authorization-token   --domain $DOMAIN   --query authorizationToken   --output text
aws codeartifact get-repository-endpoint   --domain $DOMAIN   --repository $REPO   --format npm   --query repositoryEndpoint   --output text
npm publish
pip install --index-url https://aws:$CODEARTIFACT_AUTH_TOKEN@my-domain-123456789012.d.codeartifact.us-east-1.amazonaws.com/pypi/py-internal/simple/ mypackage
```

### Best practices

- Create a shared domain per organization or platform boundary and segregate repositories by lifecycle or team ownership.
- Use upstream repositories and origin controls to reduce direct internet package consumption from CI workers.
- Rotate or fetch auth tokens dynamically inside pipelines; do not store them in long-lived secrets.
- Use package version immutability and provenance scanning where possible.
- Separate public-cache repositories from internal publish targets so governance remains clear.
- Apply resource policies for cross-account consumers instead of sharing broad IAM roles.
- Track stale packages and cache growth to manage cost.
- Document package promotion patterns when releasing approved golden dependencies.

### Operational metrics to watch

- Dependency download volume by repository and package format.
- Cache hit ratio versus external upstream fetches.
- Publish failure rate and token expiration incidents.
- Top internal packages by dependent project count.
- Repository storage growth and stale package count.
- Cross-account access events by domain policy.

### Common pitfalls

- Publishing directly to a shared upstream proxy can blur ownership boundaries.
- Long-lived copied auth tokens often fail unexpectedly in CI when they expire.
- One overly broad repository for every package format becomes hard to govern.
- Teams sometimes forget to lock transitive dependency versions after caching them.
- Unreviewed external connections can reintroduce supply-chain risk.

### Review checklist

- [ ] Domains and repositories are named by ownership and purpose.
- [ ] External connections are explicitly approved.
- [ ] Pipelines fetch short-lived auth tokens at runtime.
- [ ] Internal publish repositories are separate from public upstream proxies.
- [ ] Policies restrict who can publish versus consume.
- [ ] Retention and cleanup strategy is defined.

## Amazon ECR

### Why it matters

- Amazon Elastic Container Registry stores, secures, and distributes OCI-compatible container images and artifacts.
- It integrates tightly with CodeBuild, ECS, EKS, Lambda container images, and image scanning workflows.
- Lifecycle policies help remove stale tags and unreferenced images to control storage cost.
- Enhanced or basic scanning detects vulnerable packages in pushed images.
- Cross-region and cross-account replication makes ECR a strong base for multi-account container delivery.

### Mermaid diagram

```mermaid
flowchart LR
    Build[CI Build] --> Repo[ECR Repository]
    Repo --> Scan[Image Scanning]
    Repo --> Lifecycle[Lifecycle Policy]
    Repo --> Replication[Cross-Region Replication]
    Repo --> Runtime[ECS or EKS Runtime]
    classDef awsOrange fill:#FF9900,color:#232F3E,stroke:#232F3E,stroke-width:1px;
    classDef awsDark fill:#232F3E,color:#fff,stroke:#232F3E,stroke-width:1px;
    classDef awsBlue fill:#527FFF,color:#fff,stroke:#232F3E,stroke-width:1px;
    class Build,Scan awsOrange;
    class Repo,Runtime awsBlue;
    class Lifecycle,Replication awsDark;
```

### Explanation

1. ECR repositories are regional, private by default, and suited for immutable image delivery patterns.
2. Images should be tagged with commit SHAs or build numbers instead of mutable environment-only tags.
3. Lifecycle policies remove old images based on count or age, especially useful for ephemeral feature builds.
4. Scanning surfaces CVE findings that can block promotion or trigger patch automation.
5. Replication rules copy images to target regions or accounts to reduce pull latency and centralize build ownership.
6. Repository policies can grant pull-only access to runtime accounts while keeping push permissions centralized.

### Core elements

| Element | Details |
| --- | --- |
| Repository | Logical collection of images and tags for one service or workload. |
| Image tag | Human-friendly reference such as git SHA, semver, or environment marker. |
| Digest | Immutable content identifier that should back production deployments. |
| Lifecycle policy | Rule set that expires old tags or retains only recent images. |
| Scanning | Basic or enhanced vulnerability analysis for pushed images. |
| Replication | Automatic image copy to other regions or accounts. |
| Repository policy | Access control document for push, pull, and replication permissions. |

### AWS CLI commands

```bash
aws ecr create-repository --repository-name my-app --image-scanning-configuration scanOnPush=true --image-tag-mutability IMMUTABLE
aws ecr describe-repositories
aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
aws ecr describe-images --repository-name my-app
aws ecr start-image-scan --repository-name my-app --image-id imageTag=latest
aws ecr describe-image-scan-findings --repository-name my-app --image-id imageTag=latest
aws ecr put-lifecycle-policy --repository-name my-app --lifecycle-policy-text file://lifecycle-policy.json
aws ecr put-replication-configuration --replication-configuration file://replication.json
aws ecr batch-delete-image --repository-name my-app --image-ids imageTag=old-build-001 imageTag=old-build-002
aws ecr set-repository-policy --repository-name my-app --policy-text file://repository-policy.json
```

### Example lifecycle policy

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep last 30 release images",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["release-"],
        "countType": "imageCountMoreThan",
        "countNumber": 30
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
```

### Best practices

- Use immutable tags and deploy by digest whenever the orchestrator supports it.
- Enable image scanning and fail promotions when critical vulnerabilities remain unaddressed.
- Separate build-time push permissions from runtime pull permissions across accounts.
- Create one repository per service or artifact class to simplify policies and cleanup.
- Use lifecycle policies for preview builds, branch tags, and old release channels.
- Replicate to runtime regions instead of rebuilding images separately in every account.
- Sign images and store SBOM references alongside releases when supply-chain maturity requires it.
- Monitor 429 or permission errors from clusters pulling images at scale.

### Operational metrics to watch

- Image pull latency by region.
- Number of critical findings per promoted image.
- Repository storage growth and stale tag counts.
- Replication lag between build and runtime regions.
- Image pull failure rate from ECS or EKS.
- Percentage of deployments pinned to digests rather than tags.

### Common pitfalls

- Mutable latest tags make rollback and incident analysis harder.
- One shared repository for every service complicates permissions and lifecycle cleanup.
- Ignoring replication design can cause pull latency or cross-region egress surprises.
- Deleting images still referenced by workloads can break autoscaling or rollback.
- Scanning without enforcement only creates noisy security debt.

### Review checklist

- [ ] Repositories use immutable tagging.
- [ ] Scanning is enabled and findings are reviewed.
- [ ] Lifecycle policies are active and tested.
- [ ] Runtime accounts have pull-only access.
- [ ] Critical releases are recorded by digest.
- [ ] Replication covers required runtime regions.

## AWS CloudFormation

### Why it matters

- AWS CloudFormation provides declarative infrastructure as code for native AWS resource provisioning.
- Templates describe desired state while stacks track actual deployments and drift over time.
- Change sets preview modifications before execution, which is valuable for production governance.
- Nested stacks modularize complex systems, and StackSets extend the model across many accounts and regions.
- Drift detection helps identify manual changes that bypass infrastructure pipelines.

### Mermaid diagram

```mermaid
flowchart LR
    Template[Template File] --> ChangeSet[Change Set]
    ChangeSet --> Stack[CloudFormation Stack]
    Stack --> Nested[Nested Stacks]
    Stack --> Drift[Drift Detection]
    Stack --> StackSet[StackSets
Multi-Account or Multi-Region]
    classDef awsOrange fill:#FF9900,color:#232F3E,stroke:#232F3E,stroke-width:1px;
    classDef awsDark fill:#232F3E,color:#fff,stroke:#232F3E,stroke-width:1px;
    classDef awsBlue fill:#527FFF,color:#fff,stroke:#232F3E,stroke-width:1px;
    class Template,ChangeSet awsOrange;
    class Stack,Nested,StackSet awsBlue;
    class Drift awsDark;
```

### Explanation

1. Templates can be written in YAML or JSON and define resources, parameters, mappings, conditions, and outputs.
2. A stack is the deployable instance of a template, and it maintains resource state plus event history.
3. Change sets compute the impact of a proposed update before it touches live resources.
4. Nested stacks let large systems share reusable building blocks while preserving dependency graphs.
5. StackSets push standardized templates to many accounts and regions with delegated administration support.
6. Drift detection identifies resources whose live configuration no longer matches the stack template.

### Core elements

| Element | Details |
| --- | --- |
| Template | Declarative document describing resources and related metadata. |
| Stack | Runtime instance of a template with lifecycle events and outputs. |
| Change set | Preview of create, modify, replace, or delete actions during an update. |
| Nested stack | Reusable child stack referenced from a parent template. |
| StackSet | Fleet deployment mechanism across multiple accounts or regions. |
| Drift detection | Comparison of actual resource state to the last deployed template state. |
| Rollback | Automatic failure behavior that cleans up partial deployments when enabled. |

### AWS CLI commands

```bash
aws cloudformation validate-template --template-body file://template.yaml
aws cloudformation create-stack --stack-name network-core --template-body file://template.yaml --capabilities CAPABILITY_NAMED_IAM
aws cloudformation describe-stacks --stack-name network-core
aws cloudformation create-change-set --stack-name network-core --change-set-name update-001 --template-body file://template.yaml --capabilities CAPABILITY_NAMED_IAM
aws cloudformation describe-change-set --stack-name network-core --change-set-name update-001
aws cloudformation execute-change-set --stack-name network-core --change-set-name update-001
aws cloudformation detect-stack-drift --stack-name network-core
aws cloudformation describe-stack-resource-drifts --stack-name network-core
aws cloudformation create-stack-set --stack-set-name baseline-iam --template-body file://stackset.yaml --capabilities CAPABILITY_NAMED_IAM
aws cloudformation create-stack-instances --stack-set-name baseline-iam --accounts 111111111111 222222222222 --regions us-east-1 us-west-2
```

### Example nested-stack parent template fragment

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Parent stack calling a nested network stack
Resources:
  NetworkStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/my-bucket/templates/network.yaml
      Parameters:
        EnvironmentName: prod
Outputs:
  VpcId:
    Value: !GetAtt NetworkStack.Outputs.VpcId
```

### Best practices

- Validate templates and inspect change sets before production execution.
- Use nested stacks or modules for repeatable patterns instead of massive single templates.
- Prefer parameters for environment-specific values and exports or SSM for shared references.
- Enable termination protection on critical stacks and retain policies on irreplaceable data stores.
- Use StackSets for organization-wide guardrails such as IAM roles, logging, or Config rules.
- Run drift detection routinely on production stacks to catch manual changes.
- Keep templates under source control and promote them through environments using the same artifact.
- Tag stacks and resources consistently for cost, ownership, and operational visibility.

### Operational metrics to watch

- Change set approval lead time.
- Stack creation and update duration by template family.
- Drift count and time to remediation.
- StackSet operation failure rate across accounts.
- Rollback frequency and root cause categories.
- Template size and nested-stack reuse ratio.

### Common pitfalls

- Large monolithic stacks are difficult to update and recover.
- Manual console edits create drift that surprises later deployments.
- Replacing stateful resources unintentionally can cause outages or data loss.
- Ignoring stack exports and dependencies can create coupling deadlocks.
- StackSet rollouts without account prerequisites often fail at scale.

### Review checklist

- [ ] Templates validate successfully.
- [ ] Production changes use reviewed change sets.
- [ ] Critical stacks have termination protection.
- [ ] Nested stacks or modular design are used where complexity grows.
- [ ] Drift detection is part of operations.
- [ ] Outputs and tags are standardized.

## AWS CDK

### Why it matters

- AWS CDK lets teams define cloud infrastructure in familiar programming languages while synthesizing to CloudFormation.
- It improves reuse through constructs and type-safe composition patterns.
- L1, L2, and L3 constructs offer different abstraction levels from raw resources to opinionated patterns.
- CDK apps can contain multiple stacks, and CDK Pipelines can automate self-mutating infrastructure delivery.
- Because CDK still emits CloudFormation, teams retain access to stacks, drift checks, and change review workflows.

### Mermaid diagram

```mermaid
flowchart LR
    App[CDK App] --> L1[L1 Constructs
Cfn*]
    App --> L2[L2 Constructs
Higher Level APIs]
    App --> L3[L3 Constructs
Patterns]
    L1 --> Synth[Cloud Assembly]
    L2 --> Synth
    L3 --> Synth
    Synth --> CFN[CloudFormation Stacks]
    CFN --> Pipe[CDK Pipelines]
    classDef awsOrange fill:#FF9900,color:#232F3E,stroke:#232F3E,stroke-width:1px;
    classDef awsDark fill:#232F3E,color:#fff,stroke:#232F3E,stroke-width:1px;
    classDef awsBlue fill:#527FFF,color:#fff,stroke:#232F3E,stroke-width:1px;
    class App,Synth awsOrange;
    class L1,L2,L3,CFN,Pipe awsBlue;
```

### Explanation

1. A CDK app is the executable entry point that instantiates one or more stacks and constructs.
2. L1 constructs mirror CloudFormation resources closely, which is useful when new AWS features appear first at the CFN layer.
3. L2 constructs wrap common defaults and convenience methods around L1 resources for a more ergonomic developer experience.
4. L3 constructs combine multiple resources into opinionated patterns such as load-balanced services or secure websites.
5. The synth step generates a cloud assembly and CloudFormation templates that can be reviewed and deployed.
6. CDK Pipelines add self-mutating delivery so infrastructure definitions update their own pipeline when the source changes.

### Core elements

| Element | Details |
| --- | --- |
| App | Program entry point that groups stacks and configuration context. |
| Stack | Deployable unit that synthesizes to one CloudFormation stack. |
| Construct | Reusable building block with defined scope, id, and properties. |
| L1 | Direct CloudFormation-level construct with minimal abstraction. |
| L2 | Curated construct with sensible defaults and helper methods. |
| L3 | Opinionated pattern combining multiple constructs into one solution. |
| CDK Pipeline | Pipeline pattern that automates synth, assets, and deployment waves. |

### AWS CLI commands

```bash
aws sts get-caller-identity
aws cloudformation describe-stacks --stack-name CDKToolkit
aws ssm get-parameter --name /cdk-bootstrap/hnb659fds/version
npx cdk bootstrap aws://$ACCOUNT_ID/$AWS_REGION
npx cdk synth
npx cdk diff
npx cdk deploy PlatformStack
npx cdk destroy SandboxStack
aws cloudformation list-stack-resources --stack-name PlatformStack
aws cloudformation describe-stack-events --stack-name PlatformStack
```

### Example CDK stack fragment

```ts
import * as cdk from 'aws-cdk-lib';
import * as s3 from 'aws-cdk-lib/aws-s3';

export class PlatformStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    new s3.Bucket(this, 'ArtifactsBucket', {
      versioned: true,
      encryption: s3.BucketEncryption.S3_MANAGED,
      enforceSSL: true,
    });
  }
}
```

### Best practices

- Keep constructs small and reusable; publish internal construct libraries when patterns stabilize.
- Use L2 constructs by default and fall back to L1 only when you need an unsupported or low-level property.
- Treat cdk synth and cdk diff outputs as review artifacts in CI.
- Bootstrap accounts in a controlled manner and understand the trust model of the CDK toolkit stack.
- Pin CDK versions to avoid unexpected synthesis changes between environments.
- Separate application stacks, platform stacks, and experimental stacks for cleaner lifecycle management.
- Use context, SSM parameters, or environment configuration sparingly and document resolution order.
- For regulated environments, combine CDK with change management around the generated CloudFormation.

### Operational metrics to watch

- Construct reuse across repositories.
- Number of manual template edits avoided through construct libraries.
- Synth or diff failures caught before deployment.
- Pipeline self-mutation success rate.
- Bootstrap drift across accounts.
- Time from infrastructure code change to deployed stack.

### Common pitfalls

- Embedding too much business logic in app code makes infrastructure intent harder to review.
- Implicit defaults can hide security settings unless teams read generated templates.
- Unpinned context lookups may synthesize different templates over time.
- Mixing many environments in one app without boundaries leads to accidental deployments.
- Ignoring the CloudFormation layer makes troubleshooting stack failures harder.

### Review checklist

- [ ] Bootstrap version is current.
- [ ] CDK dependencies are pinned.
- [ ] Synth and diff are part of CI.
- [ ] Construct abstraction level is intentional.
- [ ] Generated templates are reviewable.
- [ ] Pipeline or deployment permissions are scoped correctly.

## AWS SAM

### Why it matters

- AWS Serverless Application Model simplifies building and deploying Lambda-centric applications on AWS.
- SAM templates extend CloudFormation with shorthand syntax for common serverless resources.
- sam local supports local invocation and API emulation for faster iteration before cloud deployment.
- sam deploy packages artifacts and creates or updates the underlying CloudFormation stack.
- SAM Accelerate reduces feedback time by syncing code changes to the cloud during active development.

### Mermaid diagram

```mermaid
flowchart LR
    Template[SAM Template] --> Build[sam build]
    Build --> Local[sam local invoke or start-api]
    Build --> Deploy[sam deploy]
    Deploy --> CFN[CloudFormation Stack]
    CFN --> Lambda[Lambda and API Gateway]
    Build --> Accelerate[SAM Accelerate
sam sync --watch]
    classDef awsOrange fill:#FF9900,color:#232F3E,stroke:#232F3E,stroke-width:1px;
    classDef awsDark fill:#232F3E,color:#fff,stroke:#232F3E,stroke-width:1px;
    classDef awsBlue fill:#527FFF,color:#fff,stroke:#232F3E,stroke-width:1px;
    class Template,Build awsOrange;
    class Local,Deploy,CFN,Lambda awsBlue;
    class Accelerate awsDark;
```

### Explanation

1. SAM templates add resource types like AWS::Serverless::Function and AWS::Serverless::Api that expand into CloudFormation.
2. sam build resolves dependencies and prepares deployment artifacts locally or in containerized build environments.
3. sam local invoke and sam local start-api enable functional testing with local containers and event payloads.
4. sam deploy packages code to S3, creates a change set, and updates the target CloudFormation stack.
5. SAM Accelerate keeps a tight development loop by syncing code changes rapidly to the cloud for managed integration testing.
6. Because SAM is CloudFormation-based, teams can still use stack outputs, change control, and drift-aware operations.

### Core elements

| Element | Details |
| --- | --- |
| Template | YAML definition for functions, APIs, layers, and permissions. |
| sam build | Compilation and dependency packaging step for local or CI workflows. |
| sam local | Local emulation for function invocation and HTTP APIs. |
| sam deploy | Guided or scripted deployment command that updates CloudFormation stacks. |
| SAM Accelerate | Fast sync mode for iterative development with cloud-backed resources. |
| Artifacts bucket | S3 location where packaged code is uploaded before deployment. |
| Guided config | samconfig.toml file holding environment-specific deploy settings. |

### AWS CLI commands

```bash
sam build --use-container
sam local invoke HelloWorldFunction --event events/event.json
sam local start-api
sam deploy --guided
sam sync --watch
aws cloudformation describe-stacks --stack-name sam-app-prod
aws lambda invoke --function-name sam-app-prod-HelloWorldFunction response.json
aws logs tail /aws/lambda/sam-app-prod-HelloWorldFunction --follow
aws apigatewayv2 get-apis
aws cloudformation describe-stack-events --stack-name sam-app-prod
```

### Example SAM template anatomy

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Resources:
  HelloWorldFunction:
    Type: AWS::Serverless::Function
    Properties:
      Runtime: python3.12
      Handler: app.lambda_handler
      CodeUri: src/
      Timeout: 10
      Events:
        ApiEvent:
          Type: Api
          Properties:
            Path: /hello
            Method: get
Outputs:
  ApiUrl:
    Value: !Sub 'https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/hello'
```

### Best practices

- Keep template resources minimal and isolate reusable utilities into Lambda layers or shared libraries.
- Use sam build in CI to match the packaging logic used by developers locally.
- Commit samconfig defaults carefully and override secrets or account-specific values via CI variables.
- Use SAM Accelerate for fast inner loops, but validate final templates through full deploy workflows before release.
- Test IAM policies, timeouts, and cold-start-sensitive settings with realistic integration traffic.
- Log structured JSON and trace request correlation IDs for serverless observability.
- Package dependencies in a deterministic way and scan artifacts before deployment.
- Set reserved concurrency or alarms where serverless scaling risks downstream systems.

### Operational metrics to watch

- Build-to-deploy time for serverless changes.
- Local invocation parity issues versus cloud behavior.
- Cold start and duration trends by function version.
- Error rate after sam sync or production deploy.
- Change set size and stack update duration.
- Artifact package size growth over time.

### Common pitfalls

- Local emulation can differ from managed service integrations if dependencies are mocked too heavily.
- Large deployment packages slow down iterative loops.
- Unchecked environment drift across samconfig profiles causes confusing deployments.
- Serverless shorthand can hide expanded CloudFormation resources from inexperienced reviewers.
- Rapid sync workflows can bypass normal review if used outside development guardrails.

### Review checklist

- [ ] Template validates and deploys through CloudFormation.
- [ ] Local events cover primary code paths.
- [ ] samconfig profiles are documented.
- [ ] Logs, metrics, and alarms exist for functions and APIs.
- [ ] Accelerate is limited to appropriate environments.
- [ ] Artifact bucket and IAM permissions are secured.

## Terraform on AWS

### Why it matters

- Terraform is a popular multi-cloud infrastructure as code tool that works well with AWS providers and modules.
- Teams often use it on AWS when they want a common workflow across cloud providers or non-AWS platforms.
- Remote state in S3 plus state locking in DynamoDB is the standard pattern for collaborative AWS usage.
- Modules encourage reuse and clear interfaces between foundational and application-level infrastructure.
- Terraform plans act as a reviewable diff before apply, which fits CI-driven change management.

### Mermaid diagram

```mermaid
flowchart LR
    HCL[Terraform HCL] --> Provider[AWS Provider]
    HCL --> Modules[Modules]
    Provider --> Plan[terraform plan]
    Plan --> Apply[terraform apply]
    Apply --> AWS[AWS Resources]
    HCL --> State[S3 State File]
    State --> Lock[DynamoDB Lock Table]
    classDef awsOrange fill:#FF9900,color:#232F3E,stroke:#232F3E,stroke-width:1px;
    classDef awsDark fill:#232F3E,color:#fff,stroke:#232F3E,stroke-width:1px;
    classDef awsBlue fill:#527FFF,color:#fff,stroke:#232F3E,stroke-width:1px;
    class HCL,Plan awsOrange;
    class Provider,Modules,Apply,AWS awsBlue;
    class State,Lock awsDark;
```

### Explanation

1. The AWS provider translates Terraform resources and data sources into AWS API operations.
2. Remote backend configuration stores state files in S3 so teams have a durable source of truth.
3. A DynamoDB table is commonly used to lock state and prevent concurrent applies from corrupting infrastructure.
4. Modules encapsulate repeatable patterns such as VPCs, ECS services, or observability stacks.
5. The plan phase previews intended changes, and the apply phase executes them once approved.
6. Terraform can coexist with CloudFormation or CDK, but ownership boundaries must be explicit to avoid resource conflicts.

### Core elements

| Element | Details |
| --- | --- |
| Provider | Plugin that maps Terraform resource definitions to AWS APIs. |
| Backend | State storage configuration, commonly S3 for AWS usage. |
| State lock | DynamoDB row used to serialize writes to shared state. |
| Module | Reusable collection of Terraform resources with variable inputs and outputs. |
| Workspace | State partitioning mechanism often used carefully for environment isolation. |
| Plan | Preview of adds, changes, and destroys before apply. |
| Apply | Execution step that mutates AWS resources to match desired state. |

### AWS CLI commands

```bash
aws s3api create-bucket --bucket tf-state-$ACCOUNT_ID-$AWS_REGION --region $AWS_REGION --create-bucket-configuration LocationConstraint=$AWS_REGION
aws dynamodb create-table --table-name tf-state-lock --attribute-definitions AttributeName=LockID,AttributeType=S --key-schema AttributeName=LockID,KeyType=HASH --billing-mode PAY_PER_REQUEST
aws s3api put-bucket-versioning --bucket tf-state-$ACCOUNT_ID-$AWS_REGION --versioning-configuration Status=Enabled
aws kms create-key --description "Terraform state encryption key"
terraform init
terraform validate
terraform plan -out tfplan
terraform apply tfplan
terraform state list
aws s3 ls s3://tf-state-$ACCOUNT_ID-$AWS_REGION
aws dynamodb describe-table --table-name tf-state-lock
```

### Example backend and provider configuration

```hcl
terraform {
  backend "s3" {
    bucket         = "tf-state-123456789012-us-east-1"
    key            = "platform/network/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "tf-state-lock"
    encrypt        = true
  }
}

provider "aws" {
  region = "us-east-1"
}
```

### Best practices

- Use a dedicated state bucket and lock table per security boundary or platform domain.
- Turn on bucket versioning, encryption, access logging, and restrictive bucket policies for state storage.
- Keep modules focused with stable interfaces and semantic versioning.
- Review terraform plan output in CI and avoid direct applies from laptops for production.
- Prefer one state file per stack or bounded context rather than one giant state.
- Import or move resources carefully when migrating ownership from other tools.
- Store provider credentials outside the codebase and use STS or OIDC for CI access.
- Document resource ownership boundaries when Terraform coexists with CloudFormation or CDK.

### Operational metrics to watch

- Plan approval lead time.
- Apply duration and failure rate.
- State file growth and module count.
- Drift detected through plan runs against unchanged code.
- Frequency of state lock contention.
- Reuse rate of internal modules.

### Common pitfalls

- Using local state for team-shared environments causes frequent conflicts.
- Overusing workspaces for unrelated environments can create operational confusion.
- Large shared state files magnify blast radius and slow plans.
- Manually editing state is risky and should be heavily controlled.
- Mixed ownership of the same AWS resource across tools leads to drift and outages.

### Review checklist

- [ ] Remote state bucket is encrypted and versioned.
- [ ] DynamoDB locking is enabled.
- [ ] Plans are reviewed before apply.
- [ ] Modules have clear inputs and outputs.
- [ ] Provider versions are pinned.
- [ ] Ownership boundaries with other IaC tools are explicit.

## GitOps with AWS

### Why it matters

- GitOps uses Git as the declarative source of truth for Kubernetes and platform configuration.
- On AWS, ArgoCD and Flux are common GitOps controllers for EKS-based delivery.
- The model emphasizes pull-based reconciliation from cluster to Git instead of direct imperative kubectl changes.
- GitOps improves auditability, rollback, and environment consistency for cluster workloads.
- It pairs well with image automation, policy checks, and multi-cluster operations.

### Mermaid diagram

```mermaid
flowchart LR
    Git[Git Repository] --> Controller[ArgoCD or Flux]
    Controller --> EKS[Amazon EKS Cluster]
    EKS --> Workloads[Kubernetes Workloads]
    Controller --> Policy[Policy and Health Checks]
    Image[ECR Image Updates] --> Git
    classDef awsOrange fill:#FF9900,color:#232F3E,stroke:#232F3E,stroke-width:1px;
    classDef awsDark fill:#232F3E,color:#fff,stroke:#232F3E,stroke-width:1px;
    classDef awsBlue fill:#527FFF,color:#fff,stroke:#232F3E,stroke-width:1px;
    class Git,Image awsOrange;
    class Controller,EKS,Workloads awsBlue;
    class Policy awsDark;
```

### Explanation

1. A GitOps controller continuously compares the desired state in Git with the live state in the EKS cluster.
2. ArgoCD focuses on application-centric deployment and visual synchronization status, while Flux emphasizes composable controllers and Git-native automation.
3. Controllers pull manifests or Helm charts from Git and apply them to the cluster with reconciliation loops.
4. Image automation can update Git manifests when new ECR images pass policy and test gates.
5. Cluster access is narrowed because operators change Git, and the controller performs the apply inside the cluster boundary.
6. Platform teams often combine GitOps with policy engines, admission controls, and progressive delivery tools.

### Core elements

| Element | Details |
| --- | --- |
| Git source | Repository containing manifests, Helm releases, or Kustomize overlays. |
| Controller | ArgoCD or Flux component that reconciles desired and actual cluster state. |
| EKS cluster | Managed Kubernetes control plane hosting GitOps agents and workloads. |
| Image automation | Controller that updates manifests when new approved image tags or digests exist. |
| Policy layer | OPA Gatekeeper, Kyverno, or admission controls enforcing standards. |
| Promotion repo | Repository layout used to separate dev, staging, and production overlays. |
| Drift correction | Automated reapplication of Git-defined state when manual changes occur. |

### AWS CLI commands

```bash
aws eks describe-cluster --name platform-eks
aws eks update-kubeconfig --name platform-eks --region $AWS_REGION
aws ecr describe-images --repository-name my-app
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd --namespace argocd --create-namespace
helm repo add fluxcd https://fluxcd-community.github.io/helm-charts
helm install flux fluxcd/flux2 --namespace flux-system --create-namespace
kubectl get applications -n argocd
kubectl get gitrepositories -A
kubectl get kustomizations -A
```

### Example GitOps repository layout

```yaml
clusters/
  prod/
    kustomization.yaml
    apps/
      payments.yaml
      orders.yaml
  stage/
    kustomization.yaml
apps/
  payments/
    base/
    overlays/
  orders/
    base/
    overlays/
```

### Best practices

- Separate platform, shared services, and application repositories or directories to keep blast radius understandable.
- Prefer digest-based image promotion and let automation update Git only after validation passes.
- Protect production branches aggressively because a merge effectively becomes a deployment intent.
- Use namespaces, RBAC, and repository segmentation to scope controller permissions.
- Adopt progressive delivery with Argo Rollouts or Flagger when workloads need canary or blue/green behavior.
- Implement policy checks in CI and again at cluster admission time.
- Back up cluster GitOps metadata and controller secrets where required.
- Keep manual kubectl apply access tightly restricted and observable.

### Operational metrics to watch

- Reconciliation lag between Git commit and cluster sync.
- Drift correction frequency.
- Production deployment success rate by controller or app.
- Time to promote image digests across environments.
- Policy rejection count by namespace or team.
- Manual cluster change incidents outside GitOps.

### Common pitfalls

- Mixing imperative changes with GitOps leads to confusing drift battles.
- Repository layouts that mirror org charts instead of runtime boundaries become hard to operate.
- Unpinned image tags undermine deterministic reconciliation.
- Controller permissions that are too broad turn Git mistakes into cluster-wide outages.
- Ignoring secret management creates plaintext config exposure in Git.

### Review checklist

- [ ] EKS access and kubeconfig are centrally managed.
- [ ] Git repository structure maps clearly to environments.
- [ ] Controllers run with least privilege.
- [ ] Image promotion uses tags or digests with policy gates.
- [ ] Drift is monitored and reported.
- [ ] Emergency break-glass procedures are documented.

## Deployment Strategies

### Why it matters

- Deployment strategy choice determines release risk, rollback speed, cost, and operational complexity.
- Rolling, blue/green, canary, and all-at-once patterns are all valid when matched to the right workload profile.
- AWS services such as CodeDeploy, ECS, Lambda aliases, and load balancers can implement each pattern differently.
- A good strategy balances user impact, capacity overhead, database compatibility, and observability readiness.
- Teams should standardize a default strategy and document exceptions based on service criticality.

### Mermaid diagram

```mermaid
flowchart TB
    Start[Release Decision] --> Rolling[Rolling]
    Start --> BlueGreen[Blue or Green]
    Start --> Canary[Canary]
    Start --> AllAtOnce[All at Once]
    Rolling --> R1[Incremental host batches]
    BlueGreen --> B1[Parallel environments and traffic switch]
    Canary --> C1[Small traffic shift then expand]
    AllAtOnce --> A1[Immediate full cutover]
    classDef awsOrange fill:#FF9900,color:#232F3E,stroke:#232F3E,stroke-width:1px;
    classDef awsDark fill:#232F3E,color:#fff,stroke:#232F3E,stroke-width:1px;
    classDef awsBlue fill:#527FFF,color:#fff,stroke:#232F3E,stroke-width:1px;
    class Start awsOrange;
    class Rolling,BlueGreen,Canary,AllAtOnce awsBlue;
    class R1,B1,C1,A1 awsDark;
```

### Explanation

1. Rolling updates replace instances or tasks in batches, which conserves capacity but prolongs mixed-version operation.
2. Blue/green keeps old and new environments side by side until traffic moves, which makes rollback fastest at the cost of duplicate capacity.
3. Canary sends a small percentage of traffic to the new version first, making it ideal when observability is strong and progressive risk reduction matters.
4. All-at-once switches everything immediately and should generally be reserved for stateless low-risk or easily recoverable workloads.
5. Database and contract compatibility often determine whether progressive strategies are safe.
6. Observability, alarms, and rollback automation matter more than the tool name used to implement the strategy.

### Core elements

| Element | Details |
| --- | --- |
| Rolling | Sequential or percentage-based replacement of tasks, pods, or instances. |
| Blue/green | New environment stands up separately, then traffic shifts from blue to green. |
| Canary | Very small production exposure before progressively expanding to full traffic. |
| All-at-once | Every target updates simultaneously for shortest deployment time. |
| Health checks | Load balancer or application checks that gate progression. |
| Rollback trigger | Alarm, failed hook, or manual stop that returns traffic to the stable version. |
| Compatibility plan | Schema and API strategy ensuring old and new versions can coexist as needed. |

### AWS CLI commands

```bash
aws deploy get-deployment-config --deployment-config-name CodeDeployDefault.AllAtOnce
aws deploy get-deployment-config --deployment-config-name CodeDeployDefault.HalfAtATime
aws ecs update-service --cluster prod --service web --force-new-deployment
aws lambda update-alias --function-name payments --name live --routing-config AdditionalVersionWeights={2=0.1}
aws elbv2 modify-listener --listener-arn arn:aws:elasticloadbalancing:... --default-actions Type=forward,ForwardConfig={TargetGroups=[{TargetGroupArn=arn:aws:elasticloadbalancing:...blue,Weight=90},{TargetGroupArn=arn:aws:elasticloadbalancing:...green,Weight=10}]}
aws cloudwatch describe-alarms --alarm-names prod-5xx-alarm prod-latency-alarm
aws deploy stop-deployment --deployment-id d-ABCDEF123 --auto-rollback-enabled
```

### Strategy selection guide

```markdown
- Choose rolling when capacity is limited and the service tolerates mixed versions.
- Choose blue/green when rollback speed and release validation are critical.
- Choose canary when telemetry is excellent and you want gradual confidence.
- Choose all-at-once only for low-risk, non-critical, or easily reversible changes.
```

### Best practices

- Default to blue/green or canary for customer-facing production services when cost permits.
- Keep schema changes backward compatible so old and new code can run concurrently during progressive rollouts.
- Define clear rollback thresholds tied to latency, error rate, saturation, and business KPIs.
- Automate traffic shifting where possible and minimize ad hoc manual load balancer changes.
- Run post-deploy smoke checks that validate real user journeys, not only instance health.
- Document the strategy per service so on-call engineers know expected behavior during incidents.
- Test rollback paths regularly, not only the forward deploy path.
- Align deployment strategy with auto scaling and warm-up behavior.

### Operational metrics to watch

- Error rate during each rollout phase.
- Time to detect a bad release.
- Time to complete rollback.
- Capacity overhead required for safer strategies.
- Number of mixed-version compatibility incidents.
- Customer impact minutes per failed deployment.

### Common pitfalls

- Choosing all-at-once for stateful critical services invites avoidable outages.
- Progressive rollout without alarms only delays failure discovery.
- Blue/green cost surprises happen when duplicate infrastructure is not planned.
- Canary metrics can be statistically noisy if traffic volume is too low.
- Database migrations that are not backward compatible can break every strategy.

### Review checklist

- [ ] Strategy is documented per service.
- [ ] Rollback commands and owners are known.
- [ ] Health metrics gate traffic progression.
- [ ] Schema compatibility is validated.
- [ ] Smoke tests exist for deploy verification.
- [ ] Capacity planning covers the chosen strategy.

## IaC Comparison

### Why it matters

- Choosing an IaC tool affects team skills, governance workflows, abstraction levels, and ecosystem integration.
- CloudFormation, CDK, Terraform, and Pulumi solve similar problems but with different operational trade-offs.
- The right choice depends on whether teams prefer AWS-native control, general-purpose languages, multi-cloud scope, or component reuse models.
- Many organizations use more than one tool, so explicit ownership boundaries are crucial.
- A comparison framework helps platform teams standardize defaults without blocking valid exceptions.

### Mermaid diagram

```mermaid
flowchart LR
    Need[Decision Need] --> CFN[CloudFormation]
    Need --> CDK[CDK]
    Need --> TF[Terraform]
    Need --> Pulumi[Pulumi]
    CFN --> Native[AWS Native Control]
    CDK --> Code[General Purpose Languages]
    TF --> Multi[Multi-Cloud Workflow]
    Pulumi --> SDK[Code First SDKs]
    classDef awsOrange fill:#FF9900,color:#232F3E,stroke:#232F3E,stroke-width:1px;
    classDef awsDark fill:#232F3E,color:#fff,stroke:#232F3E,stroke-width:1px;
    classDef awsBlue fill:#527FFF,color:#fff,stroke:#232F3E,stroke-width:1px;
    class Need awsOrange;
    class CFN,CDK,TF,Pulumi awsBlue;
    class Native,Code,Multi,SDK awsDark;
```

### Explanation

1. CloudFormation is the most AWS-native option and integrates directly with stacks, change sets, StackSets, and service launch features.
2. CDK builds on CloudFormation but increases developer ergonomics through familiar languages and constructs.
3. Terraform offers strong multi-cloud consistency and a large module ecosystem, which appeals to heterogeneous platforms.
4. Pulumi is code-first and SDK-driven, often favored when teams want software engineering patterns without Terraform HCL or CFN templates.
5. All four can succeed on AWS when teams define review, promotion, drift, and ownership standards.
6. The decision is less about syntax and more about governance, platform maturity, and surrounding tooling.

### Core elements

| Element | Details |
| --- | --- |
| CloudFormation | Declarative AWS-native templates with deep stack lifecycle integration. |
| CDK | Language-based abstraction that synthesizes to CloudFormation. |
| Terraform | Provider ecosystem and multi-cloud focus with plan/apply workflow. |
| Pulumi | SDK-based infrastructure using general-purpose languages and state backends. |
| State model | Stacks, state files, or checkpoints that record managed resources. |
| Diff model | Change sets, cdk diff, terraform plan, or pulumi preview. |
| Governance fit | How each tool integrates with approval, policy, and platform standards. |

### AWS CLI commands

```bash
aws cloudformation validate-template --template-body file://template.yaml
npx cdk diff
terraform plan
pulumi preview
aws cloudformation describe-stacks --stack-name platform-stack
aws resourcegroupstaggingapi get-resources --tag-filters Key=ManagedBy,Values=cloudformation,cdk,terraform,pulumi
aws configservice select-resource-config --expression "SELECT resourceId, resourceType WHERE tags.ManagedBy = 'terraform'"
```

### Quick decision table

```markdown
| Tool | Best Fit | Primary Strength | Typical Trade-off |
| --- | --- | --- | --- |
| CloudFormation | AWS-only platforms | Native lifecycle integration | Lower abstraction |
| CDK | AWS teams that want code reuse | Constructs and languages | Must understand generated CFN |
| Terraform | Multi-cloud organizations | Consistent workflow and ecosystem | Separate state management |
| Pulumi | Engineering teams preferring SDKs | Code-first flexibility | Less AWS-native governance fit |
```

### Best practices

- Choose one default IaC tool for 80 percent of workloads and document exception criteria.
- Standardize tagging, identity, and review controls regardless of tool selection.
- Avoid managing the same resource with multiple IaC systems.
- Teach teams the underlying AWS services so abstraction layers do not become blind spots.
- Adopt policy-as-code or guardrails that work across chosen tools.
- Define import, migration, and decommission procedures before tool sprawl grows.
- Track state security, backup, and recovery requirements explicitly.
- Use generated diffs as mandatory review inputs in CI/CD.

### Operational metrics to watch

- Provisioning lead time by tool.
- Drift or out-of-band change incidents.
- Module or construct reuse rates.
- Review turnaround time for planned changes.
- Tooling support burden across teams.
- State corruption or recovery incidents.

### Common pitfalls

- Allowing every team to pick a different tool increases platform cognitive load.
- Comparing tools only on syntax ignores operations and governance.
- No migration plan means tool changes become risky.
- Overabstraction can hide the real AWS behavior of resources.
- Unclear ownership across tools creates expensive drift.

### Review checklist

- [ ] Default tool and exception policy exist.
- [ ] Tagging and identity standards are cross-tool.
- [ ] State or stack recovery is documented.
- [ ] Diff review is mandatory.
- [ ] Resource ownership is singular.
- [ ] Platform enablement content exists for teams.

## CI/CD Pipeline Architectures

### Why it matters

- Pipeline architecture should match repository layout, organizational boundaries, compliance needs, and deployment targets.
- Monorepo, microservices, multi-account, and container-centric pipelines each optimize for different scaling dimensions.
- Architecture choices affect build fan-out, artifact reuse, secrets management, and observability design.
- A mature platform often supports more than one architecture while enforcing common controls.
- Documented reference architectures reduce accidental complexity and speed onboarding.

### Mermaid diagram

```mermaid
flowchart TB
    Source[Source Models] --> Mono[Monorepo Pipeline]
    Source --> Micro[Microservices Pipelines]
    Source --> Multi[Multi-Account Pipeline]
    Source --> Container[Container Pipeline]
    Mono --> Shared[Shared Build and Selective Deploy]
    Micro --> Service[Independent Service Release]
    Multi --> Promotion[Centralized Promotion Across Accounts]
    Container --> Image[Build Scan Push Deploy]
    classDef awsOrange fill:#FF9900,color:#232F3E,stroke:#232F3E,stroke-width:1px;
    classDef awsDark fill:#232F3E,color:#fff,stroke:#232F3E,stroke-width:1px;
    classDef awsBlue fill:#527FFF,color:#fff,stroke:#232F3E,stroke-width:1px;
    class Source awsOrange;
    class Mono,Micro,Multi,Container awsBlue;
    class Shared,Service,Promotion,Image awsDark;
```

### Explanation

1. Monorepo pipelines typically detect changed paths and fan out targeted builds, tests, and deployments while sharing common tooling.
2. Microservices pipelines prioritize service autonomy, so each service owns its build, test, image, and deploy stages.
3. Multi-account pipelines centralize source and build actions, then promote artifacts into target accounts through assumed roles and regional artifact stores.
4. Container pipelines emphasize image build, scan, registry push, deploy manifest generation, and runtime rollout controls.
5. Each architecture still benefits from strong artifact immutability, consistent identity, and standardized observability.
6. The reference model should define where approvals, security scans, and environment promotions occur.

### Core elements

| Element | Details |
| --- | --- |
| Monorepo | One repository with shared tooling and path-based selective execution. |
| Microservices | Service-per-repo or service-per-pipeline model for team autonomy. |
| Multi-account | Central delivery account promoting artifacts into workload accounts. |
| Container pipeline | Flow focused on image build, scan, push, and orchestrator deploy. |
| Artifact promotion | Reusing the same tested artifact across environments or accounts. |
| Policy gates | Security, compliance, and approval checks inserted at strategic stages. |
| Observability | Unified metrics and traces across pipeline executions and deployments. |

### AWS CLI commands

```bash
aws codepipeline list-pipelines
aws codebuild start-build --project-name monorepo-selective-build
aws sts assume-role --role-arn arn:aws:iam::222222222222:role/CrossAccountDeployRole --role-session-name promote-to-prod
aws ecr describe-images --repository-name service-a
aws ecs update-service --cluster prod --service service-a --force-new-deployment
aws cloudformation deploy --stack-name shared-platform --template-file infra.yaml --capabilities CAPABILITY_NAMED_IAM
aws codepipeline start-pipeline-execution --name multi-account-release
```

### Architecture guidance

```markdown
- Monorepo: use path filters, matrix builds, and shared versioning rules.
- Microservices: keep pipelines independent, but centralize golden templates and security policies.
- Multi-account: centralize build, decentralize runtime permissions, and replicate artifacts regionally.
- Container pipeline: treat image scanning, signing, and digest promotion as first-class stages.
```

### Best practices

- Prefer artifact promotion over rebuilds, regardless of architecture style.
- Standardize identity federation and short-lived credentials across pipeline models.
- In monorepos, use path-aware change detection to avoid unnecessary full builds.
- In microservices, provide golden pipeline templates so autonomy does not equal inconsistency.
- In multi-account designs, isolate build and deploy roles and audit every assume-role path.
- In container pipelines, scan and sign images before promotion to shared runtime repositories.
- Publish common operational dashboards that compare lead time, failure rate, and rollback rate across architectures.
- Document architecture selection criteria so new teams do not reinvent pipeline patterns.

### Operational metrics to watch

- Build fan-out efficiency for monorepos.
- Service deployment frequency for microservices.
- Cross-account promotion latency for multi-account pipelines.
- Image scan pass rate for container pipelines.
- Cost per pipeline execution by architecture type.
- Standard template adoption rate across teams.

### Common pitfalls

- A monorepo without path filters becomes slow and noisy.
- Microservice pipelines can drift wildly without shared guardrails.
- Multi-account designs often fail on IAM or KMS details rather than pipeline logic.
- Container pipelines that deploy mutable tags make incident response harder.
- Too many architecture variants overwhelm platform support teams.

### Review checklist

- [ ] Architecture choice is documented.
- [ ] Artifact promotion model is clear.
- [ ] Cross-account trust is validated where needed.
- [ ] Security scans and approvals are placed intentionally.
- [ ] Operational dashboards exist.
- [ ] Teams consume supported reference templates.

## Strategy Comparison - Rolling vs Blue/Green vs Canary vs All-at-Once

| Strategy | Speed | Risk | Cost Overhead | Best Fit | AWS Implementation Notes |
| --- | --- | --- | --- | --- | --- |
| Rolling | Medium | Medium | Low | Stateful or capacity-limited services | ECS rolling update, Auto Scaling groups, Kubernetes rolling updates |
| Blue/Green | Medium | Low | High | Customer-facing services needing fast rollback | CodeDeploy blue/green, ALB weighted routing, parallel target groups |
| Canary | Slow to medium | Lowest when monitored well | Medium | High-value services with strong telemetry | Lambda alias weights, ALB weights, Argo Rollouts, Flagger |
| All-at-Once | Fastest | Highest | Low | Low-risk internal jobs or reversible workloads | CodeDeploy all-at-once or full ECS replacement |

### Rolling deployment diagram

```mermaid
flowchart LR
    Old[Old Version All Hosts] --> Batch1[Update Batch 1]
    Batch1 --> Batch2[Update Batch 2]
    Batch2 --> Batch3[Update Batch 3]
    Batch3 --> New[New Version All Hosts]
    classDef awsOrange fill:#FF9900,color:#232F3E,stroke:#232F3E,stroke-width:1px;
    classDef awsDark fill:#232F3E,color:#fff,stroke:#232F3E,stroke-width:1px;
    classDef awsBlue fill:#527FFF,color:#fff,stroke:#232F3E,stroke-width:1px;
    class Old awsDark;
    class Batch1,Batch2,Batch3 awsBlue;
    class New awsOrange;
```

### Blue/green deployment diagram

```mermaid
flowchart LR
    Users[Users] --> LB[Load Balancer]
    LB --> Blue[Blue Environment
Current]
    LB -. switch traffic .-> Green[Green Environment
Next Release]
    classDef awsOrange fill:#FF9900,color:#232F3E,stroke:#232F3E,stroke-width:1px;
    classDef awsDark fill:#232F3E,color:#fff,stroke:#232F3E,stroke-width:1px;
    classDef awsBlue fill:#527FFF,color:#fff,stroke:#232F3E,stroke-width:1px;
    class Users awsOrange;
    class LB awsDark;
    class Blue,Green awsBlue;
```

### Canary deployment diagram

```mermaid
flowchart LR
    Users[Users] --> Router[Traffic Router]
    Router --> Stable[Stable Version
90 percent]
    Router --> Canary[Canary Version
10 percent]
    Canary --> Expand[Expand if healthy]
    classDef awsOrange fill:#FF9900,color:#232F3E,stroke:#232F3E,stroke-width:1px;
    classDef awsDark fill:#232F3E,color:#fff,stroke:#232F3E,stroke-width:1px;
    classDef awsBlue fill:#527FFF,color:#fff,stroke:#232F3E,stroke-width:1px;
    class Users,Expand awsOrange;
    class Router awsDark;
    class Stable,Canary awsBlue;
```

### All-at-once deployment diagram

```mermaid
flowchart LR
    Old[Old Version] --> Cutover[Immediate Cutover]
    Cutover --> New[New Version]
    classDef awsOrange fill:#FF9900,color:#232F3E,stroke:#232F3E,stroke-width:1px;
    classDef awsDark fill:#232F3E,color:#fff,stroke:#232F3E,stroke-width:1px;
    classDef awsBlue fill:#527FFF,color:#fff,stroke:#232F3E,stroke-width:1px;
    class Old awsDark;
    class Cutover awsOrange;
    class New awsBlue;
```

## Appendix A - Shared AWS CLI Setup

Use environment variables to keep the examples portable across accounts and regions.

```bash
export AWS_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export PIPELINE_NAME=my-app-pipeline
export BUILD_PROJECT=my-app-build
export DEPLOY_APP=my-app
export ECR_REPO=my-app
aws configure list
aws sts get-caller-identity
```

Recommended shared practices:
- Prefer IAM roles, IAM Identity Center, or OIDC federation instead of static access keys.
- Tag resources with Environment, Application, Owner, CostCenter, and ManagedBy.
- Use named profiles or role chaining carefully in automation.
- Standardize on one default region per workload unless multi-region delivery is required.
- Enable CloudTrail, Config, GuardDuty, and Security Hub in delivery accounts.

## Appendix B - Monorepo Reference Diagram

```mermaid
flowchart LR
    Commit[Commit in Monorepo] --> Detect[Path Change Detection]
    Detect --> BuildA[Build Service A]
    Detect --> BuildB[Build Service B]
    Detect --> Infra[Build Infrastructure]
    BuildA --> DeployA[Deploy Service A]
    BuildB --> DeployB[Deploy Service B]
    Infra --> DeployInfra[Deploy Shared Infra]
    classDef awsOrange fill:#FF9900,color:#232F3E,stroke:#232F3E,stroke-width:1px;
    classDef awsDark fill:#232F3E,color:#fff,stroke:#232F3E,stroke-width:1px;
    classDef awsBlue fill:#527FFF,color:#fff,stroke:#232F3E,stroke-width:1px;
    class Commit,Detect awsOrange;
    class BuildA,BuildB,Infra,DeployA,DeployB,DeployInfra awsBlue;
```

Use path filters, matrix builds, and dependency graphs so only impacted services build and deploy.

## Appendix C - Microservices Reference Diagram

```mermaid
flowchart LR
    RepoA[Service A Repo] --> PipeA[Pipeline A]
    RepoB[Service B Repo] --> PipeB[Pipeline B]
    RepoC[Service C Repo] --> PipeC[Pipeline C]
    PipeA --> RuntimeA[ECS or EKS A]
    PipeB --> RuntimeB[ECS or EKS B]
    PipeC --> RuntimeC[Lambda or ECS C]
    classDef awsOrange fill:#FF9900,color:#232F3E,stroke:#232F3E,stroke-width:1px;
    classDef awsDark fill:#232F3E,color:#fff,stroke:#232F3E,stroke-width:1px;
    classDef awsBlue fill:#527FFF,color:#fff,stroke:#232F3E,stroke-width:1px;
    class RepoA,RepoB,RepoC awsOrange;
    class PipeA,PipeB,PipeC,RuntimeA,RuntimeB,RuntimeC awsBlue;
```

Use golden templates for IAM, logging, image scanning, and deployment approvals to keep service autonomy manageable.

## Appendix D - Multi-Account Promotion Checklist

- Central pipeline account owns source, build, and artifact packaging.
- Target accounts own runtime resources and expose narrowly scoped deploy roles.
- Artifact buckets are encrypted and replicated to required regions.
- KMS grants allow only the roles that must decrypt artifacts.
- STS session tags or external IDs are used where appropriate.
- CloudTrail records all assume-role and deployment events.
- Promotion gates are based on test evidence, security findings, and approvals.
- Incident rollback paths are validated in each target account.

## Appendix E - Container Pipeline Reference Diagram

```mermaid
flowchart LR
    Source[Source Commit] --> Build[Build Container]
    Build --> Scan[Scan Image]
    Scan --> Sign[Sign Image]
    Sign --> Push[Push to ECR]
    Push --> Promote[Promote Digest]
    Promote --> Deploy[Deploy to ECS or EKS]
    classDef awsOrange fill:#FF9900,color:#232F3E,stroke:#232F3E,stroke-width:1px;
    classDef awsDark fill:#232F3E,color:#fff,stroke:#232F3E,stroke-width:1px;
    classDef awsBlue fill:#527FFF,color:#fff,stroke:#232F3E,stroke-width:1px;
    class Source,Build awsOrange;
    class Scan,Push,Deploy awsBlue;
    class Sign,Promote awsDark;
```

Promote digests, not mutable tags, so every environment receives the identical tested image.

## Appendix F - Operational Best Practices

- Build once, promote many: avoid environment-specific rebuilds.
- Use short-lived credentials and OIDC federation for CI where supported.
- Encrypt artifacts, logs, and state stores with customer-managed KMS keys when required.
- Standardize observability for pipeline, build, deployment, and runtime signals.
- Record commit SHA, image digest, and stack revision in release metadata.
- Automate rollback where metrics can reliably detect harm.
- Treat manual approvals as risk controls, not default process.
- Keep platform documentation updated with diagrams, commands, and ownership information.

## Appendix G - Troubleshooting Commands

```bash
aws codepipeline get-pipeline-state --name $PIPELINE_NAME
aws codebuild batch-get-builds --ids $BUILD_ID
aws deploy get-deployment --deployment-id $DEPLOYMENT_ID
aws cloudformation describe-stack-events --stack-name $STACK_NAME
aws ecr describe-image-scan-findings --repository-name $ECR_REPO --image-id imageTag=$IMAGE_TAG
aws logs tail /aws/codebuild/$BUILD_PROJECT --follow
aws logs tail /aws/lambda/$FUNCTION_NAME --follow
kubectl get pods -A
kubectl describe deployment my-app -n production
```

Focus first on artifact identity, permissions, and environment-specific configuration differences.

## Appendix H - Governance Summary

| Control Area | Recommended Baseline |
| --- | --- |
| Identity | OIDC or role-based auth for CI/CD systems |
| Encryption | KMS for artifact stores, state stores, and sensitive logs |
| Audit | CloudTrail, Config, and deployment event retention |
| Review | Pull request review plus diff or change-set inspection |
| Promotion | Immutable artifacts with environment gates |
| Rollback | Tested, automated, and observable rollback path |
| Drift | Scheduled detection for infra and cluster config |
| Ownership | Tags and documented service boundaries |
