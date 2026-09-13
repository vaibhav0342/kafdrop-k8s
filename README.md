# Kafdrop on Kubernetes

Production-oriented Kubernetes deployment of [Kafdrop](https://github.com/obsidiandynamics/kafdrop) for monitoring and browsing an AWS MSK Kafka cluster.

## Overview

Kafdrop is a lightweight web UI for Kafka. It provides visibility into:

* Kafka brokers
* Topics
* Partitions
* Consumer groups
* Consumer lag
* Messages
* Topic configuration

This deployment is designed for running Kafdrop on Amazon EKS and connecting securely to Amazon MSK using AWS IAM authentication.

## Architecture

```text
                         AWS Cloud
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  Amazon EKS                                              │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │                  kafka-tools namespace              │  │
│  │                                                    │  │
│  │   ┌────────────────────────────────────────────┐   │  │
│  │   │              Kafdrop Pods                  │   │  │
│  │   │                                            │   │  │
│  │   │  Kafdrop :9000                            │   │  │
│  │   │                                            │   │  │
│  │   │  SASL_SSL                                  │   │  │
│  │   │  AWS_MSK_IAM                               │   │  │
│  │   └──────────────────────┬─────────────────────┘   │  │
│  │                          │                         │  │
│  │                          │ IAM                     │  │
│  │                          ▼                         │  │
│  │                  ServiceAccount                    │  │
│  │                          │                         │  │
│  └──────────────────────────┼─────────────────────────┘  │
│                             │                            │
│                             ▼                            │
│                       AWS IAM Role                       │
│                             │                            │
│                             │                            │
│                             ▼                            │
│                    Amazon MSK Cluster                     │
│                                                          │
│                  ┌────────┬────────┬────────┐            │
│                  │ Broker │ Broker │ Broker │            │
│                  │   1    │   2    │   3    │            │
│                  └────────┴────────┴────────┘            │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

## Repository Structure

Example structure:

```text
kafdrop-k8s/
│
├── 01-namespace.yaml
├── 02-service-account.yaml
├── 03-deployment.yaml
├── 04-service.yaml
├── 05-pdb.yaml
├── 06-ingress.yaml
├── 07-network-policy.yaml
│
└── README.md
```

## Prerequisites

Before deploying Kafdrop, make sure the following are available:

* AWS account
* EKS cluster
* Amazon MSK cluster
* `kubectl`
* AWS CLI
* IAM permissions
* AWS Load Balancer Controller, if using ALB Ingress
* MSK IAM authentication enabled
* Network connectivity between EKS and MSK

Check Kubernetes access:

```bash
kubectl get nodes
```

Check AWS identity:

```bash
aws sts get-caller-identity
```

Set the AWS region:

```bash
export AWS_REGION=ap-south-1
```

---

# 1. Get the MSK Cluster

List MSK clusters:

```bash
aws kafka list-clusters-v2 \
  --region "$AWS_REGION" \
  --query 'ClusterInfoList[].{Name:ClusterName,Arn:ClusterArn,State:State}' \
  --output table
```

Set the cluster ARN:

```bash
export MSK_CLUSTER_ARN="YOUR_MSK_CLUSTER_ARN"
```

---

# 2. Get MSK IAM Bootstrap Brokers

Kafdrop needs the MSK bootstrap broker addresses.

Run:

```bash
aws kafka get-bootstrap-brokers \
  --region "$AWS_REGION" \
  --cluster-arn "$MSK_CLUSTER_ARN" \
  --query 'BootstrapBrokerStringSaslIam' \
  --output text
```

Example:

```text
b-1.kafkaprod.xxxxx.ap-south-1.amazonaws.com:9098,
b-2.kafkaprod.xxxxx.ap-south-1.amazonaws.com:9098,
b-3.kafkaprod.xxxxx.ap-south-1.amazonaws.com:9098
```

Port `9098` is used for MSK IAM authentication.

---

# 3. Verify MSK IAM Authentication

Check whether IAM authentication is enabled:

```bash
aws kafka describe-cluster-v2 \
  --region "$AWS_REGION" \
  --cluster-arn "$MSK_CLUSTER_ARN" \
  --query 'ClusterInfo.Provisioned.ClientAuthentication.Sasl.Iam.Enabled' \
  --output text
```

Expected:

```text
True
```

If the result is `False`, configure MSK IAM authentication before continuing.

---

# 4. Create the Namespace

Apply:

```bash
kubectl apply -f 01-namespace.yaml
```

Check:

```bash
kubectl get namespace kafka-tools
```

---

# 5. Create the IAM Policy

Kafdrop should normally be given read-only Kafka permissions.

Example policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "KafdropMSKReadOnly",
      "Effect": "Allow",
      "Action": [
        "kafka-cluster:Connect",
        "kafka-cluster:DescribeCluster",
        "kafka-cluster:DescribeClusterDynamicConfiguration",
        "kafka-cluster:DescribeTopic",
        "kafka-cluster:DescribeGroup",
        "kafka-cluster:ReadData"
      ],
      "Resource": "*"
    }
  ]
}
```

Save it as:

```text
kafdrop-msk-readonly-policy.json
```

Create the policy:

```bash
aws iam create-policy \
  --policy-name kafdrop-msk-readonly \
  --policy-document file://kafdrop-msk-readonly-policy.json
```

For production, replace `"Resource": "*"` with the required MSK cluster, topic, and group resources after validating the permissions needed by your Kafdrop version.

---

# 6. Create the IAM Role

The EKS ServiceAccount needs an IAM role.

Example role:

```text
kafdrop-msk-readonly
```

The role trust policy should allow the Kubernetes ServiceAccount:

```text
system:serviceaccount:kafka-tools:kafdrop
```

to assume the role.

The exact trust configuration depends on whether your EKS cluster uses:

* EKS Pod Identity
* IAM Roles for Service Accounts (IRSA)

Do not use both mechanisms for the same configuration unless you intentionally understand the interaction.

---

# 7. Kubernetes ServiceAccount

Example:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kafdrop
  namespace: kafka-tools
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::YOUR_AWS_ACCOUNT_ID:role/kafdrop-msk-readonly
```

Apply:

```bash
kubectl apply -f 02-service-account.yaml
```

Check:

```bash
kubectl get serviceaccount kafdrop -n kafka-tools
```

---

# 8. Kafdrop Deployment

The Kafdrop container uses the following configuration:

```yaml
env:

  - name: KAFKA_BROKERCONNECT
    value: "b-1.kafkaprod.xxxxx.amazonaws.com:9098,b-2.kafkaprod.xxxxx.amazonaws.com:9098,b-3.kafkaprod.xxxxx.amazonaws.com:9098"

  - name: KAFKA_PROPERTIES
    value: |
      security.protocol=SASL_SSL
      sasl.mechanism=AWS_MSK_IAM
      sasl.jaas.config=software.amazon.msk.auth.iam.IAMLoginModule required;
      sasl.client.callback.handler.class=software.amazon.msk.auth.iam.IAMClientCallbackHandler

  - name: CMD_ARGS
    value: >-
      --topic.deleteEnabled=false
      --topic.createEnabled=false
      --message.sendEnabled=false
```

### KAFKA_BROKERCONNECT

Defines the MSK bootstrap brokers.

```text
Kafdrop
   |
   +--> Broker 1 :9098
   +--> Broker 2 :9098
   +--> Broker 3 :9098
```

### KAFKA_PROPERTIES

Defines the Kafka security configuration.

```text
security.protocol=SASL_SSL
```

Enables encrypted communication and SASL authentication.

```text
sasl.mechanism=AWS_MSK_IAM
```

Uses AWS IAM authentication.

```text
sasl.jaas.config=software.amazon.msk.auth.iam.IAMLoginModule required;
```

Uses the AWS MSK IAM login module.

```text
sasl.client.callback.handler.class=software.amazon.msk.auth.iam.IAMClientCallbackHandler
```

Uses the AWS MSK IAM callback handler.

Kafdrop supports SASL/TLS Kafka connections through its Kafka properties configuration.

---

# 9. Production Safety Settings

The following options are intentionally disabled:

```text
--topic.deleteEnabled=false
--topic.createEnabled=false
--message.sendEnabled=false
```

This gives Kafdrop a monitoring-oriented role.

```text
                    Kafdrop
                       |
       ┌───────────────┼────────────────┐
       │               │                │
       ▼               ▼                ▼
    View Topics    View Messages    View Groups
       ✓               ✓                ✓

       Create Topic       ✗
       Delete Topic       ✗
       Send Message      ✗
```

Kafdrop supports disabling topic creation and deletion through `CMD_ARGS`; the official documentation also notes that message sending is disabled by default and can be explicitly controlled.

---

# 10. Apply the Deployment

First validate the YAML:

```bash
kubectl apply --dry-run=client -f 03-deployment.yaml
```

Apply:

```bash
kubectl apply -f 03-deployment.yaml
```

Check:

```bash
kubectl get deployment -n kafka-tools
```

Check Pods:

```bash
kubectl get pods -n kafka-tools -o wide
```

Check logs:

```bash
kubectl logs -n kafka-tools deployment/kafdrop
```

Check rollout:

```bash
kubectl rollout status deployment/kafdrop -n kafka-tools
```

---

# 11. Kafdrop Service

Expose Kafdrop internally with a ClusterIP Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kafdrop
  namespace: kafka-tools
spec:
  type: ClusterIP
  selector:
    app: kafdrop
  ports:
    - name: http
      port: 9000
      targetPort: 9000
```

Apply:

```bash
kubectl apply -f 04-service.yaml
```

Check:

```bash
kubectl get svc -n kafka-tools
```

---

# 12. Test Kafdrop Locally

Before exposing it through an ALB, test it using port-forwarding:

```bash
kubectl port-forward \
  -n kafka-tools \
  svc/kafdrop \
  9000:9000
```

Open:

```text
http://localhost:9000
```

---

# 13. Health Check

Kafdrop exposes the Spring Boot Actuator health endpoint:

```text
/actuator/health
```

You can test it through port-forward:

```bash
curl http://localhost:9000/actuator/health
```

Expected response should indicate that the application is healthy.

The official Kafdrop documentation documents Actuator endpoints under `/actuator`.

---

# 14. Pod Disruption Budget

For two Kafdrop replicas, use a PDB:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: kafdrop
  namespace: kafka-tools
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: kafdrop
```

Apply:

```bash
kubectl apply -f 05-pdb.yaml
```

Check:

```bash
kubectl get pdb -n kafka-tools
```

---

# 15. Ingress

For production, keep the Kafdrop Service as `ClusterIP` and expose it through an internal AWS Application Load Balancer.

Example:

```text
Corporate User
      |
      | HTTPS :443
      ▼
Internal AWS ALB
      |
      ▼
Kafdrop Service :9000
      |
      ▼
Kafdrop Pods
```

The ALB should use:

* Internal scheme
* HTTPS
* ACM certificate
* Restricted corporate CIDRs
* Private DNS name
* Appropriate security groups

Example hostname:

```text
kafdrop.prod.example.com
```

---

# 16. Network Security

Kafdrop requires network access to:

```text
AWS MSK :9098
```

and Kubernetes DNS:

```text
UDP/TCP :53
```

The NetworkPolicy should allow:

```text
Kafdrop Pod
    |
    +---- DNS :53
    |
    +---- MSK :9098
```

Do not open unrestricted outbound traffic unless it is required.

---

# 17. ExternalName Services

If your environment requires Kubernetes DNS names for the MSK brokers, you can create:

```text
kafka-broker-1.kafka-tools.svc.cluster.local
kafka-broker-2.kafka-tools.svc.cluster.local
kafka-broker-3.kafka-tools.svc.cluster.local
```

using `ExternalName` Services.

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kafka-broker-1
  namespace: kafka-tools
spec:
  type: ExternalName
  externalName: b-1.kafkaprod.xxxxx.amazonaws.com
```

However, these are only DNS aliases. They are **not Kafka proxies**.

For AWS MSK, the simpler production configuration is generally to use the MSK bootstrap broker names directly:

```text
b-1....amazonaws.com:9098
b-2....amazonaws.com:9098
b-3....amazonaws.com:9098
```

Kafka can subsequently discover the cluster's broker addresses.

---

# 18. Troubleshooting

## Check Pods

```bash
kubectl get pods -n kafka-tools
```

## Check Kafdrop logs

```bash
kubectl logs -n kafka-tools deployment/kafdrop --tail=200
```

## Describe Pod

```bash
kubectl describe pod -n kafka-tools -l app=kafdrop
```

## Check Service

```bash
kubectl get svc -n kafka-tools
```

## Check Endpoints

```bash
kubectl get endpoints -n kafka-tools kafdrop
```

## Check ServiceAccount

```bash
kubectl describe serviceaccount kafdrop -n kafka-tools
```

## Check Deployment

```bash
kubectl describe deployment kafdrop -n kafka-tools
```

---

# 19. Common Problems

### `ClassNotFoundException: IAMLoginModule`

The Kafdrop container does not have the AWS MSK IAM authentication library available.

Verify that the Kafdrop image being used contains the required MSK IAM authentication dependency.

### Authentication failure

Check:

```text
AWS_MSK_IAM
SASL_SSL
9098
IAM Role
IAM Policy
```

### Cannot resolve MSK hostname

Test DNS from inside the cluster:

```bash
kubectl run dns-test \
  -n kafka-tools \
  --rm -it \
  --image=busybox:1.36 \
  -- nslookup b-1.kafkaprod.xxxxx.amazonaws.com
```

### Cannot connect to port 9098

Check:

* MSK security group
* EKS node/pod security group
* MSK subnet routing
* Network ACLs
* Kubernetes NetworkPolicy
* VPC routing

### Kafdrop starts but Kafka connection fails

Check:

```bash
kubectl logs -n kafka-tools deployment/kafdrop
```

Look specifically for:

```text
SASL
SSL
IAM
Authentication
Timeout
UnknownHost
Connection refused
```

---

# 20. Deployment Order

Deploy the resources in this order:

```text
1. Namespace
       ↓
2. IAM Policy
       ↓
3. IAM Role
       ↓
4. ServiceAccount
       ↓
5. NetworkPolicy
       ↓
6. Kafdrop Deployment
       ↓
7. Kafdrop Service
       ↓
8. PDB
       ↓
9. Internal ALB Ingress
       ↓
10. DNS
```

Or, once everything is validated:

```bash
kubectl apply -f 01-namespace.yaml
kubectl apply -f 02-service-account.yaml
kubectl apply -f 03-deployment.yaml
kubectl apply -f 04-service.yaml
kubectl apply -f 05-pdb.yaml
kubectl apply -f 06-ingress.yaml
kubectl apply -f 07-network-policy.yaml
```

---

# 21. Production Checklist

Before considering the deployment production-ready:

* [ ] MSK IAM authentication enabled
* [ ] MSK bootstrap brokers use port `9098`
* [ ] EKS has network connectivity to MSK
* [ ] IAM Role created
* [ ] IAM policy attached
* [ ] Kubernetes ServiceAccount configured
* [ ] Kafdrop Deployment uses the ServiceAccount
* [ ] Kafdrop IAM authentication library verified
* [ ] Kafdrop Service is `ClusterIP`
* [ ] Kafdrop has at least 2 replicas
* [ ] PodDisruptionBudget configured
* [ ] Readiness/liveness probes configured
* [ ] CPU/memory requests and limits configured
* [ ] Topic creation disabled
* [ ] Topic deletion disabled
* [ ] Message sending disabled
* [ ] Internal ALB used for production access
* [ ] HTTPS enabled
* [ ] Corporate CIDRs restricted
* [ ] NetworkPolicy configured
* [ ] DNS configured
* [ ] IAM permissions restricted to required Kafka resources
* [ ] Kafdrop logs verified

---

# 22. Useful Commands

Check everything:

```bash
kubectl get all -n kafka-tools
```

Check Pods:

```bash
kubectl get pods -n kafka-tools -o wide
```

Check logs:

```bash
kubectl logs -n kafka-tools deployment/kafdrop --tail=200
```

Check rollout:

```bash
kubectl rollout status deployment/kafdrop -n kafka-tools
```

Restart Kafdrop:

```bash
kubectl rollout restart deployment/kafdrop -n kafka-tools
```

Check Ingress:

```bash
kubectl get ingress -n kafka-tools
```

Check Service:

```bash
kubectl get svc -n kafka-tools
```

Check PDB:

```bash
kubectl get pdb -n kafka-tools
```

---

## Security Model

The recommended production model is:

```text
                         Corporate Network
                                |
                              HTTPS
                                |
                                ▼
                       Internal AWS ALB
                                |
                                ▼
                        Kafdrop Service
                                |
                                ▼
                         Kafdrop Pod
                                |
                         AWS IAM Identity
                                |
                                ▼
                           AWS IAM Role
                                |
                                ▼
                        AWS MSK :9098
                         SASL_SSL/IAM
```

Kafdrop should be treated as a **monitoring/admin UI**, not as a public application.

Keep it private and expose it only through your corporate network, VPN, or another approved access mechanism.
