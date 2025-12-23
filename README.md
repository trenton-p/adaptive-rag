# Adaptive-CAG Agent for Contexctual Retrieval

## Overview

This solution demonstrates how to build an intelligent, Agentic AI assistant that leverages **Adaptive-CAG (Contextual-Augmented Generation)** to semantically route user queries to the appropriate category namespace, retrieves contextually relevant articles, and generates accurate responses using a large language model.

The implementation is based on the use case from [Automated Machine Learning on AWS](https://www.packtpub.com/product/automated-machine-learning-on-aws/9781801811828), and extends it with [Adaptive-RAG](https://arxiv.org/pdf/2403.14403) patterns using [Pinecone](https://pinecone.io) for semantic classification and vector search.

## Table of Contents

- [Key Features](#key-features)
- [Architecture Overview](#architecture-overview)
- [How It Works](#how-it-works)
- [Project Structure](#project-structure)
- [Pre-requisites](#pre-requisites)
- [Configuration](#configuration)
- [Deployment](#deployment)
- [Testing the Solution](#testing-the-solution)
- [Using the Application](#using-the-application)
- [API Reference](#api-reference)
- [Cleanup](#cleanup)
- [Cost Considerations](#cost-considerations)
- [Troubleshooting](#troubleshooting)
- [License](#license)

## Key Features

- **Adaptive RAG**: Dynamically routes queries to category-specific namespaces (tech, world, sports, business) based on semantic similarity.
- **Contextual Retrieval**: Implements [Anthropic's contextual retrieval](https://www.anthropic.com/news/contextual-retrieval) approach for improved chunk relevance.
- **Real-time Streaming**: News events are ingested via Kinesis and processed in real-time.
- **Response Streaming**: Chat responses are streamed to the UI for better user experience.
- **Re-ranking**: Uses Pinecone's rerank model to improve retrieval quality.
- **Serverless Architecture**: Fully serverless using AWS Lambda, Kinesis, Glue, and CloudFront.

## Architecture Overview

![Architecture](./docs/architecture.jpg)

The solution consists of five main components:

### 1. Vector Database (`components/vector_db/`)
- Creates and manages a Pinecone serverless index.
- Stores API credentials in AWS Secrets Manager.
- Imports router classification data from S3.

### 2. Data Pipeline (`components/data_pipeline/`)
- **Kinesis Stream**: Ingests news events in real-time.
- **Glue ETL Job**: Processes streaming data, and uses the [shifting-left](https://www.confluent.io/learn/what-is-shift-left/) pattern to write to S3, in Apache Iceberg format.
- **Lambda Event Handler**: 
  - Chunks incoming articles.
  - Applies contextual retrieval to each chunk.
  - Generates embeddings using Pinecone Inference.
  - Semantically classifies articles to determine target namespace.
  - Upserts vectors to the appropriate Pinecone namespace.

### 3. News Agent (`components/agent/`)
- LangGraph-based workflow that implements Adaptive CAG.
- Routes user questions to the appropriate namespace using semantic classification.
- Retrieves relevant context from Pinecone with re-ranking.
- Generates responses using Amazon Bedrock (Claude).

### 4. Contact Form (`components/contact_form/`)
- HTTP API Gateway endpoint for contact form submissions.
- SNS topic for email notifications.

### 5. Static Website (`components/website/`)
- S3-hosted static website with CloudFront CDN.
- Integrated chatbot UI for interacting with the news agent.

## How It Works

### Semantic Routing (Adaptive CAG)

The solution uses a "router" namespace in Pinecone containing embeddings from the [AG News](https://huggingface.co/datasets/sh0416/ag_news) dataset. This dataset contains news headlines classified into four categories:

| Label | Category |
|-------|----------|
| 0 | World |
| 1 | Sports |
| 2 | Business |
| 3 | Tech |

When a user asks a question:

1. The question is embedded using Pinecone's `multilingual-e5-large` model.
2. The embedding is compared against the router namespace.
3. The top matches are re-ranked to determine the most relevant category.
4. The question is routed to the appropriate category namespace for retrieval.

### Contextual Retrieval

When news articles are ingested, each chunk is enriched with contextual information:

1. Articles are split into 512-character chunks with 100-character overlap.
2. For each chunk, an LLM generates a brief context describing how the chunk fits within the full article.
3. The context is prepended to the chunk before embedding.
4. This improves retrieval accuracy by providing additional semantic context.

### Agent Workflow
```
START
  │
  ▼
┌─────────────────┐
│ route_question  │ ─── Classify question to namespace
└─────────────────┘
  │
  ├── tech ──────► tech_retriever
  ├── world ─────► world_retriever
  ├── sports ────► sports_retriever
  └── business ──► business_retriever
                        │
                        ▼
                ┌───────────────┐
                │generate_answer│ ─── Generate response with context
                └───────────────┘
                        │
                        ▼
                       END
```

## Project Structure
```
.
├── app.py                      # CDK application entry point
├── constants.py                # Configuration variables
├── requirements.txt            # Python dependencies
├── cdk.json                    # CDK configuration
├── components/
│   ├── agent/                  # News agent component
│   │   ├── __init__.py         # CDK construct
│   │   └── runtime/
│   │       ├── app/
│   │       │   ├── agent.py    # LangGraph agent implementation
│   │       │   ├── main.py     # FastAPI application
│   │       │   └── utils.py    # Pinecone utilities
│   │       ├── Dockerfile
│   │       └── requirements.txt
│   ├── contact_form/           # Contact form API
│   │   ├── __init__.py
│   │   └── runtime/
│   │       └── index.py
│   ├── data_pipeline/          # Data ingestion pipeline
│   │   ├── __init__.py
│   │   ├── assets/             # Java SDK for Glue
│   │   ├── etl-scripts/        # Glue ETL scripts
│   │   └── event_handler/      # Lambda for vectorization
│   ├── vector_db/              # Pinecone index management
│   │   ├── __init__.py
│   │   └── index_handler/
│   └── website/                # Static website
│       ├── __init__.py
│       └── public/             # HTML, CSS, JS assets
├── docs/
│   └── architecture.jpg
└── test/
    ├── main.py                 # Test data ingestion script
    └── requirements.txt
```

## Pre-requisites

### 1. Pinecone Account

1. Navigate to https://pinecone.io and click **Sign Up**
2. After logging in, [create an API key](https://docs.pinecone.io/guides/projects/manage-api-keys#create-an-api-key)
3. Configure the [S3 Integration](https://docs.pinecone.io/guides/operations/integrations/integrate-with-amazon-s3) for bulk data import

### 2. AWS Account

Ensure the following are configured:

1. **Bedrock Model Access**: [Enable access](https://docs.aws.amazon.com/bedrock/latest/userguide/model-access-modify.html) to:
   - `anthropic.claude-3-haiku-20240307-v1:0` (text generation)
   - `cohere.embed-english-v3` (embeddings - optional, Pinecone inference is used by default)

2. **S3 Bucket**: [Create an S3 bucket](https://docs.aws.amazon.com/AmazonS3/latest/userguide/create-bucket-overview.html) for:
   - Router data import (`vector-exports/router/`)
   - ETL scripts and assets
   - Event data storage (Iceberg tables)

3. **Router Dataset**: Prepare the AG News dataset in parquet format:
```python
   from datasets import load_dataset
   
   # Load and prepare AG News dataset
   dataset = load_dataset("sh0416/ag_news", split="train")
   
   # Map labels to namespace names
   label_map = {0: "world", 1: "sports", 2: "business", 3: "tech"}
   
   # Transform and save as parquet
   # Include: id, text (headline), values (embedding), namespace (category)
```
   Upload to `s3://<bucket>/vector-exports/router/`

4. **Apache Iceberg Connector**: Subscribe to the [Apache Iceberg Connector for AWS Glue](https://aws.amazon.com/marketplace/pp/prodview-iicxofvpqvsio) in AWS Marketplace

### 3. Third-party Tools

| Tool | Version | Installation |
|------|---------|--------------|
| AWS CDK | >= 2.179.0 | `npm install -g aws-cdk` |
| Python | >= 3.10 | [python.org](https://python.org) |
| Node.js | >= 18 | [nodejs.org](https://nodejs.org) |
| Docker | Latest | Required for Lambda container builds |

## Configuration

Update `constants.py` with your values:
```python
# General
CONTACT_EMAIL="your-email@example.com"      # Receives contact form submissions
DATA_LAKE_BUCKET="your-bucket-name"         # S3 bucket for data storage

# Pinecone
PINECONE_API_KEY="your-pinecone-api-key"
PINECONE_REGION="us-east-1"                 # Pinecone serverless region
PINECONE_IMPORT_URI="s3://your-bucket/vector-exports"  # Router data location
PINECONE_INTEGRATION_ID="your-integration-id"  # From S3 integration setup
PINECONE_EMBEDDING_MODEL="multilingual-e5-large"
PINECONE_PROD_INDEX="acme-news-index"       # Name for your Pinecone index

# AWS
BEDROCK_EMBEDDING_MODEL="cohere.embed-english-v3"
BEDROCK_TEXT_MODEL="anthropic.claude-3-haiku-20240307-v1:0"
AWS_REGION="us-east-1"
```

## Deployment

### 1. Bootstrap CDK (first time only)
```bash
cdk bootstrap aws://<ACCOUNT_ID>/<REGION>
```

See [Bootstrap your environment](https://docs.aws.amazon.com/cdk/v2/guide/bootstrapping-env.html) for details.

### 2. Create Virtual Environment
```bash
python3 -m venv .venv
source .venv/bin/activate  # Linux/macOS
# or
.venv\Scripts\activate     # Windows
```

### 3. Install Dependencies
```bash
pip install -r requirements.txt
```

### 4. Synthesize CloudFormation Template
```bash
cdk synth
```

### 5. Deploy
```bash
cdk deploy
```

### Deployment Outputs

After deployment, note the following outputs:

| Output | Description |
|--------|-------------|
| `CloudFrontUrl` | URL of the deployed website |
| `GlueJobID` | ID of the streaming ETL job |
| `KinesisStreamName` | Name of the ingestion stream |
| `DataBucketName` | S3 bucket for data storage |
| `TopicArn` | SNS topic for contact notifications |

## Testing the Solution

### Ingest Sample News Data

Use the provided test script to ingest news articles:
```bash
cd test
pip install -r requirements.txt

python main.py \
  --count 100 \
  --stream <KinesisStreamName> \
  --region us-east-1
```

This script:
1. Downloads the CNN/DailyMail dataset
2. Formats articles with IDs, timestamps, summaries, and content
3. Sends records to the Kinesis stream for processing

**Note**: Allow 5-10 minutes for the data pipeline to process and vectorize the articles.

### Verify Data Ingestion

1. **Check Glue Job**: In the AWS Console, navigate to AWS Glue > Jobs and verify the `EventEtlJob` is running
2. **Check Pinecone**: In the Pinecone console, verify vectors appear in the category namespaces (tech, world, sports, business)
3. **Check S3**: Verify Iceberg tables are created in `s3://<bucket>/production-data/event-data/`

## Using the Application

1. Open the `CloudFrontUrl` in your browser
2. Click the chat icon in the bottom-right corner
3. Ask questions about news topics:
   - "What are the latest developments in artificial intelligence?"
   - "Tell me about recent sports championships"
   - "What's happening in the business world?"
   - "Any news about international politics?"

The agent will:
1. Route your question to the appropriate category
2. Retrieve relevant news articles
3. Generate a contextual response

## API Reference

### Chat Endpoint
```
POST /api/chat
Content-Type: application/json

{
  "question": "What's happening in tech?",
  "thread_id": "session-123"
}

Response: Streaming text/plain
```

### Contact Form Endpoint
```
POST /api/contact
Content-Type: application/json

{
  "email": "user@example.com",
  "question": "Your message here"
}

Response:
{
  "message": "Thank you! We've received your message..."
}
```

## Cleanup

To destroy all deployed resources:
```bash
cdk destroy
```

**Note**: The Pinecone index will be automatically deleted. Ensure you have backups if needed.

## Cost Considerations

This solution uses the following billable services:

| Service | Cost Factor |
|---------|-------------|
| **Pinecone** | Serverless pricing based on read/write units and storage |
| **Amazon Bedrock** | Per-token pricing for Claude 3 Haiku |
| **AWS Glue** | DPU-hours for the streaming ETL job |
| **Amazon Kinesis** | Shard hours and PUT payload units |
| **AWS Lambda** | Request count and duration |
| **Amazon CloudFront** | Data transfer and requests |
| **Amazon S3** | Storage and requests |

**Recommendations**:
- Stop the Glue job when not ingesting data
- Use smaller test datasets during development
- Monitor Pinecone usage in the dashboard
- Set up AWS Budget alerts

## Troubleshooting

### Common Issues

**Glue Job Fails to Start**
- Ensure the Apache Iceberg Connector is subscribed in AWS Marketplace
- Verify S3 bucket permissions
- Check Glue job logs in CloudWatch

**No Vectors in Pinecone**
- Verify the Lambda event handler is processing Kinesis records
- Check Lambda logs for embedding or upsert errors
- Ensure Pinecone API key is valid

**Chat Returns Empty Responses**
- Verify data has been ingested and vectorized
- Check the agent Lambda logs
- Ensure Bedrock model access is enabled

**Contact Form Not Working**
- Confirm email subscription in SNS
- Check the Lambda function logs
- Verify CORS settings if testing from localhost

## Contributing

Contributions are welcome! Please:

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Acknowledgments

- [Automated Machine Learning on AWS](https://www.packtpub.com/product/automated-machine-learning-on-aws/9781801811828) by Packt
- [Adaptive-RAG Paper](https://arxiv.org/pdf/2403.14403)
- [Anthropic Contextual Retrieval](https://www.anthropic.com/news/contextual-retrieval)