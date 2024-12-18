# Tracing with Amazon OpenSearch

## Architecture
![architecture](/assets/arch.jpg)

## Prerequisites
- An Amazon EKS cluster is ready
- EKS addon Load BalancerController and Pod Identity Agent installed
- An OpenSearch Domain is created
- An OpenSearch Ingestion Pipeline is created 

## Steps

Create a IAM role for OpenSearch Ingestion pipeline accessing OpenSearch domain
```sh
cat <<EOF > opensearch-write-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "es:DescribeDomain",
            "Resource": "arn:aws:es:*:{your-account-id}:domain/*"
        },
        {
            "Effect": "Allow",
            "Action": "es:ESHttp*",
            "Resource": "arn:aws:es:*:{your-account-id}:domain/{domain-namedomain}/*"
        }
    ]
}
EOF

POLICY_ARN=$(aws iam create-policy \
    --policy-name OpenSearchWriteIAMPolicy \
    --policy-document file://opensearch-write-policy.json \
    --query Policy.Arn \
    --output text)

# echo policy ARN
echo ${POLICY_ARN}

cat <<EOF > trust-relationship.json
{
   "Version":"2012-10-17",
   "Statement":[
      {
         "Effect":"Allow",
         "Principal":{
            "Service":"osis-pipelines.amazonaws.com"
         },
         "Action":"sts:AssumeRole"
      }
   ]
}
EOF

# Create IAM role
aws iam create-role --role-name OpenSearchWriteIAMRole \
  --assume-role-policy-document file://trust-relationship.json
# Attach IAM policy
aws iam attach-role-policy --role-name OpenSearchWriteIAMRole \
  --policy-arn ${POLICY_ARN}
```
Goto your OpenSearch Domain -> Security configuration, add Access policy as below 。

```sh
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::{your-account-id}:role/OpenSearchWriteIAMRole"
      },
      "Action": ["es:DescribeDomain", "es:*"],
      "Resource": "arn:aws:es:{region}:{your-account-id}:domain/{domain-name}/*"
    }
  ]
}
```
Confiuge OpenSearch Domain Security Role. Goto OpenSearch Dashboard > Management > Security > Roles > Create role

***Name:***  
<Role name, e.g. osi-pipeline-role>  

***Cluster permissions:***  
cluster_monitor,  
cluster_composite_ops,  
cluster_all,  
cluster:admin/ingest/pipeline/put, 
cluster:admin/opendistro/ism/policy/write, 
indices:admin/template/get, 
indices:admin/template/put  

***Index:*** *  

***Index permissions:***  
indices_all, manage_aliases


Once Role is created，configure Mapped users。
OpenSearch Dashboard > Management > Security > Roles > {Role name} > Mapped users
In Backend roles，input OpenSearchWriteIAMRole ARN created in previous step:

Update your OpenSearch Ingestion Pipeline Configuration as below(remember update the varialbe with your accordingly):

```yaml
version: "2"
entry-pipeline:
  source:
    otel_trace_source:
      path: "/v1/traces"
  processor:
    - trace_peer_forwarder:
  sink:
    - pipeline:
        name: "span-pipeline"
    - pipeline:
        name: "service-map-pipeline"
span-pipeline:
  source:
    pipeline:
      name: "entry-pipeline"
  processor:
    - otel_traces:
  sink:
    - opensearch:
        index_type: "trace-analytics-raw"
        hosts: ["https://${AOSDomainEndpoint}"]
        aws:                  
          sts_role_arn: "${OpenSearchIngestionPipelineRole.Arn}"
          region: "${AWS::Region}"
service-map-pipeline:
  source:
    pipeline:
      name: "entry-pipeline"
  processor:
    - service_map:
  sink:
    - opensearch:
        index_type: "trace-analytics-service-map"
        hosts: ["https://${AOSDomainEndpoint}"]
        aws:                  
          sts_role_arn: "${OpenSearchIngestionPipelineRole.Arn}"
          region: "${AWS::Region}"
```
> **Note**: You need to onfiure the security group to allow Adot collector-->Ingestion Pipeline and Ingestion Pipeline-->OpenSearch Domain.

Configure role for OpenTelemetry collector accessing OpenSearch Ingestion pipeline 
```sh
cat <<EOF > adot-pipeline-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PermitsWriteAccessToPipeline",
      "Effect": "Allow",
      "Action": "osis:Ingest",
      "Resource": "arn:aws:osis:{region}:{your-account-id}:pipeline/{pipeline-name}"
    }
  ]
}
EOF

PIPELINE_POLICY_ARN=$(aws iam create-policy \
    --policy-name Collector2PipelineIAMPolicy \
    --policy-document file://adot-pipeline-policy.json \
    --query Policy.Arn \
    --output text)

# policy ARN
echo ${PIPELINE_POLICY_ARN}

cat <<EOF > pod-trust-relationship.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "pods.eks.amazonaws.com"
            },
            "Action": [
                "sts:AssumeRole",
                "sts:TagSession"
            ]
        }
    ]
}
EOF

# Create IAM role
aws iam create-role --role-name Adot2PipelineIAMRole \
  --assume-role-policy-document file://pod-trust-relationship.json

aws iam attach-role-policy --role-name Adot2PipelineIAMRole \
  --policy-arn ${PIPELINE_POLICY_ARN}
```

Create SA for Adot collector
```sh
kubectl create sa otel-collector-sa -n otel-collector
```

Associate adot role and sa
```sh
aws eks create-pod-identity-association \
  --cluster-name <value> \
  --namespace otel-collector \
  --service-account otel-collector-sa \
  --role-arn <value>
```

Build miroservices apps and push docker image(***before you run the script, please update line 11 with your Ingestion pipeline name***):  
```sh
./scripts/01-build-push.sh
```

Deploy adot collecoter and microservices apps:
```sh
./scripts/02-apply-k8s-manifests.sh
```
Check the status of pods:
```sh
kubectl get pods -A
NAMESPACE                NAME                                            READY   STATUS    RESTARTS   AGE
analytics-service        analytics-service-59788b56bc-rq9v4              1/1     Running   0          10h
authentication-service   authentication-service-7c5475496b-tsbdv         1/1     Running   0          10h
client-service           client-service-8b78894b4-p7d26                  1/1     Running   0          10h
database-service         database-service-54b65bc5c8-xlncp               1/1     Running   0          10h
inventory-service        inventory-service-55fd6f876f-mcjc7              1/1     Running   0          10h
kube-system              aws-load-balancer-controller-85445687df-4jbrn   1/1     Running   0          10h
kube-system              aws-load-balancer-controller-85445687df-r5q4c   1/1     Running   0          10h
kube-system              aws-node-fsszb                                  2/2     Running   0          23h
kube-system              aws-node-stq28                                  2/2     Running   0          11h
kube-system              aws-node-x9j6r                                  2/2     Running   0          23h
kube-system              coredns-586b798467-222p5                        1/1     Running   0          23h
kube-system              coredns-586b798467-hx6sn                        1/1     Running   0          23h
kube-system              eks-pod-identity-agent-88xh6                    1/1     Running   0          11h
kube-system              eks-pod-identity-agent-k7mq2                    1/1     Running   0          11h
kube-system              eks-pod-identity-agent-qfxvs                    1/1     Running   0          11h
kube-system              kube-proxy-8x8rn                                1/1     Running   0          23h
kube-system              kube-proxy-j9qnh                                1/1     Running   0          11h
kube-system              kube-proxy-qvwqv                                1/1     Running   0          23h
mysql                    mysql-7fc594f4f7-2z8pl                          1/1     Running   0          10h
order-service            order-service-76d57b8c89-lgpc2                  1/1     Running   0          10h
otel-collector           otel-collector-84f4d99bbd-j4hs6                 1/1     Running   0          5h14m
payment-service          payment-service-7dff54fcd9-8lwmq                1/1     Running   0          10h
recommendation-service   recommendation-service-6885bcc444-74bcn         1/1     Running   0          10h
```
Get the client-service url:
```sh
kubectl get svc -A
NAME             TYPE           CLUSTER-IP      EXTERNAL-IP                                                                     PORT(S)        AGE
client-service   LoadBalancer   172.20.15.111   k8s-clientse-clientse-xxxxx-xxxxx.elb.us-east-1.amazonaws.com   80:32052/TCP   10h
```

Access the demo shop and then will generate trace metrics:
![shop](/assets/shop.png)

Goto OpenSearch Dashboard > Observability > Traces to view the traces we just generated:
![shop](/assets/db1.jpeg)
![shop](/assets/db2.jpeg)