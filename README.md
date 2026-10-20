# NovaMart Multi-Agent RAG

NovaMart is a multi-agent customer-support system built with Strands Agents,
Amazon Bedrock, Amazon Bedrock AgentCore, DynamoDB, S3, S3 Vectors,
CloudWatch, and AWS X-Ray.

## Architecture

The system uses five cooperating agents:

- **OrchestratorAgent** routes each request and manages shared workflow state.
- **InventoryAgent** reads customer and order data from DynamoDB.
- **PolicyAgent** coordinates three parallel knowledge-base retrievers for
  returns, shipping, and warranty policies.
- **RefundAgent** evaluates refund eligibility from order facts and policy data.
- **CommunicationAgent** produces the final customer-facing response.

Policy retrieval runs concurrently with
`ThreadPoolExecutor(max_workers=3)` and `as_completed()`. The three retrievers
use `temperature=0.0`; the PolicyAgent coordinator uses `temperature=0.2`.

Workflow state is stored in DynamoDB with optimistic locking. The deployed
runtime uses Bedrock Guardrails, AgentCore Memory, CloudWatch logging, and
X-Ray distributed tracing.

## Repository layout

```text
agentcore/       AgentCore Runtime and CDK configuration
infrastructure/  CloudFormation, data seeding, and cleanup utilities
src/             Agents, orchestration, retrieval, deployment, and tracing
config.py        CloudFormation and environment configuration loader
```

## Requirements

- Python 3.12
- Node.js 20 or newer
- AWS credentials with access to Bedrock, Bedrock AgentCore, DynamoDB, S3,
  S3 Vectors, CloudFormation, IAM, CloudWatch, and X-Ray
- AgentCore CLI 0.30.0 or newer

```bash
npm install -g @aws/agentcore@0.30.0
pip install -r requirements.txt
```

## Infrastructure

Deploy the foundation stack:

```bash
aws cloudformation deploy \
  --template-file infrastructure/stack.yaml \
  --stack-name novamart-agentcore \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

Seed DynamoDB and the policy-document bucket:

```bash
python infrastructure/seed_data.py
```

Create three Bedrock Knowledge Bases using Titan Text Embeddings v2 and the
S3 Vectors indexes created by the stack:

| Knowledge base | S3 prefix | Vector index |
|---|---|---|
| Returns | `policies/returns/` | `returns-policy-index` |
| Shipping | `policies/shipping/` | `shipping-policy-index` |
| Warranty | `policies/warranty/` | `warranty-policy-index` |

## Configuration

```bash
cp .env.example .env
```

Set the three Knowledge Base IDs in `.env`. Runtime and guardrail values are
printed by the deployment command and can then be added to the same file.

```bash
python config.py
agentcore validate
```

## Deployment

```bash
python src/agent_orchestrator.py deploy
```

The deployment creates or updates the AgentCore Runtime, Bedrock Guardrail,
AgentCore Memory, CloudWatch logging configuration, and X-Ray tracing.

## Usage

Invoke the deployed runtime:

```bash
python src/agent_orchestrator.py invoke "What is the return policy for premium customers?"
```

Run the local end-to-end scenarios:

```bash
python src/agent_orchestrator.py test
```

Start an interactive session:

```bash
python src/agent_orchestrator.py chat
```

## Cleanup

```bash
python infrastructure/cleanup.py
python infrastructure/cleanup.py --yes
```
