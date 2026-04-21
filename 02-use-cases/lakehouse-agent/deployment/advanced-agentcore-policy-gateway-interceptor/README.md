# Advanced AgentCore Policy + Lambda Interceptor (CDK)

This CDK project extends the lakehouse-agent sample with a layered security
architecture that combines **Cedar-based AgentCore Policy** and a **Design 3
Request Interceptor Lambda**. It implements the three patterns described in
the blog post *"Build Secure AI Agent Behavior with Policy and Lambda
Interceptors in Amazon Bedrock AgentCore"*:

| Design | Mechanism | Demo rule |
|---|---|---|
| **Design 1 — Policy Only** | Cedar `forbid` rule on the Gateway | Policyholders cannot invoke `get_claims_summary` |
| **Design 2 — Interceptor Only** | Request Interceptor performs `sts:AssumeRole` to scope credentials, so Lake Formation applies row- and column-level security | Each user sees only their own rows and permitted columns |
| **Design 3 — Policy + Interceptor** | Interceptor injects `geography`, Cedar evaluates `context.input.geography` | EU users cannot invoke `query_claims` or `get_claim_details` |

## Prerequisites

This CDK sample runs **after the base lakehouse-agent is deployed** (Steps 1–7
in [deployment/README.md](../README.md)). It reads every input — Gateway ARN,
interceptor Lambda ARNs, Cognito IDs, etc. — from the SSM parameters produced
by those steps.

Required tooling:

- AWS credentials with permissions to create AgentCore Policy Engine and
  Cedar policies (`bedrock-agentcore:*`), update the Gateway, and attach IAM
  inline policies to the Gateway role.
- AWS CLI v2 configured for the same account and region as the base deployment.
- Node.js 18+ and npm.
- Python 3.10+ (for the pre-deploy and verification scripts) with the same
  virtual environment used for Phase 1.

> **Region note**: All commands below assume `AWS_REGION=us-east-1`, matching
> the base lakehouse-agent deployment. Export `AWS_REGION` before running any
> step if your shell default differs.

## Directory layout

```
advanced-agentcore-policy-gateway-interceptor/
├── README.md                       # (this file)
├── package.json / tsconfig.json    # CDK TypeScript project
├── cdk.json.example                # Template — cdk.json is generated at deploy-time
├── bin/app.ts                      # CDK entry point (reads account/region from context)
├── lib/policy-stack.ts             # PolicyStack: Policy Engine + Cedar policies + Gateway re-attach
├── policies/                       # Cedar source (one file per policy)
├── lambda/interceptor-request/     # Design 3 Request Interceptor Lambda source
├── scripts/
│   ├── pre-deploy.sh               # Runs the 3 steps below in one go
│   ├── generate-cdk-context.sh     # Generates cdk.json from SSM
│   └── detach-interceptors.py      # Detaches Interceptors before Cedar policy creation
└── verification/
    └── verify_policy.py            # 13-check FGAC regression suite
```

## Deploy

### Step 1 — Pre-deploy

`pre-deploy.sh` does three things in sequence:

1. **Generate `cdk.json`** from SSM Parameter Store (account ID is derived
   from `aws sts get-caller-identity`).
2. **Detach the Interceptors** from the Gateway. Cedar policy creation sends
   internal MCP validation requests that are SigV4-signed (not Bearer-token
   authenticated), which fail against a JWT-validating Interceptor. CDK
   re-attaches both Interceptors together with the Policy Engine in Step 2.
3. **Overwrite the base `interceptor-request/lambda_function.py`** with the
   Design 3 version (adds user geography injection) and redeploy that Lambda.

```bash
cd 02-use-cases/lakehouse-agent/deployment/advanced-agentcore-policy-gateway-interceptor
AWS_REGION=us-east-1 bash scripts/pre-deploy.sh
```

### Step 2 — CDK deploy

```bash
npm ci
# Bootstrap once per account/region if you have not deployed any CDK stack yet:
# npx cdk bootstrap
npx cdk deploy --require-approval never
```

This creates:

- **`CfnPolicyEngine`** — the AgentCore Policy Engine.
- **`CfnPolicy` x 4** — Cedar policies from `policies/*.cedar`. `permit_all`
  is created first (with `IGNORE_ALL_FINDINGS` to bypass the Overly Permissive
  warning), then the three `forbid` policies in parallel.
- **IAM inline policy** on the existing Gateway role for
  `bedrock-agentcore:AuthorizeAction` etc.
- **`AwsCustomResource` → `UpdateGateway`** — re-attaches both Interceptors
  and attaches the Policy Engine in `ENFORCE` mode in a single API call.

Deployment takes about 2 minutes.

### Step 3 — Verify the Policy Engine is active

```bash
AWS_REGION=us-east-1 python3 -c "
import boto3
client = boto3.Session(region_name='us-east-1').client('bedrock-agentcore-control')
for e in client.list_policy_engines().get('policyEngines', []):
    if 'Lakehouse' in e['name']:
        print(f'Engine: {e[\"policyEngineId\"]} ({e[\"status\"]})')
        for p in client.list_policies(policyEngineId=e['policyEngineId']).get('policies', []):
            print(f'  {p[\"name\"]}: {p[\"status\"]}')
"
```

All four policies should report `ACTIVE`.

### Step 4 — Run the end-to-end verification

```bash
cd ../../..                                  # back to lakehouse-agent/
source .venv/bin/activate                    # same venv used for Phase 1
python deployment/advanced-agentcore-policy-gateway-interceptor/verification/verify_policy.py
```

Expected output:

```
Results: 13/13 passed
```

## What the policies enforce

| Policy file | Pattern | Effect |
|---|---|---|
| `permit_all.cedar` | Baseline permit | Without this, AgentCore defaults to deny-by-default once a Policy Engine is attached |
| `forbid_policyholder_summary.cedar` | Design 1 | Blocks `get_claims_summary` when `principal.getTag("cognito:groups") like "*policyholders*"` |
| `forbid_eu_individual_claims.cedar` | Design 3 | Blocks `query_claims` and `get_claim_details` when `context.input.geography == "EU"` |
| `forbid_restricted_geography.cedar` | Design 3 | Blocks every tool when `context.input.geography == "RESTRICTED"` |

The `geography` attribute is injected by the Design 3 Request Interceptor at
`params.arguments.geography` (top level). Cedar maps that to
`context.input.geography`. The demo Lambda ships a hard-coded mapping in
`USER_GEOGRAPHY` — replace with a DynamoDB lookup for production.

## Cleanup

Destroy **in reverse order**. Phase 2 first, then Phase 1.

### Phase 2 — This CDK stack

```bash
cd 02-use-cases/lakehouse-agent/deployment/advanced-agentcore-policy-gateway-interceptor
npx cdk destroy --force
```

`cdk destroy` does the following via the `AwsCustomResource`
and the `CfnPolicy` / `CfnPolicyEngine` resource lifecycles:

1. Detaches the Policy Engine from the Gateway (Interceptors remain attached).
2. Deletes the four Cedar policies.
3. Deletes the Policy Engine.
4. Removes the inline IAM policy from the Gateway role.

> **Note:** The Design 3 Request Interceptor Lambda source (with geography
> injection) remains deployed after `cdk destroy`. That is intentional — the
> Lambda is a Phase 1 resource and is cleaned up in the Phase 1 cleanup below.
> If you want to roll back to the original Phase 1 Lambda (without geography
> injection) before destroying Phase 1, restore
> `deployment/5-gateway-setup/interceptor-request/lambda_function.py` from git
> and redeploy:
>
> ```bash
> git checkout -- deployment/5-gateway-setup/interceptor-request/lambda_function.py
> cd deployment/5-gateway-setup/interceptor-request
> AWS_REGION=us-east-1 ./deploy.sh
> ```

### Phase 1 — Base lakehouse-agent

Follow the standard cleanup in the parent guide — each Phase 1 step has a
dedicated cleanup script, run in reverse order:

```bash
cd 02-use-cases/lakehouse-agent/deployment
cd 6-lakehouse-agent              && python cleanup_agent.py
cd ../5-gateway-setup             && python cleanup_gateway.py
cd ../4-mcp-lakehouse-server      && python cleanup_runtime.py
cd ../3-s3tables-setup            && python cleanup_s3tables.py
cd ../2-lakehouse-tenant-roles-setup && python cleanup_iam_roles.py
cd ../1-cognito-setup             && python cleanup_cognito.py
```

See [../README.md](../README.md) for details.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `CfnPolicy` → `CREATE_FAILED` with `InterceptorException` | The Gateway still had the JWT-validating Interceptor attached while Cedar tried its internal MCP validation | Re-run `scripts/pre-deploy.sh` (it detaches Interceptors) then `cdk deploy` again |
| All tool calls return 500 | After detach, the Response Interceptor Lambda was missing when CDK re-attached | Deploy the Response Interceptor first: `deployment/5-gateway-setup/interceptor-response/deploy.sh` |
| `permit_all` fails with "Overly Permissive" | `validationMode: FAIL_ON_ANY_FINDINGS` on a broad permit | PolicyStack already uses `IGNORE_ALL_FINDINGS` for `permit_all` — rerun `cdk deploy` |
| `context.input` returns `attribute not found` | Cedar rule used a wildcard `action` | List tools explicitly in `action in [...]` (see `forbid_eu_individual_claims.cedar`) |
| Every tool returns DENY after deploy | `permit_all` is not `ACTIVE` | Re-check `list_policies` status; if not ACTIVE, re-run `cdk deploy` |

## References

- Blog post: *Build Secure AI Agent Behavior with Policy and Lambda Interceptors in Amazon Bedrock AgentCore*
- [Phase 1 deployment guide](../README.md)
- [lakehouse-agent README](../../README.md)
