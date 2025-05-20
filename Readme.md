# Multi-Agent AI Campaign Planning System on AWS

## Introduction

Marketing campaigns often involve complex, multi-step planning processes that can benefit from AI automation. In this solution, we deploy a **multi-agent AI campaign planning system** on AWS. The system coordinates multiple specialized AI agents to take a marketing brief from input to a full campaign plan, using AWS’s serverless and AI services. By leveraging **Amazon Bedrock** foundation models (e.g. Anthropic Claude 3 and Amazon Titan) for generation and **AWS Step Functions** for orchestration, we enable autonomous collaboration between agents to solve complex planning tasks. Each agent is implemented as an AWS Lambda function specializing in a sub-task (input parsing, interpretation, planning, content creation, channel strategy, or evaluation). A Step Functions workflow chains these Lambdas, ensuring they work in harmony towards the campaign goals. Supporting AWS components – Amazon S3, DynamoDB, Kendra (or Amazon Bedrock Knowledge Bases via OpenSearch), and SNS – provide storage, knowledge retrieval, and notifications. This notebook will guide you through deploying all these resources and configuring the AI agents, complete with **secure IAM roles**, **error handling**, and **logging** for a production-ready solution.

## Architecture and Components

&#x20;*Figure: High-level architecture combining a knowledge base (documents in S3 indexed by an OpenSearch vector DB), Bedrock foundation models (FMs) for generation, AWS Lambda functions (orange) as specialized agents, and DynamoDB (purple) for state storage. This serverless design is orchestrated by a Step Functions workflow (not pictured) to enable multi-agent collaboration.*

Our campaign planner system consists of the following AWS components:

* **Amazon Bedrock Models (Claude 3, Amazon Titan)** – These provide the AI capabilities for each agent role. Amazon Bedrock offers access to top-tier foundation models via API. Here we use *Anthropic Claude v3* (a powerful conversational model) and *Amazon Titan Text* (Amazon’s own generative model) for various tasks. Each agent Lambda will invoke a Bedrock model to perform its generative task (e.g. interpreting input, writing content).
* **AWS Lambda Functions (Agents 0–5)** – We define six Lambda functions, one per agent role:

  * *Agent 0: Input Extractor* – Parses the initial marketing brief and extracts key information (e.g. target audience, product details, campaign objectives).
  * *Agent 1: Business Interpreter* – Interprets the business context and goals from the extracted input, possibly consulting the knowledge base for additional insights (market trends, past campaigns).
  * *Agent 2: Campaign Planner* – Develops the campaign strategy (timeline, channels, messaging themes) for the given audience and objectives.
  * *Agent 3: Content Generator* – Produces sample campaign content (social media posts, ad copy, etc.), using templates appropriate to the target segment (Gen-Z or Enterprise).
  * *Agent 4: Channel Strategist* – Recommends distribution channels and scheduling, tailoring to where the target audience (e.g. Gen-Z vs B2B enterprise) is most active.
  * *Agent 5: Evaluator* – Reviews the plan and content, estimating KPIs (engagement or lead metrics) and ensuring the campaign meets the original goals.
    Each Lambda runs Python code (using `boto3` to call Bedrock models) and is configured with appropriate IAM permissions. The agents communicate sequentially via Step Functions, and persist important data to DynamoDB as needed for cross-step access.
* **AWS Step Functions (Orchestration State Machine)** – Step Functions coordinates the workflow, invoking each Lambda in turn and passing results along. This ensures the multi-step generative process executes in order and with proper error handling. Step Functions is ideal for orchestrating such multi-agent workflows: it can coordinate multiple AWS services into cohesive serverless workflows and even retry or catch errors automatically so that the process is robust. In our design, the state machine will execute Agents 0 → 1 → 2 → 3 → 4 → 5 in sequence (with branching or parameterization to handle Gen-Z vs Enterprise logic). Logging is enabled so that each step’s input/output and any errors are recorded in CloudWatch for monitoring.
* **Amazon S3 (Briefs and Assets storage)** – S3 buckets store input documents and generated campaign assets. For example, a marketing brief (PDF/Doc or JSON) can be uploaded to S3 and trigger the workflow, and any content pieces generated (images, text files) can be saved to S3 for review. S3 provides a durable, scalable store for these large objects. We will create an S3 bucket to hold campaign briefs and another for any content assets or knowledge base documents. (In this notebook we’ll populate the brief bucket with a dummy brief for demonstration.)
* **Amazon DynamoDB (Data Store)** – DynamoDB is used to store structured data that the agents produce or need. For instance, Agent 0’s extracted fields can be stored as an item, which Agent 1 or others can retrieve. DynamoDB serves as a system-of-record for campaign state (parsed inputs, interim results, final outputs, and status flags). It’s a fast NoSQL database that is fully managed and suited for storing JSON-like items. We will create a DynamoDB table to hold campaign data (partitioned by Campaign ID). As each agent runs, it can update this table with its results and status. This allows persistence beyond the Step Functions execution (in case we want to query results later) and decouples data sharing between agents.
* **Amazon Kendra (Knowledge Base)** – To enrich the campaign planning with context, we use Amazon Kendra (or Amazon Bedrock’s Knowledge Base feature with an OpenSearch vector index) as a **near real-time knowledge base**. Amazon Kendra is an intelligent search service that can index unstructured data and enable semantic search and Q\&A over it. We will simulate a knowledge repository of marketing data – for example, previous campaign briefs, audience research reports, product FAQs, messaging guidelines, etc. The Business Interpreter agent (and possibly others) can query this knowledge base to ground its outputs in facts. In our deployment, we create a Kendra index and ingest some dummy documents (like a “Gen Z social media trends 2024” PDF and an “Enterprise SaaS lead gen tips” text) to illustrate. This allows an agent to query, for example, “What are top channels for Gen-Z engagement?” and get relevant context to incorporate in the plan. *(Note: Amazon Bedrock knowledge bases can also be used with an OpenSearch Serverless vector store for retrieval-augmented generation; the setup is similar – for simplicity we demonstrate with Kendra.)*
* **Amazon SNS (Notifications)** – SNS (Simple Notification Service) is used to send notifications of campaign planning outcomes or alerts. For example, when the campaign plan is fully generated (or if a failure occurs), the system can publish a message to an SNS topic. Subscribers (which could be email endpoints, SMS, or applications) will get notified. This keeps marketing teams informed in real time. We will set up an SNS topic and (optionally) subscribe an email to it for demonstration. In production, this could trigger an email to the marketing lead that “Campaign Plan XYZ is ready” with details.
* **AWS IAM (Security Roles)** – We create dedicated IAM roles to follow least-privilege access for each component: e.g. Lambda execution roles that allow only the necessary actions (invoke Bedrock models, read/write S3 and DynamoDB, query Kendra, publish to SNS), a role for Step Functions to invoke Lambdas and SNS, and a role for Kendra to access S3 (if needed). By using proper IAM roles and policies, we ensure the system is secure and each service interacts only as authorized.

**Campaign Logic – Gen-Z vs Enterprise:** Our system is designed to handle different campaign contexts: **consumer campaigns targeting Gen-Z** (focused on brand awareness and engagement) versus **B2B enterprise campaigns** (focused on lead generation and conversion). This affects content tone, channel selection, and success metrics. We incorporate this in the agents’ prompts and logic. For example, Agent 2 (Planner) will set goals like “maximize TikTok engagement and brand mentions” for a Gen-Z campaign vs. “generate qualified leads (MQLs) via LinkedIn and email” for an enterprise campaign. Agent 3 (Content Generator) uses different templates: a Gen-Z post might include casual language, memes or emojis, whereas enterprise content will be formal and data-driven. According to marketing research, 45% of marketers prioritize brand awareness while 36% prioritize lead generation as a top goal – our system dynamically balances these priorities based on the campaign type. We will demonstrate how the **audience segment** (Gen-Z consumer vs Enterprise/B2B) can be passed as an input parameter and influence the agents’ behavior (e.g. through conditional logic in Lambda or a Choice state in Step Functions).

## Setup Instructions (AWS Access & Environment)

Before deploying resources, ensure your AWS credentials and region are set up for this notebook. You’ll need AWS programmatic access with rights to create the above resources. If running in an environment like AWS SageMaker or Cloud9, an IAM role with appropriate permissions can be attached instead of access keys.

1. **AWS Credentials:** If not already configured, provide your AWS Access Key ID, Secret Access Key, and default region. For security, avoid hard-coding secrets in the notebook; instead, store them securely (e.g. use environment variables or AWS config files). For example, run the following in a code cell to set environment variables (or ensure `~/.aws/credentials` is configured):

   ```python
   import os
   os.environ['AWS_ACCESS_KEY_ID'] = 'AKIA....YOURKEY'
   os.environ['AWS_SECRET_ACCESS_KEY'] = 'YOURSECRET'
   os.environ['AWS_DEFAULT_REGION'] = 'us-east-1'  # choose a region with Bedrock support
   ```

   Alternatively, use AWS CLI configuration or an IAM role. **Region:** Use a region that supports Amazon Bedrock (e.g. `us-east-1` or `us-west-2` as of this writing) and Amazon Kendra (many major regions). Make sure your account is enabled for Bedrock API access.
2. **Python Environment:** This notebook uses `boto3` (AWS SDK for Python). It should be pre-installed in most AWS environments. If not, install via `!pip install boto3`. Also, for logging, we use Python’s `logging` module.
3. **Permissions:** The AWS IAM user/role running this notebook must have rights to create IAM roles, Lambdas, Step Functions state machines, S3 buckets, DynamoDB tables, Kendra indexes, and SNS topics, as well as to invoke Bedrock models. If using an admin role, that’s fine for setup, but for production each service will get its own limited role as we configure below.

With credentials set, we can proceed to deploy the infrastructure. We’ll use **boto3** clients for each service to create resources programmatically. (You can also deploy via CloudFormation or AWS CDK, but here we’ll show direct SDK calls for clarity.)

## 1. Provision AWS Resources

Let’s create the AWS resources needed for our system in a step-by-step manner. This includes S3 buckets, a DynamoDB table, a Kendra index, SNS topic, IAM roles, Lambda functions, and the Step Functions state machine. We will also upload some **dummy data** (marketing briefs, knowledge base docs, etc.) as part of the setup.

### 1.1 Create S3 Buckets for Briefs and Assets

We’ll create two S3 buckets: one for input briefs (`campaign-briefs`) and one for any output assets or knowledge documents (`campaign-assets`). Bucket names must be globally unique, so you may need to modify the names (here we include a random suffix for uniqueness).

```python
import boto3, json, random, string

region = os.environ.get('AWS_DEFAULT_REGION', 'us-east-1')
s3 = boto3.client('s3', region_name=region)

# Define unique bucket names
suffix = ''.join(random.choice(string.ascii_lowercase + string.digits) for _ in range(6))
briefs_bucket = f"ai-campaign-briefs-{suffix}"
assets_bucket = f"ai-campaign-assets-{suffix}"

for bucket in [briefs_bucket, assets_bucket]:
    try:
        s3.create_bucket(Bucket=bucket, CreateBucketConfiguration={'LocationConstraint': region})
        print(f"Created bucket: {bucket}")
    except Exception as e:
        print(f"Bucket creation failed (maybe name not unique or permission issue): {e}")
```

This will create two new S3 buckets in your region. We’ll use `briefs_bucket` to store incoming campaign brief files and `assets_bucket` for storing generated content or any knowledge base docs.

Next, let’s populate a dummy marketing brief. We create a sample brief in JSON format containing fields for campaign name, target segment, product, objectives, etc., and upload it to the `campaign-briefs` bucket. In practice, this could be a file provided by a user (e.g., via an API or web interface), but for our demo we’ll just create a test JSON.

```python
# Dummy marketing brief content
campaign_brief = {
    "campaign_id": "CAMPAIGN123",
    "campaign_name": "GenZ Social Hype 2025",
    "target_audience": "Gen-Z",
    "product": "Tech Gadget X",
    "objectives": "Drive brand engagement and social buzz among Gen-Z users for Tech Gadget X launch.",
    "key_messages": "Trendy, innovative, sustainable; speak the Gen-Z lingo; emphasize community and creativity.",
    "channels": "social_media",
    "timeline": "Q3 2025",
    "budget": "50k USD"
}
brief_key = "sample_brief_genz.json"
# Upload the sample brief to S3
s3.put_object(Bucket=briefs_bucket, Key=brief_key, Body=json.dumps(campaign_brief))
print(f"Uploaded dummy brief to s3://{briefs_bucket}/{brief_key}")
```

We now have a sample campaign brief for a Gen-Z campaign in S3. We’ll later use this as the input for our Step Function.

*(Similarly, you could add an enterprise brief – for brevity, we proceed with one scenario. The system will handle either based on `target_audience` field.)*

### 1.2 Create DynamoDB Table for Campaign Data

Now, we create a DynamoDB table to persist the campaign data through the workflow. Each campaign can be an item, identified by a primary key (e.g. `campaign_id`). We also include a sort key for flexibility (e.g. `data_type` to store different items like “input”, “plan”, “content”, etc. under the same campaign\_id if desired). For simplicity, we’ll use a composite key of `campaign_id` (string hash key) and `step` (string range key), so each agent’s output can be a separate item.

```python
dynamodb = boto3.client('dynamodb', region_name=region)

table_name = f"AI_CampaignData_{suffix}"
try:
    # Define a table with a composite primary key: campaign_id (HASH) and step (RANGE)
    table_resp = dynamodb.create_table(
        TableName=table_name,
        AttributeDefinitions=[
            {"AttributeName": "campaign_id", "AttributeType": "S"},
            {"AttributeName": "step", "AttributeType": "S"}
        ],
        KeySchema=[
            {"AttributeName": "campaign_id", "KeyType": "HASH"},
            {"AttributeName": "step", "KeyType": "RANGE"}
        ],
        BillingMode='PAY_PER_REQUEST'  # on-demand pricing
    )
    dynamodb.get_waiter('table_exists').wait(TableName=table_name)
    print(f"DynamoDB Table created: {table_name}")
except Exception as e:
    print(f"Error creating DynamoDB table (it might already exist or permission issue): {e}")
```

This will create the DynamoDB table and wait until it’s active. We use on-demand billing for simplicity (no provisioning needed). The table will be used by Lambdas to `PutItem` or `UpdateItem` their results and to `GetItem` data from previous steps. For example, Agent 0 will store the parsed brief as item `{campaign_id: "CAMPAIGN123", step: "input_extracted", ...}`; Agent 1 can retrieve that and then store `{campaign_id: "CAMPAIGN123", step: "business_context", ...}`, and so on.

### 1.3 Set Up Amazon Kendra (Knowledge Base Index)

Next, we set up an **Amazon Kendra index** to serve as our knowledge base. We will ingest some dummy documents relevant to campaigns. *Note:* Kendra index creation can take a few minutes and requires an IAM role. For the sake of completeness, we will illustrate the steps, but depending on environment you may choose to skip actual Kendra creation to save time/costs and instead mock knowledge retrieval in the Lambdas.

Let’s create an IAM role for Kendra first, which Kendra will use to access S3 (if we use S3 as a data source). We attach the AmazonKendraReadOnlyAccess policy for simplicity (or create a custom policy granting read access to the specific bucket and the ability to write index data).

```python
iam = boto3.client('iam')
kendra_role_name = f"KendraIndexRole-{suffix}"
try:
    # Create IAM role for Kendra with trust policy for Kendra service
    trust_policy = {
        "Version": "2012-10-17",
        "Statement": [{
            "Effect": "Allow",
            "Principal": {"Service": "kendra.amazonaws.com"},
            "Action": "sts:AssumeRole"
        }]
    }
    role_resp = iam.create_role(
        RoleName=kendra_role_name,
        AssumeRolePolicyDocument=json.dumps(trust_policy),
        Description="Role for Kendra index to access S3 data"
    )
    # Attach a policy for Kendra to read S3 and write CloudWatch logs
    iam.attach_role_policy(RoleName=kendra_role_name, PolicyArn="arn:aws:iam::aws:policy/AmazonKendraReadOnlyAccess")
    iam.attach_role_policy(RoleName=kendra_role_name, PolicyArn="arn:aws:iam::aws:policy/CloudWatchLogsFullAccess")
    kendra_role_arn = role_resp['Role']['Arn']
    print(f"Created Kendra IAM role: {kendra_role_arn}")
except Exception as e:
    kendra_role = iam.get_role(RoleName=kendra_role_name)
    kendra_role_arn = kendra_role['Role']['Arn']
    print(f"Kendra role already exists or failed to create, using: {kendra_role_arn}")
```

Now create the Kendra index itself:

```python
kendra = boto3.client('kendra', region_name=region)
kendra_index_name = f"CampaignKnowledgeBase-{suffix}"
try:
    index_resp = kendra.create_index(
        Name=kendra_index_name,
        RoleArn=kendra_role_arn,
        Edition="DEVELOPER_EDITION"  # small edition for demo
    )
    index_id = index_resp['Id']
    print(f"Creating Kendra index (ID: {index_id})...")
    # Wait for the index to be active
    waiter = kendra.get_waiter('index_created')
    waiter.wait(Id=index_id)
    print("Kendra index is active.")
except Exception as e:
    print(f"Error creating Kendra index: {e}")
    # If index exists or creation failed, you might retrieve existing index ID or skip
    index_id = None
```

If successful, we have a Kendra index ready. Now we add dummy documents to it. We’ll simulate two documents: one for Gen-Z marketing insights and one for Enterprise marketing. We can use the `BatchPutDocument` API to directly ingest documents into Kendra.

```python
if index_id:
    documents = [
        {
            "Id": "doc1",
            "Title": "GenZ Social Media Trends 2024",
            "Blob": b"Gen Z audiences heavily use TikTok, Instagram, and YouTube. Authenticity and social causes drive engagement. Best times to post are evenings. Past campaigns show that interactive challenges and meme-based content perform exceptionally well for Gen Z brand engagement.",
            "ContentType": "PLAIN_TEXT"
        },
        {
            "Id": "doc2",
            "Title": "B2B SaaS Lead Gen Best Practices",
            "Blob": b"Enterprise marketing (e.g., B2B SaaS) relies on LinkedIn, industry webinars, and whitepapers for lead generation. Emphasize ROI, case studies, and data security in messaging. Multi-touch campaigns with email nurturing yield the best conversion rates. Key metrics include MQLs, SQLs, and pipeline generated.",
            "ContentType": "PLAIN_TEXT"
        }
    ]
    try:
        put_resp = kendra.batch_put_document(IndexId=index_id, Documents=documents)
        print("Added dummy documents to Kendra index.")
    except Exception as e:
        print(f"Failed to add documents to Kendra: {e}")
```

Now our knowledge base has some content. Agents can query this Kendra index using the Query API. For example, Agent 1 (Interpreter) might call `kendra.query(IndexId=index_id, QueryText="Where do Gen Z audiences engage most?")` to retrieve relevant passages (in our case, it would find TikTok/Instagram from doc1). The **Kendra Retriever** will return the top matching passage, which the agent can include in its reasoning. This is an example of Retrieval-Augmented Generation (RAG) to ground the model’s output in real data.

*(If you prefer to use the Amazon Bedrock Knowledge Base with OpenSearch Serverless, the steps would involve creating a vector index and using Bedrock’s knowledge base APIs. Those are beyond this scope, but conceptually similar – documents would be ingested into the vector store and Bedrock would retrieve them when prompted.)*

### 1.4 Create SNS Topic for Notifications

We create an SNS topic to broadcast notifications when campaigns are processed. We’ll call it `CampaignCompletionTopic`. We can (optionally) subscribe an email for demonstration – note that if your AWS account is in SNS sandbox, you must confirm the subscription.

```python
sns = boto3.client('sns', region_name=region)
topic_name = f"CampaignCompletionTopic-{suffix}"
topic_resp = sns.create_topic(Name=topic_name)
topic_arn = topic_resp['TopicArn']
print(f"Created SNS topic: {topic_arn}")
# (Optional) subscribe an email endpoint for notifications
# email = "your-email@example.com"
# sns.subscribe(TopicArn=topic_arn, Protocol='email', Endpoint=email)
```

We will use this `topic_arn` in our Step Function to send a notification when the workflow completes or if it fails. For example, a message like “✅ Campaign GenZ Social Hype 2025 has been successfully planned!” or an error alert.

### 1.5 Create IAM Roles for Lambda and Step Functions

Now we set up IAM roles with appropriate policies:

* **Lambda Execution Role:** One role that all agent Lambdas can use (or separate roles per Lambda for stricter separation). It needs: permission to call Bedrock (InvokeModel and possibly access Bedrock knowledge bases), to read/write our S3 buckets, to read/write the DynamoDB table, to query Kendra, and to write CloudWatch logs. We’ll create one role and attach managed policies and custom inline policies for specific resources.
* **Step Functions Role:** Allows Step Functions to invoke the Lambda functions and publish to SNS. We’ll allow `states:StartExecution` as needed (if Step Functions triggers itself, not in our case) and more importantly `lambda:InvokeFunction` on our Lambdas and `sns:Publish` on our topic.
* (We already made a Kendra role earlier for the index; no additional roles needed for Kendra usage from Lambdas beyond permissions we’ll include.)

Let’s create the Lambda role:

```python
lambda_role_name = f"CampaignAgentLambdaRole-{suffix}"
lambda_assume_role_policy = {
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Principal": {"Service": "lambda.amazonaws.com"},
        "Action": "sts:AssumeRole"
    }]
}
try:
    role = iam.create_role(RoleName=lambda_role_name,
                           AssumeRolePolicyDocument=json.dumps(lambda_assume_role_policy),
                           Description="Lambda execution role for AI Campaign agents")
    lambda_role_arn = role['Role']['Arn']
    # Attach AWS managed policies for basic Lambda execution and Bedrock access
    iam.attach_role_policy(RoleName=lambda_role_name, PolicyArn="arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole")  # allows CloudWatch Logs
    # There isn't a public managed policy for Bedrock yet; use full managed one if exists or inline:
    bedrock_policy = {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "bedrock:InvokeModel",
                    "bedrock:InvokeModelWithResponseStream",
                    "bedrock:InvokeAgent"  # if using Agents/knowledge bases
                ],
                "Resource": "*"
            }
        ]
    }
    iam.put_role_policy(RoleName=lambda_role_name, PolicyName="BedrockInvokePolicy", PolicyDocument=json.dumps(bedrock_policy))
    # Inline policy for S3, DynamoDB, Kendra, SNS if needed:
    resource_policy = {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "s3:GetObject", "s3:ListBucket", "s3:PutObject"
                ],
                "Resource": [
                    f"arn:aws:s3:::{briefs_bucket}", f"arn:aws:s3:::{briefs_bucket}/*",
                    f"arn:aws:s3:::{assets_bucket}", f"arn:aws:s3:::{assets_bucket}/*"
                ]
            },
            {
                "Effect": "Allow",
                "Action": [
                    "dynamodb:PutItem", "dynamodb:GetItem", "dynamodb:UpdateItem", "dynamodb:Scan", "dynamodb:Query"
                ],
                "Resource": f"arn:aws:dynamodb:{region}:{sns.get_caller_identity()['Account']}:table/{table_name}"
            },
            {
                "Effect": "Allow",
                "Action": [
                    "kendra:Query"
                ],
                "Resource": "*" if not index_id else f"arn:aws:kendra:{region}:*:index/{index_id}"
            },
            {
                "Effect": "Allow",
                "Action": [
                    "sns:Publish"
                ],
                "Resource": topic_arn
            }
        ]
    }
    iam.put_role_policy(RoleName=lambda_role_name, PolicyName="ResourceAccessPolicy", PolicyDocument=json.dumps(resource_policy))
    print(f"IAM role created for Lambda: {lambda_role_arn}")
except Exception as e:
    lambda_role = iam.get_role(RoleName=lambda_role_name)
    lambda_role_arn = lambda_role['Role']['Arn']
    print(f"Using existing Lambda role: {lambda_role_arn}")
```

We created a role with CloudWatch log access, full Bedrock invoke access, and an inline policy restricting S3 and DynamoDB to our specific resources (note: for simplicity, we allowed DynamoDB on the whole table ARN and Kendra Query on all or specific index). Adjust as needed for least privilege (e.g. if you know the account ID, fill it in the ARN for Dynamo). Also included SNS\:Publish so Lambdas could directly send notifications if we choose (though we plan to let Step Functions handle SNS).

Now Step Functions role:

```python
sf_role_name = f"StepFunctionRole-{suffix}"
sf_assume_role_policy = {
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Principal": {"Service": "states.amazonaws.com"},
        "Action": "sts:AssumeRole"
    }]
}
try:
    sf_role = iam.create_role(RoleName=sf_role_name,
                              AssumeRolePolicyDocument=json.dumps(sf_assume_role_policy),
                              Description="Step Functions execution role for AI Campaign workflow")
    sf_role_arn = sf_role['Role']['Arn']
    # Policy for Step Functions to invoke Lambdas and publish to SNS
    sf_policy = {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow", 
                "Action": "lambda:InvokeFunction", 
                "Resource": "*"  # you can restrict to specific Lambda ARNs once we create them
            },
            {
                "Effect": "Allow",
                "Action": "sns:Publish",
                "Resource": topic_arn
            },
            {
                "Effect": "Allow",
                "Action": "dynamodb:UpdateItem",
                "Resource": f"arn:aws:dynamodb:{region}:*:table/{table_name}"
            }
        ]
    }
    iam.put_role_policy(RoleName=sf_role_name, PolicyName="StepFuncInvokePolicy", PolicyDocument=json.dumps(sf_policy))
    print(f"IAM role created for Step Functions: {sf_role_arn}")
except Exception as e:
    sf_role = iam.get_role(RoleName=sf_role_name)
    sf_role_arn = sf_role['Role']['Arn']
    print(f"Using existing StepFunctions role: {sf_role_arn}")
```

We allow Step Functions to invoke any Lambda (or we will attach specific ARNs after Lambdas are created), to publish to our SNS topic, and to update DynamoDB (perhaps updating an item that marks overall campaign status as “completed” – we’ll include that as a step).

With IAM roles ready, we proceed to create the Lambda functions.

### 1.6 Deploy Lambda Functions for Each Agent

We will create six Lambda functions corresponding to Agent0 through Agent5. For clarity and modularity, we will write their code inline and then create each Lambda with that code (zipped in-memory for deployment). Each Lambda will follow a similar pattern:

* Parse input (from event or fetch from Dynamo/S3 as needed).
* Invoke an Amazon Bedrock model (Claude or Titan) with an appropriate prompt to perform its task.
* Optionally call Kendra if needed for that agent (for example, Agent1 might do a Kendra query).
* Return the result and also store it to DynamoDB for persistence.
* Handle exceptions and log errors, possibly writing an error status to DynamoDB or raising to let Step Functions catch it.

We will use **Anthropic Claude v3** for more reasoning-intensive tasks (like Business Interpreter, Campaign Planner, Evaluator) and **Amazon Titan Text** for content generation tasks, just as an example of using multiple models. (In practice, you could use whichever model suits the task best – Claude might excel at longer coherent reasoning, Titan might be good at formatted outputs, etc.)

Let’s define the Lambda code for each agent as Python strings. We will keep them simple due to space, but in a real scenario you’d include more robust logic and formatting.

```python
import zipfile, io

# Helper to create a deployment package with given code
def create_lambda_zip(code_str):
    zip_buffer = io.BytesIO()
    with zipfile.ZipFile(zip_buffer, 'w') as z:
        z.writestr('lambda_function.py', code_str)
    zip_buffer.seek(0)
    return zip_buffer.read()

# Common Bedrock model IDs (adjust if needed to actual available IDs)
CLAUDE_MODEL = "anthropic.claude-v2"  # placeholder for Claude v3 if available
TITAN_MODEL = "amazon.titan-text-express-v1"

# Code template for Lambdas
# Note: We insert appropriate model_id and agent-specific logic below.
lambda_client = boto3.client('lambda', region_name=region)

# Agent 0: Input Extractor
agent0_code = r"""
import boto3, json, os, logging
logging.basicConfig(level=logging.INFO)
bedrock = boto3.client('bedrock-runtime')
dynamodb = boto3.client('dynamodb')
s3 = boto3.client('s3')
def lambda_handler(event, context):
    # Expect event to have S3 bucket/key of the brief or direct brief content
    try:
        bucket = event.get('brief_bucket')
        key = event.get('brief_key')
        if bucket and key:
            obj = s3.get_object(Bucket=bucket, Key=key)
            brief_text = obj['Body'].read().decode('utf-8')
            brief = json.loads(brief_text) if brief_text.strip().startswith('{') else {"raw_text": brief_text}
        else:
            # if direct content provided
            brief = event.get('brief', {})
        campaign_id = brief.get('campaign_id', 'UNKNOWN')
        # Construct prompt for Claude to extract important info (in JSON) from the brief
        prompt = f"Extract the key marketing information from the following brief and return as JSON:\n{json.dumps(brief)}"
        model_id = "{claude_model}"
        payload = {{ "inputText": prompt, "textGenerationConfig": {{"maxTokenCount": 500}} }}
        response = bedrock.invoke_model(modelId=model_id, body=json.dumps(payload))
        result = json.loads(response['body'].read())  # Assuming model returns JSON text
        # Store result in DynamoDB
        dynamodb.put_item(
            TableName=os.environ['DYNAMO_TABLE'],
            Item={{ "campaign_id": {{ "S": campaign_id }}, "step": {{ "S": "input_extracted" }}, "data": {{ "S": json.dumps(result) }} }}
        )
        return {{ "campaign_id": campaign_id, "extracted": result }}
    except Exception as e:
        logging.error(f"Error in Agent0 extraction: {{e}}")
        raise
""".replace("{claude_model}", CLAUDE_MODEL)

# Agent 1: Business Interpreter
agent1_code = r"""
import boto3, json, os, logging
logging.basicConfig(level=logging.INFO)
bedrock = boto3.client('bedrock-runtime')
dynamodb = boto3.client('dynamodb')
kendra = boto3.client('kendra')
def lambda_handler(event, context):
    try:
        campaign_id = event.get('campaign_id')
        # Retrieve extracted input from DynamoDB
        resp = dynamodb.get_item(TableName=os.environ['DYNAMO_TABLE'], Key={ "campaign_id": {"S": campaign_id}, "step": {"S": "input_extracted"} })
        brief_data = {}
        if 'Item' in resp:
            brief_data = json.loads(resp['Item']['data']['S'])
        # Use Kendra to get any relevant info for the target audience or product
        audience = brief_data.get('target_audience', '') or brief_data.get('target_segment', '')
        kendra_info = ""
        if audience:
            query_text = f"{audience} marketing insights"
            try:
                qres = kendra.query(IndexId=os.environ['KENDRA_INDEX'], QueryText=query_text)
                # take top result text
                if 'ResultItems' in qres and qres['ResultItems']:
                    passage = qres['ResultItems'][0].get('DocumentExcerpt', {}).get('Text', '')
                    kendra_info = f"Here are some insights: {passage}"
            except Exception as ke:
                logging.warning(f"Kendra query failed: {ke}")
        # Construct prompt for Claude to interpret business context
        business_prompt = "We have the following campaign brief data: " + json.dumps(brief_data) + ".\n"
        if kendra_info:
            business_prompt += kendra_info + "\n"
        business_prompt += "Interpret the business objectives and marketing context in a few sentences, considering the target audience and industry."
        model_id = "{claude_model}"
        payload = { "inputText": business_prompt, "textGenerationConfig": {"maxTokenCount": 300} }
        response = bedrock.invoke_model(modelId=model_id, body=json.dumps(payload))
        interpretation = json.loads(response['body'].read())  # assume output is JSON or plain text we can wrap
        if isinstance(interpretation, dict) and 'results' in interpretation:
            interpretation_text = interpretation['results'][0].get('outputText', '')
        else:
            interpretation_text = str(interpretation)
        # Store interpretation in DynamoDB
        dynamodb.put_item(
            TableName=os.environ['DYNAMO_TABLE'],
            Item={ "campaign_id": {"S": campaign_id}, "step": {"S": "business_context"}, "data": {"S": interpretation_text} }
        )
        return { "campaign_id": campaign_id, "interpretation": interpretation_text }
    except Exception as e:
        logging.error(f"Error in Agent1 interpretation: {e}")
        raise
""".replace("{claude_model}", CLAUDE_MODEL)

# Agent 2: Campaign Planner
agent2_code = r"""
import boto3, json, os, logging
logging.basicConfig(level=logging.INFO)
bedrock = boto3.client('bedrock-runtime')
dynamodb = boto3.client('dynamodb')
def lambda_handler(event, context):
    try:
        campaign_id = event.get('campaign_id')
        # Get business interpretation and original brief from Dynamo
        interp_resp = dynamodb.get_item(TableName=os.environ['DYNAMO_TABLE'], Key={"campaign_id": {"S": campaign_id}, "step": {"S": "business_context"}})
        brief_resp = dynamodb.get_item(TableName=os.environ['DYNAMO_TABLE'], Key={"campaign_id": {"S": campaign_id}, "step": {"S": "input_extracted"}})
        interpretation = interp_resp.get('Item', {}).get('data', {}).get('S', '')
        brief_data = {}
        if 'Item' in brief_resp:
            brief_data = json.loads(brief_resp['Item']['data']['S'])
        audience = brief_data.get('target_audience', '') or brief_data.get('target_segment', '')
        goal = "brand engagement" if audience and audience.lower().startswith("gen") else "lead generation"
        # Prompt Titan to produce a campaign plan
        plan_prompt = f"Create a marketing campaign plan for the following scenario:\nBrief: {json.dumps(brief_data)}\nContext: {interpretation}\nThe campaign is targeting {audience} with a goal of {goal}. Include: objectives, key messaging pillars, chosen channels, timeline, and success metrics."
        model_id = "{titan_model}"
        payload = {"inputText": plan_prompt, "textGenerationConfig": {"maxTokenCount": 500}}
        response = bedrock.invoke_model(modelId=model_id, body=json.dumps(payload))
        result = json.loads(response['body'].read())
        plan_text = result.get('results', [{}])[0].get('outputText', '') if 'results' in result else str(result)
        # Store plan
        dynamodb.put_item(
            TableName=os.environ['DYNAMO_TABLE'],
            Item={"campaign_id": {"S": campaign_id}, "step": {"S": "campaign_plan"}, "data": {"S": plan_text}}
        )
        return {"campaign_id": campaign_id, "plan": plan_text}
    except Exception as e:
        logging.error(f"Error in Agent2 planning: {e}")
        raise
""".replace("{titan_model}", TITAN_MODEL)

# Agent 3: Content Generator
agent3_code = r"""
import boto3, json, os, logging
logging.basicConfig(level=logging.INFO)
bedrock = boto3.client('bedrock-runtime')
dynamodb = boto3.client('dynamodb')
s3 = boto3.client('s3')
def lambda_handler(event, context):
    try:
        campaign_id = event.get('campaign_id')
        # Get campaign plan and brief data
        plan_resp = dynamodb.get_item(TableName=os.environ['DYNAMO_TABLE'], Key={"campaign_id": {"S": campaign_id}, "step": {"S": "campaign_plan"}})
        brief_resp = dynamodb.get_item(TableName=os.environ['DYNAMO_TABLE'], Key={"campaign_id": {"S": campaign_id}, "step": {"S": "input_extracted"}})
        plan_text = plan_resp.get('Item', {}).get('data', {}).get('S', '')
        brief_data = {}
        if 'Item' in brief_resp:
            brief_data = json.loads(brief_resp['Item']['data']['S'])
        audience = brief_data.get('target_audience', '').lower()
        # Choose style based on audience
        tone = "casual, trendy, youthful voice with emojis and slang" if "gen-z" in audience.lower() else "professional, authoritative tone"
        content_prompt = f"Based on the campaign plan: '''{plan_text}''', generate a set of 3 short content pieces (e.g. social media posts) in a {tone}. Ensure the content aligns with the key messages and objectives."
        model_id = "{titan_model}"
        payload = {"inputText": content_prompt, "textGenerationConfig": {"maxTokenCount": 400}}
        response = bedrock.invoke_model(modelId=model_id, body=json.dumps(payload))
        result = json.loads(response['body'].read())
        content = result.get('results', [{}])[0].get('outputText', '') if 'results' in result else str(result)
        # Store content in Dynamo and also save to S3 as a file
        dynamodb.put_item(
            TableName=os.environ['DYNAMO_TABLE'],
            Item={"campaign_id": {"S": campaign_id}, "step": {"S": "content_samples"}, "data": {"S": content}}
        )
        s3.put_object(Bucket=os.environ['ASSETS_BUCKET'], Key=f"{campaign_id}_content.txt", Body=content)
        return {"campaign_id": campaign_id, "content": content}
    except Exception as e:
        logging.error(f"Error in Agent3 content generation: {e}")
        raise
""".replace("{titan_model}", TITAN_MODEL)

# Agent 4: Channel Strategist
agent4_code = r"""
import boto3, json, os, logging
logging.basicConfig(level=logging.INFO)
bedrock = boto3.client('bedrock-runtime')
dynamodb = boto3.client('dynamodb')
def lambda_handler(event, context):
    try:
        campaign_id = event.get('campaign_id')
        plan_resp = dynamodb.get_item(TableName=os.environ['DYNAMO_TABLE'], Key={"campaign_id": {"S": campaign_id}, "step": {"S": "campaign_plan"}})
        brief_resp = dynamodb.get_item(TableName=os.environ['DYNAMO_TABLE'], Key={"campaign_id": {"S": campaign_id}, "step": {"S": "input_extracted"}})
        plan = plan_resp.get('Item', {}).get('data', {}).get('S', '')
        brief_data = {}
        if 'Item' in brief_resp:
            brief_data = json.loads(brief_resp['Item']['data']['S'])
        audience = brief_data.get('target_audience', '')
        # Ask Claude to suggest channel strategy
        prompt = f"Campaign Plan: {plan}\nNow, as a marketing channel strategist, list the best channels and scheduling for this campaign targeting {audience}. Provide reasoning for each channel choice."
        model_id = "{claude_model}"
        payload = {"inputText": prompt, "textGenerationConfig": {"maxTokenCount": 300}}
        response = bedrock.invoke_model(modelId=model_id, body=json.dumps(payload))
        result = json.loads(response['body'].read())
        strategy = result.get('results', [{}])[0].get('outputText', '') if 'results' in result else str(result)
        dynamodb.put_item(
            TableName=os.environ['DYNAMO_TABLE'],
            Item={"campaign_id": {"S": campaign_id}, "step": {"S": "channel_strategy"}, "data": {"S": strategy}}
        )
        return {"campaign_id": campaign_id, "strategy": strategy}
    except Exception as e:
        logging.error(f"Error in Agent4 channel strategy: {e}")
        raise
""".replace("{claude_model}", CLAUDE_MODEL)

# Agent 5: Evaluator
agent5_code = r"""
import boto3, json, os, logging
logging.basicConfig(level=logging.INFO)
bedrock = boto3.client('bedrock-runtime')
dynamodb = boto3.client('dynamodb')
sns = boto3.client('sns')
def lambda_handler(event, context):
    try:
        campaign_id = event.get('campaign_id')
        # Gather plan, content, strategy from DynamoDB
        plan = dynamodb.get_item(TableName=os.environ['DYNAMO_TABLE'], Key={"campaign_id": {"S": campaign_id}, "step": {"S": "campaign_plan"}}).get('Item', {}).get('data', {}).get('S', '')
        content = dynamodb.get_item(TableName=os.environ['DYNAMO_TABLE'], Key={"campaign_id": {"S": campaign_id}, "step": {"S": "content_samples"}}).get('Item', {}).get('data', {}).get('S', '')
        strategy = dynamodb.get_item(TableName=os.environ['DYNAMO_TABLE'], Key={"campaign_id": {"S": campaign_id}, "step": {"S": "channel_strategy"}}).get('Item', {}).get('data', {}).get('S', '')
        prompt = f"Evaluate the proposed campaign plan, content, and channel strategy. Plan: {plan} Content: {content} Channels: {strategy}. Provide a brief evaluation of how well this meets the objectives and suggest any improvements, and list key KPIs to track (e.g., impressions, clicks, leads)."
        model_id = "{claude_model}"
        payload = {"inputText": prompt, "textGenerationConfig": {"maxTokenCount": 400}}
        response = bedrock.invoke_model(modelId=model_id, body=json.dumps(payload))
        result = json.loads(response['body'].read())
        evaluation = result.get('results', [{}])[0].get('outputText', '') if 'results' in result else str(result)
        # Store final evaluation
        dynamodb.put_item(
            TableName=os.environ['DYNAMO_TABLE'],
            Item={"campaign_id": {"S": campaign_id}, "step": {"S": "evaluation"}, "data": {"S": evaluation}}
        )
        # Publish notification via SNS with summary
        message = f"Campaign {campaign_id} planning complete. Summary: {evaluation[:200]}..."  # first 200 chars
        sns.publish(TopicArn=os.environ['SNS_TOPIC'], Message=message, Subject="Campaign Plan Complete")
        return {"campaign_id": campaign_id, "evaluation": evaluation}
    except Exception as e:
        logging.error(f"Error in Agent5 evaluation: {e}")
        # Publish a failure notice
        sns.publish(TopicArn=os.environ['SNS_TOPIC'], Message=f"Campaign {campaign_id} failed at evaluation step: {e}", Subject="Campaign Plan Failed")
        raise
""".replace("{claude_model}", CLAUDE_MODEL)

# Package and create each Lambda function
agent_codes = [agent0_code, agent1_code, agent2_code, agent3_code, agent4_code, agent5_code]
for i, code in enumerate(agent_codes):
    func_name = f"CampaignAgent{i}-{suffix}"
    zip_bytes = create_lambda_zip(code)
    try:
        resp = lambda_client.create_function(
            FunctionName=func_name,
            Runtime='python3.9',
            Role=lambda_role_arn,
            Handler='lambda_function.lambda_handler',
            Code={'ZipFile': zip_bytes},
            Timeout=60,
            MemorySize=256,
            Environment={'Variables': {
                'DYNAMO_TABLE': table_name,
                'ASSETS_BUCKET': assets_bucket,
                'KENDRA_INDEX': index_id or "",
                'SNS_TOPIC': topic_arn
            }}
        )
        print(f"Created Lambda: {func_name}")
    except Exception as e:
        print(f"Error creating Lambda {func_name}: {e}")
```

We iterated through each agent’s code, created a deployment package (a simple ZIP with `lambda_function.py`), and created the Lambda function. Key points in this code:

* Each Lambda has environment variables configured for the DynamoDB table name, S3 bucket, Kendra index, and SNS topic ARN so that these resources can be referenced internally.
* The Lambdas use the Bedrock runtime client to call the model. We send prompts as `inputText` to either Claude or Titan. We assume the model returns a JSON structure with an `outputText` (which is true for Bedrock’s standardized response format). In some cases, we attempt `json.loads` on the response – depending on the model, we might need to adjust parsing (the code assumes Bedrock returns a JSON with `results[0].outputText`).
* Logging is enabled at INFO level in each Lambda for debugging (CloudWatch Logs will capture these).
* Error handling: We catch exceptions in each Lambda, log the error, and then re-raise so that Step Functions knows the execution failed. Agent5 also explicitly sends an SNS notification on failure.
* Agent0 reads the brief from S3 (or from event directly), uses Claude to parse it, and stores the extracted JSON.
* Agent1 retrieves that data and uses both Kendra (for audience insights) and Claude to produce a business interpretation.
* Agent2 uses Titan to generate a structured campaign plan (we prompt for objectives, pillars, channels, timeline, metrics).
* Agent3 uses Titan to generate sample content, adjusting tone by audience, and saves it to S3 (in addition to Dynamo).
* Agent4 uses Claude to recommend channel strategy with rationale.
* Agent5 uses Claude to evaluate everything and sends out an SNS notification with a short summary.

At this point, all Lambda functions should be deployed. You can verify by listing them or checking in AWS console. If any function failed to create (e.g. code size issues or role issues), ensure the IAM role is correct and maybe reduce code size if needed (embedding all code as above is purely for demonstration; in practice you’d package dependencies properly or use layers).

### 1.7 Define Step Functions State Machine

Finally, we create the Step Functions state machine that ties the agents together. We will use the Amazon States Language (JSON-based) to define a workflow where each state invokes one of our Lambda functions. We also include error handling (Catch clauses) so that if any agent fails, we send a failure notification and stop the execution gracefully.

Our state machine will look like:

```
Start -> Agent0 (Input Extractor) -> Agent1 (Interpreter) -> Agent2 (Planner)
      -> Agent3 (Content) -> Agent4 (Channels) -> Agent5 (Evaluator) -> End
```

We will pass along the `campaign_id` (and brief S3 location initially) as the input. Each Lambda returns the `campaign_id` plus its result, and we feed that into the next. (Step Functions can merge the result with the state – by default result replaces input, but we can use `ResultPath` or simply ensure each Lambda returns the campaign\_id to carry it forward).

We’ll include a Catch on each state to handle errors: on error, we can publish to SNS (but our Agent5 already does, and Lambdas on error would have partially done it). For demonstration, we’ll show a simple catch that moves to a Fail state or sends an SNS message. However, since our Lambdas themselves raise exceptions (triggering Step Functions error), it might be simpler to rely on Agent5 catching final issues.

Let’s build the state machine definition in JSON:

```python
import json
# Construct ARNs for each Lambda
account_id = boto3.client('sts').get_caller_identity().get('Account')
lambda_arns = [f"arn:aws:lambda:{region}:{account_id}:function:CampaignAgent{i}-{suffix}" for i in range(6)]

# Define the Step Functions state machine JSON
states = {}
for i in range(6):
    state_name = f"Agent{i}Step"
    lambda_arn = lambda_arns[i]
    is_last = (i == 5)
    states[state_name] = {
        "Type": "Task",
        "Resource": lambda_arn,
        "Parameters": {
            "campaign_id.$": "$.campaign_id"  # pass through the campaign_id from state input
        },
        "Retry": [
            {
                "ErrorEquals": ["Lambda.ServiceException","Lambda.AWSLambdaException","Lambda.SdkClientException"],
                "IntervalSeconds": 2,
                "MaxAttempts": 3,
                "BackoffRate": 2.0
            }
        ],
        "Catch": [
            {
                "ErrorEquals": ["States.ALL"],
                "ResultPath": "$.error",
                "Next": "FailureNotify"
            }
        ],
        "Next": f"Agent{i+1}Step" if not is_last else "NotifySuccess"
    }
# Add a step for final success notification (though Agent5 already does via SNS, we include a state as well for completeness)
states["NotifySuccess"] = {
    "Type": "Task",
    "Resource": "arn:aws:states:::aws-sdk:sns:publish",
    "Parameters": {
      "TopicArn": topic_arn,
      "Message.$": "States.Format('Campaign {} completed successfully.', $.campaign_id)",
      "Subject": "Campaign Plan Successful"
    },
    "End": True
}
# Step for failure catch
states["FailureNotify"] = {
    "Type": "Task",
    "Resource": "arn:aws:states:::aws-sdk:sns:publish",
    "Parameters": {
      "TopicArn": topic_arn,
      "Message.$": "States.Format('Campaign {} FAILED. Error: {}', $.campaign_id, $.error.Error)",
      "Subject": "Campaign Plan Failed"
    },
    "End": True
}

workflow_definition = {
    "Comment": "Multi-agent AI Campaign Planning state machine",
    "StartAt": "Agent0Step",
    "States": states
}

stepfunctions = boto3.client('stepfunctions', region_name=region)
sf_name = f"CampaignPlannerWorkflow-{suffix}"
try:
    sf_resp = stepfunctions.create_state_machine(
        name=sf_name,
        definition=json.dumps(workflow_definition),
        roleArn=sf_role_arn,
        type='STANDARD'
    )
    sf_arn = sf_resp['stateMachineArn']
    print(f"Created Step Functions State Machine: {sf_arn}")
except Exception as e:
    print(f"Error creating state machine: {e}")
    sf_arn = None
```

We built the state machine definition dynamically. Key aspects:

* We used the Lambda ARNs we created. Each state uses the `Resource` as the Lambda ARN (Step Functions will invoke it directly). We also provided a `Retry` block for transient Lambda errors (retry up to 3 times with backoff for known Lambda exceptions).
* We added a `Catch` on each agent state to handle any error (`States.ALL`). If an error occurs, it transitions to a `FailureNotify` state which uses the built-in AWS SDK integration to publish to SNS. That state sends a message containing the campaign\_id and error message. Then it ends the execution.
* After Agent5, we added a `NotifySuccess` step (also using direct SNS publish) to signal successful completion. In practice, Agent5 already publishes a detailed evaluation message, so this might be redundant – but it serves as a simple confirmation.
* We used the Step Functions intrinsic function `States.Format` to construct the message strings with the campaign\_id and error info.
* The Step Functions role we created is used (`sf_role_arn`), which must allow those Lambda invokes and SNS publish (which we set up).

Logging: By default, Step Functions records execution history, but you can also enable CloudWatch Logs for the state machine. For brevity, we did not specify a logging configuration here, but you could do so by adding a `loggingConfiguration` in the create\_state\_machine call. Step Functions will log each state transition and outcome, which aids debugging.

Now the orchestrator is ready.

## 2. Running the Multi-Agent Campaign Planner

With everything deployed, let’s execute a test run of the workflow using our dummy Gen-Z brief.

We will start the Step Function with input specifying our campaign ID and the S3 location of the brief. The state machine will then invoke each agent in turn. We can monitor progress in the AWS Step Functions console (it provides a visual state flow and step-by-step details) or via CloudWatch Logs.

```python
if sf_arn:
    # Input to execution: specify S3 bucket/key of the brief and the campaign_id
    start_input = {
        "campaign_id": campaign_brief["campaign_id"],
        "brief_bucket": briefs_bucket,
        "brief_key": brief_key
    }
    exec_resp = stepfunctions.start_execution(
        stateMachineArn=sf_arn,
        input=json.dumps(start_input),
        name=f"TestExecution-{suffix}"
    )
    execution_arn = exec_resp['executionArn']
    print(f"Started execution: {execution_arn}")
```

The Step Functions execution is now in progress. You can query its status:

```python
if sf_arn:
    history = stepfunctions.describe_execution(executionArn=execution_arn)
    print("Execution Status:", history['status'])
    # If succeeded, you can get the output:
    if 'output' in history:
        print("Execution output:", history['output'])
```

Since this is a demo with dummy data and we are not actually calling real Bedrock models here (unless you run this in an environment with Bedrock access), the actual response might not be meaningful. In a real run, you would inspect the output which would contain the final evaluation from Agent5 and possibly references to stored content.

**Monitoring:** During execution, each Lambda will log details to CloudWatch Logs (look for log groups `/aws/lambda/CampaignAgent0-...` etc.). Any errors or exceptions will appear there. The Step Functions console will show a *green* path if all went well or a *red* state where a failure happened. Thanks to our Catch blocks, even if a failure happens at, say, Agent3, the workflow will go to `FailureNotify`, send an SNS alert, and end gracefully.

**CloudWatch Integration:** We have the basic logging via CloudWatch for Lambdas and execution history. For deeper monitoring, you might set up CloudWatch Alarms on the SNS notifications (e.g., alert if a failure SNS message is sent), or create a dashboard that tracks the number of campaigns processed, their duration, etc. Step Functions automatically records metrics like execution count, fail/success count, and duration which you can view in CloudWatch. Our Lambdas also could emit custom metrics (not implemented here) such as number of content pieces generated or an estimated KPI. Those could be tracked over time.

## 3. Templates for Generative Outputs

It’s important to use the right prompting techniques to get the desired style of output for different audiences. In our Lambda code, we used simple prompt strings, but these can be templatized and stored externally for easier iteration. Here are examples of the prompt templates we used (which you can refine):

* **Gen-Z Content Template:** *“Based on the campaign plan: '''{plan}''', generate a set of 3 short social media posts in a casual, trendy, youthful voice with emojis and slang. Ensure the content aligns with the key messages and objectives.”* – This prompt instructs the model to produce upbeat, informal content. For example, it might output tweets or captions like: *“🚀 Ready for the #TechGadgetX challenge? Gen Z creators, show us your most creative unboxing! 🎉 #TechXLaunch”*. The use of emojis and slang (“🚀”, “🎉”) would appeal to Gen Z’s engagement style.
* **Enterprise Content Template:** *“Based on the campaign plan: '''{plan}''', write 3 short LinkedIn posts in a professional, authoritative tone, highlighting ROI and industry insights. Ensure the content aligns with key messages and objectives.”* – This yields a more formal output, e.g.: *“Tech Gadget X is revolutionizing the manufacturing floor – increasing productivity by 30%. 📈 Learn how this innovation drives ROI in our latest whitepaper. #B2BTech #Innovation”*. No slang, focuses on data and ROI.
* **Campaign Plan Template:** *“Create a marketing campaign plan for the following scenario:\nBrief: {brief\_data}\nContext: {interpretation}\nThe campaign is targeting {audience} with a goal of {goal}. Include: objectives, key messaging pillars, chosen channels, timeline, and success metrics.”* – This guided the Titan model to produce a structured plan. The output might be a few paragraphs or a JSON-like outline of objectives, messages, channels, etc. For Gen-Z, success metrics might be engagement rate or social shares, whereas for Enterprise, metrics might be number of leads or conversion rate.
* **Channel Strategy Template:** *“Campaign Plan: {plan}\nNow, as a marketing channel strategist, list the best channels and scheduling for this campaign targeting {audience}. Provide reasoning for each channel choice.”* – This prompt for Claude encourages an explanation. For Gen-Z, the answer might highlight TikTok daily challenges, Instagram Reels weekly, etc., with reasoning like “Gen-Z spends time on TikTok; short-form video will maximize engagement.” For enterprise, it might focus on LinkedIn weekly thought leadership posts or quarterly webinars, explaining that decision-maker engagement is higher on those channels.
* **Evaluation Template:** *“Evaluate the proposed campaign plan, content, and channel strategy... Provide a brief evaluation of how well this meets the objectives and suggest improvements, and list key KPIs to track (impressions, clicks, leads, etc.).”* – This prompt ensures the AI behaves like a marketing expert reviewing the plan. The output might say: *“The plan effectively targets Gen-Z through interactive social content, likely boosting brand awareness (KPI: social engagement rate). One improvement: increase presence on YouTube for longer-form content. Key KPIs: TikTok views, Instagram engagement, hashtag mentions.”* – or for enterprise: *“… effectively aligns with lead generation goals via LinkedIn and email nurturing. Ensure the messaging includes clear CTAs. Key KPIs: webinar sign-ups, MQLs generated, conversion rate.”*

By externalizing these templates (you could store them in S3 or Parameter Store), you make it easy to tweak the messaging style without changing code. The system can choose the appropriate template based on `target_audience` or a field in the brief.

## 4. Error Handling, Logging, and Security Considerations

**Error Handling:** We implemented retries for Lambdas and catch handlers in the workflow. If any agent fails (for example, Bedrock returns an error or a Lambda times out), the Step Functions execution will not abruptly hang – our `Catch` sends a failure notification and ends the run. Within Lambdas, we catch exceptions and log them. You could extend this by adding more granular checks (e.g., if Bedrock’s response is empty or not as expected, handle that separately). Also consider using Step Functions `Parallel` or `Map` states if you need to fan-out tasks (not needed here, but useful in other multi-agent designs).

**Logging & Monitoring:** Each Lambda has `logging` enabled; these logs go to CloudWatch Logs (in groups named after the Lambda). The Step Functions execution history records each state’s input and output (which can be viewed in the console or fetched via API). We also might create CloudWatch Alarms on certain metrics. For instance, if the SNS topic receives a “Campaign Plan Failed” message, we could trigger an alarm for developers to investigate. On success messages, we might trigger an automated email to the campaign owner (as we do via SNS). AWS CloudWatch Logs Insights can be used to search logs, for example to find all occurrences of a certain error or to track how long each agent took (each Lambda log has a duration reported).

**CloudWatch Dashboards:** In a production scenario, you might create a dashboard showing the number of campaigns processed per day (Step Functions metric for executions), average execution time, and perhaps custom business metrics like average engagement predicted. You can emit custom metrics from Lambdas (using CloudWatch `PutMetricData`) if needed.

**Security:** We used IAM roles to limit access. The Lambda role doesn’t allow anything beyond what’s necessary (invoking Bedrock, reading our specific S3 buckets and DynamoDB table, querying Kendra, publishing to SNS). The Step Functions role can only invoke our Lambdas and SNS. Always follow the principle of least privilege: if this were production, you’d refine the resource ARNs in the policies to only exactly the functions and resources needed (in our code we sometimes used `*` for simplicity when the exact ARN was not known upfront). Also, sensitive data: if the campaign briefs or outputs contain sensitive info, ensure the S3 buckets and Dynamo tables are encrypted and access-controlled. We didn’t explicitly set encryption here (AWS by default encrypts S3 and Dynamo at rest nowadays). Consider enabling KMS CMKs for S3 and Dynamo if needed.

**Clean Up:** If you’re running this as a demo, remember to delete resources to avoid ongoing costs: the Kendra index (especially in Developer Edition it’s minimal cost, but still), the Lambdas (incur minimal cost when not invoked), and Step Function, as well as the S3 buckets (and any uploaded objects) and DynamoDB table. The SNS topic (and any unused subscriptions) should also be cleaned up. You can do this manually or via boto3 calls at the end of the notebook. For example:

```python
# Cleanup example (uncomment to execute)
# stepfunctions.delete_state_machine(stateMachineArn=sf_arn)
# for fn in lambda_arns:
#     lambda_client.delete_function(FunctionName=fn.split(':')[-1])
# dynamodb.delete_table(TableName=table_name)
# sns.delete_topic(TopicArn=topic_arn)
# kendra.delete_index(Id=index_id)  # and wait for deletion
# iam.detach_role_policy(RoleName=lambda_role_name, PolicyArn="arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole")
# iam.delete_role(RoleName=lambda_role_name)
# iam.delete_role(RoleName=sf_role_name)
# (Also delete S3 buckets and their objects)
```
