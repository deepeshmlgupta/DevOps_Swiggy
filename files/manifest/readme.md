# End-to-End Guide: Build and Deploy to EKS via CodeBuild with ECR Integration

Audience: DevOps engineers using the AWS Console (UI). CLI snippets are provided only for optional troubleshooting.

Repository/Image/Cluster identifiers used throughout:
- ECR repository: deepesh/swiggy
- EKS cluster: deepesh-eks-cluster
- CodeBuild execution role: arn:aws:iam::058264346116:role/AWS-DevOps-Secret-Role
- Secrets Manager secret: myapp/ci with JSON keys ECR_REPO and EKS_CLUSTER
- Kubernetes manifests: manifest/deployment.yml and manifest/service.yml
- Docker image tag: latest

---

## 1) Overview and Root Causes

This guide documents a working CI/CD path where:
- CodeBuild builds a Docker image and pushes it to Amazon ECR with tag latest.
- A deploy stage uses CodeBuild to kubectl apply Kubernetes manifests to the Amazon EKS cluster.

Common issues encountered and their root causes:
1. ECR login/pipe error in CodeBuild
   - Symptom: “Cannot perform an interactive login from a non TTY device”
   - Root cause: A stray backslash/newline broke the pipe in the login command (`aws ecr get-login-password --region $AWS_DEFAULT_REGION \ | docker login ...`). The pipe split across lines caused an interactive attempt rather than reading from STDIN.

2. EKS access denied for DescribeCluster
   - Symptom: `AccessDeniedException: not authorized to perform eks:DescribeCluster`
   - Root cause: CodeBuild execution role did not have eks:DescribeCluster permission on the target EKS cluster.

3. Kubernetes credentials error
   - Symptom: “the server has asked for the client to provide credentials”
   - Root cause: The CodeBuild execution role was not mapped in the EKS aws-auth ConfigMap, so the role could not authenticate/authorize to the Kubernetes API.

4. Container name mismatch with kubectl set image
   - Symptom: “unable to find container named ‘swiggy-container’”
   - Root cause: The container name provided to kubectl set image did not match the actual container name in the Deployment spec. We avoided set image by always using image: ...:latest plus imagePullPolicy: Always and/or kubectl rollout restart.

---

## 2) Prerequisites

- You can sign in to the AWS Console with permissions to:
  - View/manage ECR repositories and images.
  - Create/read Secrets Manager secrets.
  - Create/edit CodeBuild projects.
  - Attach IAM policies or edit the CodeBuild service role.
  - View/edit the EKS aws-auth ConfigMap.
- Your CodeBuild environment can run Docker builds (privileged mode enabled).
- Your manifests exist in the repo under manifest/deployment.yml and manifest/service.yml.

---

## 3) Step-by-Step: AWS Console Instructions

### A) Validate ECR repository and verify images
1. Open the AWS Console and navigate to Amazon ECR.
2. In the left navigation, select Repositories.
3. Find and select the repository deepesh/swiggy.
   - If it does not exist, create it with the default settings (private repository).
4. Open the Images tab to verify images after a build.
   - Expected tag: latest.

[Insert screenshot placeholder: ECR > Repositories list showing deepesh/swiggy]
[Insert screenshot placeholder: ECR > deepesh/swiggy > Images tab listing latest]

Tip: After completing the build step in this guide, return here to confirm the latest image is pushed.

---

### B) Create or view the Secrets Manager secret (myapp/ci)
We store two values in a single JSON secret to simplify consumption in CodeBuild.

1. Navigate to AWS Secrets Manager in the AWS Console.
2. Choose Store a new secret (or locate myapp/ci if it already exists).
3. Secret type: Other type of secret.
4. For Key/value pairs, enter:
   - Key: ECR_REPO, Value: deepesh/swiggy
   - Key: EKS_CLUSTER, Value: deepesh-eks-cluster
5. Alternatively, set the secret value as JSON:
   {
     "ECR_REPO": "deepesh/swiggy",
     "EKS_CLUSTER": "deepesh-eks-cluster"
   }
6. Secret name: myapp/ci
7. Complete the wizard to create the secret.

[Insert screenshot placeholder: Secrets Manager > Create secret with JSON editor showing ECR_REPO and EKS_CLUSTER]
[Insert screenshot placeholder: Secrets Manager > Secret details for myapp/ci]

Note: We’ll map CodeBuild environment variables from this secret in the project configuration.

---

### C) Review and configure the CodeBuild project (Build stage)
This project builds the Docker image and pushes it to ECR as latest, and produces imageDetail.json as an artifact.

1. Navigate to AWS CodeBuild > Build projects.
2. Create build project (or select your existing build project to edit).
3. Project configuration:
   - Name: swiggy-build (example)
4. Environment:
   - Environment image: Use an AWS managed image (e.g., aws/codebuild/standard:7.0 or latest standard).
   - Operating system/Runtime: Defaults from the selected image are fine.
   - Privileged: Enable (required for Docker builds).
   - Service role: Select existing arn:aws:iam::058264346116:role/AWS-DevOps-Secret-Role.
   - Environment variables:
     - AWS_ACCOUNT_ID: plaintext, value = 058264346116
     - AWS_DEFAULT_REGION: plaintext, value = your region (e.g., ap-south-1 or us-east-1)
     - ECR_REPO: type = Secrets Manager, value = myapp/ci:ECR_REPO
     - EKS_CLUSTER: type = Secrets Manager, value = myapp/ci:EKS_CLUSTER (used by deploy stage if you reuse this project)
5. Buildspec:
   - Choose Use a buildspec file and set the path to buildspec.yml at the repository root.
6. Artifacts:
   - Type: Amazon S3
   - Specify an S3 bucket and a name (e.g., swiggy-build-artifacts).
   - Artifacts packaging: Leave default (ZIP) or None. Ensure imageDetail.json is included.
7. Save the project.

[Insert screenshot placeholder: CodeBuild > Create project > Environment (privileged enabled, service role selected)]
[Insert screenshot placeholder: CodeBuild > Create project > Environment variables with Secrets Manager mapping]
[Insert screenshot placeholder: CodeBuild > Create project > Artifacts configuration]

---

### D) Update IAM permissions on the CodeBuild execution role
Grant the minimal set of permissions for ECR push, reading the secret, logging, artifacts, and describing the EKS cluster.

1. Navigate to AWS IAM > Roles.
2. Find and select role AWS-DevOps-Secret-Role.
3. Attach or update permissions:
   - Option 1 (quicker, broader): Attach managed policies such as:
     - AmazonEC2ContainerRegistryPowerUser (or ECRFullAccess if acceptable)
     - CloudWatchLogsFullAccess (or a minimal custom logs policy)
     - AmazonS3FullAccess (or a minimal S3 access policy for your artifacts bucket)
     - SecretsManagerReadWrite (or minimal GetSecretValue on myapp/ci)
   - Option 2 (recommended minimal): Add an inline policy with least-privilege statements (see sample policy in Appendix A).
4. Save changes.

[Insert screenshot placeholder: IAM > Role: AWS-DevOps-Secret-Role > Permissions tab with attached policies/inline policy]

Key required actions:
- ECR (push and describe): ecr:GetAuthorizationToken, ecr:BatchCheckLayerAvailability, ecr:CompleteLayerUpload, ecr:UploadLayerPart, ecr:InitiateLayerUpload, ecr:PutImage, ecr:BatchGetImage, ecr:DescribeRepositories, ecr:ListImages
- EKS: eks:DescribeCluster (for the target cluster only)
- Secrets Manager: secretsmanager:GetSecretValue, secretsmanager:DescribeSecret (for myapp/ci)
- CloudWatch Logs: logs:CreateLogGroup, logs:CreateLogStream, logs:PutLogEvents
- S3 (artifacts if used): s3:PutObject, s3:GetObject, s3:ListBucket for the artifacts bucket/prefix

---

### E) Map the CodeBuild role in EKS aws-auth ConfigMap
This lets the CodeBuild role authenticate/authorize to your cluster. We’ll temporarily map to the system:masters group to validate flow end-to-end, then discuss least-privilege.

1. Navigate to Amazon EKS > Clusters > deepesh-eks-cluster.
2. Open the Configuration tab.
3. Access or Permissions sub-tab (varies by console version).
4. Locate aws-auth ConfigMap management:
   - Look for a section like “aws-auth ConfigMap” with a View or Edit button.
   - If guided by “Access entries”, expand advanced/compatibility to edit aws-auth ConfigMap.
5. Click Edit and add a new mapRoles entry for the CodeBuild role while preserving existing node role mappings.

Paste/merge the following YAML fragment into the mapRoles list (do not remove existing entries):

mapRoles:
- rolearn: arn:aws:iam::058264346116:role/AWS-DevOps-Secret-Role
  username: aws-developer:build
  groups:
  - system:masters
# Keep your existing node role mapping(s). For example:
- rolearn: arn:aws:iam::<ACCOUNT_ID>:role/<YourNodeInstanceRole>
  username: system:node:{{EC2PrivateDNSName}}
  groups:
  - system:bootstrappers
  - system:nodes

6. Save the ConfigMap.

[Insert screenshot placeholder: EKS > deepesh-eks-cluster > Configuration > Access > Edit aws-auth ConfigMap]
[Insert screenshot placeholder: aws-auth editor with both NodeInstanceRole and AWS-DevOps-Secret-Role present]

Security note: system:masters grants full admin in the cluster. See Section 6 for least-privilege alternatives.

---

### F) Ensure Deployment manifest is compatible with latest-tag strategy
Because the build always pushes :latest, avoid kubectl set image and instead either set imagePullPolicy: Always or perform a rollout restart after apply.

1. Open your manifest/deployment.yml.
2. In the container spec, ensure:
   - image: <account>.dkr.ecr.<region>.amazonaws.com/deepesh/swiggy:latest
   - imagePullPolicy: Always

Minimal example fragment:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: swiggy-deployment
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: swiggy
  template:
    metadata:
      labels:
        app: swiggy
    spec:
      containers:
      - name: swiggy   # Ensure this name matches any kubectl commands if you ever use set image
        image: <AWS_ACCOUNT_ID>.dkr.ecr.<AWS_DEFAULT_REGION>.amazonaws.com/deepesh/swiggy:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8080

[Insert screenshot placeholder: GitHub/Code editor view of manifest/deployment.yml showing imagePullPolicy: Always]

---

### G) Run the build and deploy (Console)
1. Build project:
   - Go to CodeBuild > swiggy-build.
   - Click Start build.
   - Monitor the Phase details and Logs. Confirm it pushes :latest and publishes imageDetail.json.
2. Deploy project:
   - If you use a separate deploy CodeBuild project, run it now (see Section 5 for example buildspec-deploy.yml).
   - It should:
     - Login to ECR (same non-interactive login).
     - Update kubeconfig for the cluster using aws eks update-kubeconfig.
     - kubectl apply -f manifest/deployment.yml -f manifest/service.yml
     - Optionally: kubectl rollout restart deployment/swiggy-deployment

[Insert screenshot placeholder: CodeBuild build logs showing successful docker push to ECR]
[Insert screenshot placeholder: CodeBuild deploy logs showing kubectl apply and rollout outputs]

---

## 4) Corrected Buildspec (Build Stage) and Artifacts

Place the following buildspec.yml at the root of your repository. It:
- Logs in to ECR without the TTY error.
- Builds and pushes the image with the latest tag.
- Writes an artifact imageDetail.json with the pushed image URI.

Key notes:
- Requires environment variables: AWS_ACCOUNT_ID, AWS_DEFAULT_REGION, ECR_REPO.
- Produces artifact imageDetail.json that contains the image URI (useful for subsequent stages or debugging).


version: 0.2

env:
  variables:
    # Set in the CodeBuild console:
    # - AWS_ACCOUNT_ID: 058264346116
    # - AWS_DEFAULT_REGION: e.g., ap-south-1
    # - ECR_REPO: from Secrets Manager, myapp/ci:ECR_REPO
  exported-variables:
    - IMAGE_URI

phases:
  pre_build:
    commands:
      - echo "Logging in to Amazon ECR..."
      - aws ecr get-login-password --region "$AWS_DEFAULT_REGION" | docker login --username AWS --password-stdin "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com"
  build:
    commands:
      - echo "Building the Docker image..."
      - IMAGE_URI="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$ECR_REPO:latest"
      - docker build -t "$IMAGE_URI" .
  post_build:
    commands:
      - echo "Pushing the Docker image to ECR: $IMAGE_URI"
      - docker push "$IMAGE_URI"
      - echo "Writing imageDetail.json artifact..."
      - printf '{"imageUri":"%s","tag":"latest"}' "$IMAGE_URI" > imageDetail.json
artifacts:
  files:
    - imageDetail.json


---

## 5) Optional: Deploy Buildspec (Deploy Stage)

If you run deployment in a separate CodeBuild project, use a short buildspec that:
- Performs ECR login (for pulling images, if needed).
- Acquires kubeconfig against deepesh-eks-cluster.
- Applies the Kubernetes manifests.
- Triggers a rollout restart (since we use :latest).


version: 0.2

env:
  variables:
    # Set in the CodeBuild console:
    # - AWS_ACCOUNT_ID: 058264346116
    # - AWS_DEFAULT_REGION: e.g., ap-south-1
    # - EKS_CLUSTER: from Secrets Manager, myapp/ci:EKS_CLUSTER
    # - ECR_REPO: from Secrets Manager, myapp/ci:ECR_REPO
phases:
  pre_build:
    commands:
      - echo "Logging in to Amazon ECR..."
      - aws ecr get-login-password --region "$AWS_DEFAULT_REGION" | docker login --username AWS --password-stdin "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com"
      - echo "Configuring kubectl for EKS cluster: $EKS_CLUSTER"
      - aws eks update-kubeconfig --name "$EKS_CLUSTER" --region "$AWS_DEFAULT_REGION"
  build:
    commands:
      - echo "Applying Kubernetes manifests..."
      - kubectl apply -f manifest/deployment.yml -f manifest/service.yml
  post_build:
    commands:
      - echo "Restarting deployment to pull :latest..."
      - kubectl rollout restart deployment/swiggy-deployment
      - echo "Waiting for rollout to complete..."
      - kubectl rollout status deployment/swiggy-deployment
artifacts:
  files:
    - manifest/deployment.yml
    - manifest/service.yml

---

## 6) Security Considerations and Least-Privilege Alternative

- Mapping AWS-DevOps-Secret-Role to system:masters simplifies initial validation but grants cluster-admin.
- Least-privilege approach:
  1) Create a Kubernetes Role (or ClusterRole) with only the permissions needed to get/list/apply resources (deployments, services) in the target namespace(s).
  2) Bind that Role to a Kubernetes Group (e.g., ci-deployers).
  3) In aws-auth, map AWS-DevOps-Secret-Role to that group instead of system:masters.
- This reduces blast radius while still allowing the pipeline to deploy.

---

## 7) Verification Checklist

- ECR
  - [ ] ECR > Repositories > deepesh/swiggy > Images shows tag latest.
- CodeBuild (Build)
  - [ ] Build succeeds; logs show docker push to the ECR repository.
  - [ ] Artifact imageDetail.json exists in the configured S3 artifact location and contains the imageUri.
- CodeBuild (Deploy)
  - [ ] aws eks update-kubeconfig succeeds in logs.
  - [ ] kubectl apply for manifest/deployment.yml and manifest/service.yml succeeds.
  - [ ] If using :latest + imagePullPolicy: Always, kubectl rollout restart completes successfully.
- Kubernetes workload
  - [ ] kubectl get deploy shows DESIRED equals AVAILABLE.
  - [ ] kubectl get pods shows new pods running.
  - [ ] kubectl get svc shows service endpoints healthy and accessible as expected.

[Insert screenshot placeholder: ECR image list with latest]
[Insert screenshot placeholder: CodeBuild “Succeeded” status]
[Insert screenshot placeholder: CodeBuild logs showing eks update-kubeconfig and kubectl apply success]
[Insert screenshot placeholder: kubectl get pods output]

---

## 8) FAQ and Troubleshooting

Q1: “Cannot perform an interactive login from a non TTY device.”
- Cause: The ECR login pipe was broken by a stray backslash/newline.
- Fix: Use a one-line pipe without a trailing backslash:

aws ecr get-login-password --region "$AWS_DEFAULT_REGION" | docker login --username AWS --password-stdin "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com"

Q2: `AccessDeniedException: not authorized to perform eks:DescribeCluster`
- Cause: CodeBuild role lacks eks:DescribeCluster on the EKS cluster.
- Fix: Grant eks:DescribeCluster on arn:aws:eks:<region>:058264346116:cluster/deepesh-eks-cluster (see Appendix A).

Q3: “the server has asked for the client to provide credentials”
- Cause: The CodeBuild role is not mapped in aws-auth ConfigMap.
- Fix: Add arn:aws:iam::058264346116:role/AWS-DevOps-Secret-Role to aws-auth under mapRoles with an appropriate group (e.g., system:masters for testing, or a least-privilege group).

Q4: kubectl set image fails with container name not found
- Cause: Container name doesn’t match the Deployment spec.
- Recommended approach: Avoid set image when using :latest. Use imagePullPolicy: Always and/or run kubectl rollout restart deployment/<name> after apply.

---

## Appendix A — Sample Least-Privilege Inline Policy for AWS-DevOps-Secret-Role

Adjust ARNs for your account, region, and S3 artifacts bucket as needed. This example grants only what’s required for the described pipeline.

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EcrAuthToken",
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken"
      ],
      "Resource": "*"
    },
    {
      "Sid": "EcrRepositoryPushPull",
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:CompleteLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:InitiateLayerUpload",
        "ecr:PutImage",
        "ecr:BatchGetImage",
        "ecr:DescribeRepositories",
        "ecr:ListImages"
      ],
      "Resource": "arn:aws:ecr:<REGION>:058264346116:repository/deepesh/swiggy"
    },
    {
      "Sid": "EksDescribeCluster",
      "Effect": "Allow",
      "Action": [
        "eks:DescribeCluster"
      ],
      "Resource": "arn:aws:eks:<REGION>:058264346116:cluster/deepesh-eks-cluster"
    },
    {
      "Sid": "SecretsManagerReadCiSecret",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:<REGION>:058264346116:secret:myapp/ci*"
    },
    {
      "Sid": "CloudWatchLogsWrite",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    },
    {
      "Sid": "ArtifactsBucketMinimal",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::<YOUR_ARTIFACTS_BUCKET>",
        "arn:aws:s3:::<YOUR_ARTIFACTS_BUCKET>/*"
      ]
    }
  ]
}

Replace <REGION> and <YOUR_ARTIFACTS_BUCKET> accordingly.

---

## Appendix B — Example aws-auth ConfigMap (Merged)

This shows both the NodeInstanceRole and the CodeBuild role. When editing in the console, only add the CodeBuild role entry—do not remove existing node mappings.

apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::<ACCOUNT_ID>:role/<YourNodeInstanceRole>
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
    - rolearn: arn:aws:iam::058264346116:role/AWS-DevOps-Secret-Role
      username: aws-developer:build
      groups:
        - system:masters

---

## Appendix C — Notes on Container Name and kubectl commands

- If you ever choose to use kubectl set image, confirm the container name in the Deployment matches your command. For example:
  - Deployment container name: swiggy
  - Command must reference that name: `kubectl set image deployment/swiggy-deployment swiggy=<image-uri>:tag`
- With the :latest strategy, prefer:
  - `imagePullPolicy: Always`
  - `kubectl rollout restart deployment/swiggy-deployment`