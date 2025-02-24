# Use Cases

## The manual mode: plan and manual apply

For the plan & manual approval workflow, please either set `.spec.approvePlan` to be the blank value, or omit the field.

```diff
apiVersion: infra.contrib.fluxcd.io/v1alpha1
kind: Terraform
metadata:
  name: helloworld
  namespace: flux-system
spec:
+ approvePlan: "" # or you can omit this field
- approvePlan: "auto"
  interval: 1m
  path: ./
  sourceRef:
    kind: GitRepository
    name: helloworld
    namespace: flux-system
```

Then the controller will tell you how to use field `.spec.approvePlan` to approve the plan.
After making change and push, it will apply the plan to create real resources.

```diff
apiVersion: infra.contrib.fluxcd.io/v1alpha1
kind: Terraform
metadata:
  name: hello-world
  namespace: flux-system
spec:
+ approvePlan: "plan-main-b8e362c206" # first 8 digits of a commit hash is enough
- approvePlan: ""
  interval: 1m
  path: ./
  sourceRef:
    kind: GitRepository
    name: helloworld
    namespace: flux-system
```

## The drift detection only mode: plan and apply will be skipped

To only run drift detection, skipping the plan and apply stages, set `.spec.approvePlan` to `disable`.

```yaml
apiVersion: infra.contrib.fluxcd.io/v1alpha1
kind: Terraform
metadata:
  name: hello-world
  namespace: flux-system
spec:
  approvePlan: "disable"
  interval: 1m
  path: ./
  sourceRef:
    kind: GitRepository
    name: helloworld
    namespace: flux-system
```

## Disable Drift Detection

Drift detection is enabled by default. Use the `.spec.disableDriftDetection` field to disable:

```yaml
apiVersion: infra.contrib.fluxcd.io/v1alpha1
kind: Terraform
metadata:
  name: helloworld
  namespace: flux-system
spec:
  approvePlan: "auto"
  disableDriftDetection: true
  interval: 1m
  path: ./
  sourceRef:
    kind: GitRepository
    name: helloworld
    namespace: flux-system
```

## Use with AWS EKS IRSA

AWS Elastic Kubernetes Service (EKS) offers IAM Roles for Service Accounts (IRSA) as a mechanism by which to provide
credentials to Kubernetes pods. This can be used to provide the required AWS credentials to Terraform runners
for performing plans and applies.

You can use `eksctl` to associate an OIDC provider with your EKS cluster, for example:

```shell
eksctl utils associate-iam-oidc-provider --cluster CLUSTER_NAME --approve
```

Then follow the instructions [here](https://docs.aws.amazon.com/eks/latest/userguide/create-service-account-iam-policy-and-role.html)
to add a trust policy to the IAM role which grants the necessary permissions for Terraform.
Please note that if you have installed the controller following the README, then the `namespace:serviceaccountname`
will be `flux-system:tf-runner`. You'll obtain a Role ARN to use in the next step.

Finally, annotate the ServiceAccount for the `tf-runner` with the obtained Role ARN in your cluster:

```shell
kubectl annotate -n flux-system serviceaccount tf-runner eks.amazonaws.com/role-arn=ROLE_ARN
```

If deploying the `tf-controller` via Helm, this can be accomplished as follows:
```yaml
values:
  runner:
    serviceAccount:
      annotations:
        eks.amazonaws.com/role-arn: ROLE_ARN
```

## Setting Terraform Variables

**This is a breaking change of the `v1alpha1` API.**
Users who are upgrading from TF-controller <= 0.7.0 require updating `varsFrom`,
from a single object:
```yaml
  varsFrom:
    kind: ConfigMap
    name: cluster-config
```
to be an array of object, like this:
```yaml
  varsFrom:
  - kind: ConfigMap
    name: cluster-config
```

You can pass variables to Terraform using the `vars` and `varsFrom` fields.

Inline variables can be set using `vars`. The `varsFrom` field accepts a list of ConfigMaps / Secrets.
You may use the `varsKeys` property of `varsFrom` to select specific keys from the input or omit this field
to select all keys from the input source.

Note that in the case of the same variable key being passed multiple times, the controller will use
the lattermost instance of the key passed to `varsFrom`.

```yaml
apiVersion: infra.contrib.fluxcd.io/v1alpha1
kind: Terraform
metadata:
  name: helloworld
  namespace: flux-system
spec:
  approvePlan: "auto"
  interval: 1m
  path: ./
  sourceRef:
    kind: GitRepository
    name: helloworld
    namespace: flux-system
  vars:
  - name: region
    value: us-east-1
  - name: env
    value: dev
  - name: instanceType
    value: t3-small
  varsFrom:
  - kind: ConfigMap
    name: cluster-config
    varsKeys:
    - nodeCount
    - instanceType
  - kind: Secret
    name: cluster-creds
```

The `vars` field supports HCL string, number, bool, object and list types. For example, the following variable can be populated using the accompanying Terraform spec:

```hcl
variable "cluster_spec" {
  type = object({
      region = string
      env = string
      node_count = number
      public = bool
  })
}
```

```yaml
apiVersion: infra.contrib.fluxcd.io/v1alpha1
kind: Terraform
metadata:
  name: helloworld
  namespace: flux-system
spec:
  approvePlan: "auto"
  interval: 1m
  path: ./
  sourceRef:
    kind: GitRepository
    name: helloworld
    namespace: flux-system
  vars:
  - name: cluster_spec
    value:
      region: us-east-1
      env: dev
      node_count: 10
      public: false
```

## Managing Terraform State

By default, `tf-controller` will use the [Kubernetes backend](https://www.terraform.io/language/settings/backends/kubernetes) to store the Terraform statefile in cluster.

The statefile is stored in a secret named: `tfstate-default-${secretSuffix}`. The default `suffix` will be the name of the Terraform resource, however you may override this setting using `.spec.backendConfig.secretSuffix`.

You can disable the backend

### Use custom backend

If you wish to use a custom backend, you can configure it by defining the `backendConfig.customConfiguration` with one of the backends such as GCS or S3:

```yaml
apiVersion: infra.contrib.fluxcd.io/v1alpha1
kind: Terraform
metadata:
  name: helloworld
  namespace: flux-system
spec:
  approvePlan: "auto"
  backendConfig:
    customConfiguration: |
      backend "s3" {
        bucket                      = "s3-terraform-state1"
        key                         = "dev/terraform.tfstate"
        region                      = "us-east-1"
        endpoint                    = "http://localhost:4566"
        skip_credentials_validation = true
        skip_metadata_api_check     = true
        force_path_style            = true
        dynamodb_table              = "terraformlock"
        dynamodb_endpoint           = "http://localhost:4566"
        encrypt                     = true
      }
  interval: 1m
  path: ./
  sourceRef:
    kind: GitRepository
    name: helloworld
    namespace: flux-system
  runnerPodTemplate:
    spec:
      image: registry.io/tf-runner:xyz
```


### Backup the statefile

For the following `terraform` resources:

```bash
$ kubectl get terraform

NAME       READY     STATUS         AGE
my-stack   Unknown   Initializing   28s
```

We can export the state like this:
```bash
kubectl get secret tfstate-default-my-stack -ojsonpath='{.data.tfstate}' | base64 -d | gzip -d > terraform.tfstate
```

### Restore the statefile

To restore the statefile or import an existing statefile we can use the following operation:

```bash
gzip terraform.tfstate

NAME=my-stack

kubectl create secret \
  generic tfstate-default-${NAME} \
  --from-file=tfstate=terraform.tfstate.gz \
  --dry-run=client -o=yaml \
  | yq e '.metadata.annotations["encoding"]="gzip"' - > tfstate-default-${NAME}.yaml

kubectl apply -f tfstate-default-${NAME}.yaml
```

## Health Checks

For some resources, it may be useful to perform health checks on them to verify that they are ready to accept connection before the terraform goes into `Ready` state:

```
# main.tf

output "rdsAddress" {
  value = "mydb.xyz.us-east-1.rds.amazonaws.com"
}

output "rdsPort" {
  value = "3306"
}

output "myappURL" {
  value = "https://example.com/"
}
```

```yaml
apiVersion: infra.contrib.fluxcd.io/v1alpha1
kind: Terraform
metadata:
  name: helloworld
  namespace: flux-system
spec:
  approvePlan: "auto"
  interval: 1m
  path: ./
  sourceRef:
    kind: GitRepository
    name: helloworld
    namespace: flux-system
  healthChecks:
    - name: rds
      type: tcp
      address: "{{.rdsAddress}}:{{.rdsPort}}" # uses standard Go package template format to parse outputs to url
      timeout: 10s # optional, defaults to 20s
    - name: myapp
      type: http
      url: "{{.myappURL}}"
      timeout: 5s
    - name: url_not_from_output
      type: http
      url: "https://example.org"
```

## Destroy resources on deletion

The resources created by terraform are not defaulted to destroyed after the object is deleted from the cluster. To enable destroy resources on object deletion, set `.spec.destroyResourcesOnDeletion` to `true`.

```yaml
apiVersion: infra.contrib.fluxcd.io/v1alpha1
kind: Terraform
metadata:
  name: helloworld
  namespace: flux-system
spec:
  approvePlan: "auto"
  destroyResourcesOnDeletion: true
  interval: 1m
  path: ./
  sourceRef:
    kind: GitRepository
    name: helloworld
    namespace: flux-system
```

## Write outputs to a secret

Outputs created by Terraform can be written to a secret using `.spec.writeOutputsToSecret`.

### Write all outputs

We can specify a target secret, and the controller will write all outputs to the secret by default.

```yaml
apiVersion: infra.contrib.fluxcd.io/v1alpha1
kind: Terraform
metadata:
  name: helloworld
  namespace: flux-system
spec:
  approvePlan: "auto"
  interval: 1m
  path: ./
  sourceRef:
    kind: GitRepository
    name: helloworld
    namespace: flux-system
  writeOutputsToSecret:
    name: helloworld-output
```

### Write outputs selectively

We can choose only a subset of outputs by specify output names we'd like to write in `outputs` array.

```yaml
apiVersion: infra.contrib.fluxcd.io/v1alpha1
kind: Terraform
metadata:
  name: helloworld
  namespace: flux-system
spec:
  approvePlan: "auto"
  interval: 1m
  path: ./
  sourceRef:
    kind: GitRepository
    name: helloworld
    namespace: flux-system
  writeOutputsToSecret:
    name: helloworld-output
    outputs:
    - hello_world
    - my_sensitive_data
```

### Output name mapping

Some time we'd like to use rename an output, so that it can be consumed by other Kubernetes controllers.
For example, we might retrieve a key from a Secret manager, and it's an AGE key, which must be ending with ".agekey" in the secret.
In this case, we need to rename the output. TF-controller supports mapping output name using the "old_name:new_name" format.
In the following example, we write `age_key` output as `age.agekey` entry in the `helloworld-output` Secret's data.

```yaml
apiVersion: infra.contrib.fluxcd.io/v1alpha1
kind: Terraform
metadata:
  name: helloworld
  namespace: flux-system
spec:
  approvePlan: "auto"
  interval: 1m
  path: ./
  sourceRef:
    kind: GitRepository
    name: helloworld
    namespace: flux-system
  writeOutputsToSecret:
    name: helloworld-output
    outputs:
    - age_key:age.agekey
```

## Customize runner pod

### Customize runner pod metadata

In some situations, it is needed to add custom labels and annotations to the runner pod used to reconcile Terraform.
For example, for Azure AKS to grant pod active directory permissions using Azure Active Directory (AAD) Pod Identity,
a label like `aadpodidbinding: myIdentity` on the pod is required.

```yaml
apiVersion: infra.contrib.fluxcd.io/v1alpha1
kind: Terraform
metadata:
  name: helloworld
  namespace: flux-system
spec:
  approvePlan: "auto"
  interval: 1m
  path: ./
  sourceRef:
    kind: GitRepository
    name: helloworld
    namespace: flux-system
  runnerPodTemplate:
    metadata:
      labels:
        aadpodidbinding: myIdentity
      annotations:
        company.com/abc: xyz
```

### Customize runner pod image

By default, the Terraform controller uses `RUNNER_POD_IMAGE` environment variable to identify the runner pod image to use.
You can customize the image to use on the global level by updating the value of the environment variable or you
can specify an image to use per Terraform object for its reconciliation.

```yaml
apiVersion: infra.contrib.fluxcd.io/v1alpha1
kind: Terraform
metadata:
  name: helloworld
  namespace: flux-system
spec:
  approvePlan: "auto"
  interval: 1m
  path: ./
  sourceRef:
    kind: GitRepository
    name: helloworld
    namespace: flux-system
  runnerPodTemplate:
    spec:
      image: registry.io/tf-runner:xyz
```

You can use [`runner.Dockerfile`](https://github.com/weaveworks/tf-controller/blob/main/runner.Dockerfile) as a basis of customizing runner pod image.

## Using OCI Artifact as Source

To use OCI artifacts as the source of TF-controller, you need Flux2 version v0.32.0 or higher.

Assuming that you have Terraform files (your root module may contain sub-modules) under ./modules,
you can use Flux CLI to create an OCI artifact for your Terraform modules
by running the following commands:

```bash
flux push artifact oci://ghcr.io/tf-controller/helloworld:$(git rev-parse --short HEAD) \
    --path="./modules" \
    --source="$(git config --get remote.origin.url)" \
    --revision="$(git branch --show-current)/$(git rev-parse HEAD)"

flux tag artifact oci://ghcr.io/tf-controller/helloworld:$(git rev-parse --short HEAD) \
    --tag main
```

Then you define a source (`OCIRepository`), and use it as the `sourceRef` of your Terraform object.

```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: OCIRepository
metadata:
  name: helloworld-oci
spec:
  interval: 1m
  url: oci://ghcr.io/tf-controller/helloworld
  ref:
    tag: main
---
apiVersion: infra.contrib.fluxcd.io/v1alpha1
kind: Terraform
metadata:
  name: helloworld-tf-oci
spec:
  path: ./
  approvePlan: "auto"
  interval: 1m
  sourceRef:
    kind: OCIRepository
    name: helloworld-oci
  writeOutputsToSecret:
    name: helloworld-outputs
```
