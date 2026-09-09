# GenAI Agents on AWS

A hands-on workshop for building, deploying, and operating customer-support agents on Amazon EKS. The repository contains two parallel learning tracks:

- **Self-managed GenAI**: open-source model serving and data services running in the cluster.
- **Integrated GenAI**: Amazon Bedrock and Amazon Bedrock AgentCore managed capabilities connected to the same agent patterns.

The agents expose a small HTTP API and stream responses as Server-Sent Events (SSE). A shared Chainlit chat UI routes requests to the currently deployed lab services.

## What this repository includes

- Terraform infrastructure for EKS Auto Mode, networking, IAM, ECR, ALB, storage, observability, model serving, and managed agent services.
- Containerized agent labs under `modules/`.
- A shared Chainlit UI with profiles for single-agent and multi-agent deployments.
- Optional CodeBuild image prebuilds so a fresh environment does not require a local Docker daemon.

## Architecture

```text
Browser
  |
  v
ALB Ingress -> Chainlit UI (default namespace)
                    |
                    +--> customer-agent (default namespace)
                    +--> orchestrator-agent (default namespace)
                    |       +--> order-agent
                    |       +--> product-agent
                    |
                    +--> customer-agent (agents namespace)
                    +--> orchestrator-agent (agents namespace)

Shared platform services:
  LiteLLM | vLLM | Langfuse | Milvus | Neo4j | AgentCore Memory
```

Both tracks use the OpenAI-compatible LiteLLM endpoint from the agent code. LiteLLM routes requests to a self-managed vLLM model or to Bedrock depending on the model alias. The integrated track also provisions EKS Pod Identity for AgentCore Memory and managed tool access.

## Repository layout

```text
terraform/                 AWS and Kubernetes infrastructure
  *.tf                     Terraform resources and outputs
  scripts/                 Provisioning helper scripts
modules/
  20-self-managed/         Self-managed GenAI labs
  30-integrated/            Bedrock and AgentCore labs
  ui/                      Shared Chainlit chat UI
```

Each lab directory normally contains an agent implementation, `requirements.txt`, a `Dockerfile`, and one or more Kubernetes manifests. The manifests use `${ACCOUNT_ID}` and `${AWS_REGION}` placeholders; replace them before applying a manifest manually.

## Lab catalog

### Self-managed track

| Module | Focus | Image tag(s) |
| --- | --- | --- |
| `200-strands-agents` | Strands single agent with tools | `customer-agent:strands` |
| `300-observability-langfuse` | Agent tracing with Langfuse | `customer-agent:langfuse` |
| `400-rag-milvus` | Retrieval-augmented generation with Milvus | `customer-agent:milvus` |
| `500-memory-milvus` | Conversation memory backed by Milvus | `customer-agent:milvus-memory` |
| `600-agent-tools-mcp` | MCP server and MCP-connected agent | `customer-agent:mcp`, `mcp-server:v1` |
| `700-multi-agent-a2a` | A2A orchestrator with order and product specialists | `orchestrator-agent:v1`, `order-agent:v1`, `product-agent:v1` |
| `800-knowledge-graph` | Knowledge graph agent with Neo4j | `customer-agent:graph` |

### Integrated track

| Module | Focus | Image tag(s) |
| --- | --- | --- |
| `100-strands-bedrock` | Strands agent using Bedrock through LiteLLM | `customer-agent:bedrock` |
| `200-observability-langfuse` | Bedrock agent tracing with Langfuse | `customer-agent:bedrock-langfuse` |
| `300-memory-agentcore` | AgentCore Memory sessions and actors | `customer-agent:agentcore-memory` |
| `400-managed-tools` | AgentCore managed browser/code-interpreter tools | `customer-agent:agentcore-tools` |
| `500-multi-agent-a2a` | A2A orchestrator with Bedrock and sandbox specialists | `orchestrator-agent:bedrock`, `order-agent:bedrock`, `product-agent:bedrock` |
| `550-evaluation-agentcore` | AgentCore observability and evaluation workflow | `customer-agent:agentcore-eval` |

## Prerequisites

You need:

- An AWS account with permission to create EKS, EC2/VPC, IAM, ECR, S3, CodeBuild, ALB, Bedrock, and AgentCore resources.
- AWS CLI authenticated to the target account and region.
- Terraform `>= 1.3`.
- `kubectl` configured locally after the cluster is created.
- Access to the required Bedrock models in the selected region for the integrated labs. Anthropic models may require an AWS Marketplace subscription on first use.
- A browser for the public ALB endpoints. Docker is optional when `prebuild_images = true` (the default).

The deployment creates billable AWS resources, including EKS, load balancers, NAT gateways, storage, CodeBuild, and model-serving workloads. Use a dedicated workshop account where possible.

## Quick start

Run Terraform from the `terraform/` directory:

```bash
cd terraform
terraform init
terraform apply \
  -var='aws_region=us-west-2' \
  -var='eks_cluster_id=genai-workshop' \
  -var='allowed_ingress_cidrs=["YOUR_PUBLIC_IP/32"]'
```

Use `0.0.0.0/0` only when intentionally opening the public ALB and Neo4j endpoints to everyone. A narrower CIDR such as your current public IP is recommended.

The default apply also:

1. Creates the EKS cluster and supporting namespaces.
2. Creates ECR repositories for the workshop images.
3. Archives `../modules/`, uploads the archive to a private S3 bucket, and starts a CodeBuild job for all module images.
4. Deploys shared services including LiteLLM, Langfuse, Milvus, Neo4j, and the Chainlit UI.
5. Creates AgentCore Memory and integrated-track IAM wiring.

The initial apply can take several minutes while EKS, Auto Mode capacity, Helm releases, and CodeBuild become ready.

## Connect to the cluster

Terraform prints the kubeconfig command as an output. Run it after apply:

```bash
terraform output -raw configure_kubectl
# Run the printed command, then verify access:
kubectl get nodes
kubectl get pods -A
```

Get the Chainlit URL from the ALB ingress:

```bash
kubectl get ingress chainlit-ui \
  -o jsonpath='http://{.status.loadBalancer.ingress[0].hostname}'
```

Open the returned address in a browser and choose a profile. A profile is available when its corresponding lab has been deployed; otherwise the UI reports that the agent cannot be reached.

Other useful endpoints:

```bash
# LiteLLM admin UI / proxy hostname
kubectl get ingress -n litellm litellm \
  -o jsonpath='http://{.status.loadBalancer.ingress[0].hostname}'

# Watch shared workloads
kubectl get pods -A -w
```

## Deploy a lab manually

Terraform prebuilds the images and creates the ECR repositories. To deploy a single-agent self-managed lab, substitute the placeholders in its manifest and apply it in the default namespace:

```bash
cd modules/20-self-managed/200-strands-agents/customer-agent
sed -i "s/\\${ACCOUNT_ID}/$(aws sts get-caller-identity --query Account --output text)/g; s/\\${AWS_REGION}/us-west-2/g" k8s.yaml
kubectl apply -f k8s.yaml
kubectl rollout status deployment/customer-agent
```

For an integrated lab, use the `agents` namespace and its service account. For example:

```bash
cd modules/30-integrated/300-memory-agentcore/customer-agent
sed -i "s/\\${ACCOUNT_ID}/$(aws sts get-caller-identity --query Account --output text)/g; s/\\${AWS_REGION}/us-west-2/g" k8s.yaml
kubectl apply -f k8s.yaml
kubectl -n agents rollout status deployment/customer-agent
```

Multi-agent labs require all of their manifests: specialists first, then the orchestrator. The self-managed A2A manifests can be applied with:

```bash
cd modules/20-self-managed/700-multi-agent-a2a/a2a-agents
kubectl apply -f k8s-specialists.yaml -f k8s-orchestrator.yaml
kubectl rollout status deployment/order-agent
kubectl rollout status deployment/product-agent
kubectl rollout status deployment/orchestrator-agent
```

Because several single-agent labs intentionally reuse the `customer-agent` Service, deploying a new single-agent manifest replaces the image used by that Service. This is expected: the UI profile describes the lab family, while the active image is whichever variant was deployed last in that namespace.

## Build and push an image yourself

The CodeBuild prebuild is convenient for a clean environment, but each lab can also be changed and pushed manually. Authenticate Docker to ECR, then build from the lab directory:

```bash
export AWS_REGION=us-west-2
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export ECR_REGISTRY=${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

aws ecr get-login-password --region "$AWS_REGION" \
  | docker login --username AWS --password-stdin "$ECR_REGISTRY"

docker build -t "$ECR_REGISTRY/customer-agent:custom" .
docker push "$ECR_REGISTRY/customer-agent:custom"
```

Update the image tag in the Kubernetes manifest and reapply it. For multi-agent images, use the corresponding Dockerfiles (`Dockerfile.order`, `Dockerfile.product`, `Dockerfile.orchestrator`, or the integrated `Dockerfile.sandbox`).

To skip the Terraform-managed CodeBuild prebuild:

```bash
terraform apply \
  -var='aws_region=us-west-2' \
  -var='eks_cluster_id=genai-workshop' \
  -var='allowed_ingress_cidrs=["YOUR_PUBLIC_IP/32"]' \
  -var='prebuild_images=false'
```

## Agent API

Every customer agent provides:

- `GET /healthz` for a basic health check.
- `POST /chat` with a JSON body containing `query`, and optional `session_id` and `actor_id`.
- `text/event-stream` responses containing JSON `token` chunks and a final `[DONE]` event.

Example:

```bash
kubectl port-forward service/customer-agent 8080:8080
curl -N http://127.0.0.1:8080/chat \
  -H 'content-type: application/json' \
  -d '{"query":"Where is order 1234?","session_id":"local-test"}'
```

For integrated agents, port-forward the service in the `agents` namespace:

```bash
kubectl -n agents port-forward service/customer-agent 8080:8080
```

## Troubleshooting

**The UI says it cannot reach an agent**

Check that the expected Service exists and that its pods are ready:

```bash
kubectl get svc,pods
kubectl -n agents get svc,pods
kubectl describe pod -l app=customer-agent
```

**Pods are stuck in `ImagePullBackOff`**

Confirm the image tag exists in ECR and that the manifest uses the correct AWS account and region:

```bash
aws ecr describe-images --repository-name customer-agent --region "$AWS_REGION"
kubectl describe pod -l app=customer-agent
```

**The ALB has no hostname yet**

The EKS Auto Mode load balancer may still be provisioning. Check ingress events:

```bash
kubectl describe ingress chainlit-ui
```

**An integrated request returns a Bedrock authorization error**

Verify the selected region supports the requested model, the account has model access or Marketplace subscription where required, and that the LiteLLM pod is healthy:

```bash
kubectl -n litellm get pods
kubectl -n litellm logs deploy/litellm --tail=100
```

## Cleanup

Destroy the workshop when finished to avoid ongoing charges:

```bash
cd terraform
terraform destroy \
  -var='aws_region=us-west-2' \
  -var='eks_cluster_id=genai-workshop' \
  -var='allowed_ingress_cidrs=["YOUR_PUBLIC_IP/32"]'
```

Review the Terraform plan carefully before confirming destruction, especially if the AWS account contains resources outside this workshop.

## Contributing

Keep module source self-contained in its lab directory. When adding or renaming an image, update the image matrix in `terraform/codebuild-images.tf`, the relevant Kubernetes manifest, and the UI registry in `modules/ui/agents_registry.py` when the profile or service routing changes.
