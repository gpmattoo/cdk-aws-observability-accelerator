# Singe New EKS Cluster Open Source Observability Accelerator

## Architecture

The following figure illustrates the architecture of the pattern we will be deploying for Single EKS Cluster Open Source Observability pattern using open source tooling such as AWS Distro for Open Telemetry (ADOT), Amazon Managed Service for Prometheus (AMP), Amazon Managed Grafana :

![Architecture](./images/CDK_Architecture_diagram.png)

Monitoring Amazon Elastic Kubernetes Service (Amazon EKS) for metrics has two categories:
the control plane and the Amazon EKS nodes (with Kubernetes objects).
The Amazon EKS control plane consists of control plane nodes that run the Kubernetes software,
such as etcd and the Kubernetes API server. To read more on the components of an Amazon EKS cluster,
please read the [service documentation](https://docs.aws.amazon.com/eks/latest/userguide/clusters.html).

## Objective

- Deploys one production grade Amazon EKS cluster.
- AWS Distro For OpenTelemetry Operator and Collector for Metrics and Traces
- Logs with [AWS for FluentBit](https://github.com/aws/aws-for-fluent-bit)
- Installs Grafana Operator to add AWS data sources and create Grafana Dashboards to Amazon Managed Grafana.
- Installs FluxCD to perform GitOps sync of a Git Repo to EKS Cluster. We will use this later for creating Grafana Dashboards and AWS datasources to Amazon Managed Grafana.
- Installs External Secrets Operator to retrieve and Sync the Grafana API keys.
- Amazon Managed Grafana Dashboard and data source
- Alerts and recording rules with AWS Managed Service for Prometheus

## Prerequisites:

Ensure that you have installed the following tools on your machine.

1. [aws cli](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
2. [kubectl](https://Kubernetes.io/docs/tasks/tools/)
3. [cdk](https://docs.aws.amazon.com/cdk/v2/guide/getting_started.html#getting_started_install)
4. [npm](https://docs.npmjs.com/cli/v8/commands/npm-install)

## Deploying

1. Clone your forked repository

```sh
git clone https://github.com/aws-observability/cdk-aws-observability-accelerator.git
```

2. Install the AWS CDK Toolkit globally on your machine using

    ```bash
    npm install -g aws-cdk
    ```

3. Amazon Managed Grafana workspace

To visualize metrics collected, you need an Amazon Managed Grafana workspace. If you have an existing workspace, create an environment variable as described below.

To create a new workspace, visit [our supporting example for Grafana](https://aws-observability.github.io/terraform-aws-observability-accelerator/helpers/managed-grafana/)

!!! note
    For the URL `https://g-xyz.grafana-workspace.us-east-1.amazonaws.com`, the workspace ID would be `g-xyz`

```bash
export AWS_REGION=<YOUR AWS REGION>
export COA_AMG_WORKSPACE_ID=g-xxx
export COA_AMG_ENDPOINT_URL=https://g-xyz.grafana-workspace.us-east-1.amazonaws.com
```

4. GRAFANA API KEY 

Amazon Managed Grafana provides a control plane API for generating Grafana API keys.

```bash
export GO_AMG_API_KEY=$(aws grafana create-workspace-api-key \
  --key-name "grafana-operator-key-new" \
  --key-role "ADMIN" \
  --seconds-to-live 432000 \
  --workspace-id $COA_AMG_WORKSPACE_ID \
  --query key \
  --output text)
```

5. SSM Parameter for GRAFANA API KEY

Update the Grafana API key secret in AWS SSM Parameter Store using the above new Grafana API key. This will be referenced by Grafana Operator deployment of our solution to access Amazon Managed Grafana from Amazon EKS Cluster

```bash
aws aws ssm put-parameter \
    --name "/cdk-accelerator/grafana-api-key" \
    --type "SecureString" \
    --value $GO_AMG_API_KEY \
    --region $AWS_REGION
```

6. Install project dependencies by running `npm install` in the main folder of this cloned repository

7. Once all pre-requisites are set you are ready to deploy the pipeline. Run the following command from the root of this repository to deploy the pipeline stack:

```bash
make build
make pattern single-new-eks-opensource-observability deploy
```

## Verify the resources

Run update-kubeconfig command. You should be able to get the command from CDK output message.

```sh
aws eks update-kubeconfig --name single-new-eks-opensource-observability-accelerator --region <your region> --role-arn arn:aws:iam::xxxxxxxxx:role/single-new-eks-opensource-singleneweksopensourceob-82N8N3BMJYYI
```

Let’s verify the resources created by Steps above.
```sh
kubectl get nodes # Output shows the EKS Managed Node group nodes

kubectl get ns | kubeflow # Output shows kubeflow namespace

kubectl get pods --namespace=grafana-operator  # Output shows Grafana Operator pods
```

## Visualization

#### 1. Grafana dashboards

Login to your Grafana workspace and navigate to the Dashboards panel. You should see a list of dashboards under the `Observability Accelerator Dashboards`

![Dashboard](./images/All-Dashboards.png)

Open a `Node Exporter` dashboard and you should be able to view its visualization as shown below :

![NodeExporter_Dashboard](./images/Node-Exporter.png)


Open a `Kubelet` dashboard and you should be able to view its visualization as shown below :

![Kubelet_Dashboard](./images/Kubelet.png)

From the cluster to view all dashboards as Kubernetes objects, run

```console
kubectl get grafanadashboards -A
NAMESPACE          NAME                                   AGE
grafana-operator   cluster-grafanadashboard               138m
grafana-operator   java-grafanadashboard                  143m
grafana-operator   kubelet-grafanadashboard               13h
grafana-operator   namespace-workloads-grafanadashboard   13h
grafana-operator   nginx-grafanadashboard                 134m
grafana-operator   node-exporter-grafanadashboard         13h
grafana-operator   nodes-grafanadashboard                 13h
grafana-operator   workloads-grafanadashboard             13h
```

You can inspect more details per dashboard using this command

```console
kubectl describe grafanadashboards cluster-grafanadashboard -n grafana-operator
```

Grafana Operator and Flux always work together to synchronize your dashboards with Git.
If you delete your dashboards by accident, they will be re-provisioned automatically.

## Troubleshooting

### 1. Grafana dashboards missing or Grafana API key expired

In case you don't see the grafana dashboards in your Amazon Managed Grafana console, check on the logs on your grafana operator pod using the below command :

```bash
kubectl get pods -n grafana-operator
```

Output:

```console
NAME                                READY   STATUS    RESTARTS   AGE
grafana-operator-866d4446bb-nqq5c   1/1     Running   0          3h17m
```

```bash
kubectl logs grafana-operator-866d4446bb-nqq5c -n grafana-operator
```

Output:

```console
1.6857285045556655e+09	ERROR	error reconciling datasource	{"controller": "grafanadatasource", "controllerGroup": "grafana.integreatly.org", "controllerKind": "GrafanaDatasource", "GrafanaDatasource": {"name":"grafanadatasource-sample-amp","namespace":"grafana-operator"}, "namespace": "grafana-operator", "name": "grafanadatasource-sample-amp", "reconcileID": "72cfd60c-a255-44a1-bfbd-88b0cbc4f90c", "datasource": "grafanadatasource-sample-amp", "grafana": "external-grafana", "error": "status: 401, body: {\"message\":\"Expired API key\"}\n"}
github.com/grafana-operator/grafana-operator/controllers.(*GrafanaDatasourceReconciler).Reconcile
```

If you observe, the the above `grafana-api-key error` in the logs, your grafana API key is expired. Please use the operational procedure to update your `grafana-api-key` :

- First, lets create a new Grafana API key.

```bash
export GO_AMG_API_KEY=$(aws grafana create-workspace-api-key \
  --key-name "grafana-operator-key-new" \
  --key-role "ADMIN" \
  --seconds-to-live 432000 \
  --workspace-id <YOUR_WORKSPACE_ID> \
  --query key \
  --output text)
```

- Finally, update the Grafana API key secret in AWS Secrets Manager using the above new Grafana API key:

```bash
aws aws ssm put-parameter \
    --name "/terraform-accelerator/grafana-api-key" \
    --type "SecureString" \
    --value "{\"GF_SECURITY_ADMIN_APIKEY\": \"${GO_AMG_API_KEY}\"}" \
    --region <Your AWS Region>
```

- If the issue persists, you can force the synchronization by deleting the `externalsecret` Kubernetes object.

```bash
kubectl delete externalsecret/external-secrets-sm -n grafana-operator
```

