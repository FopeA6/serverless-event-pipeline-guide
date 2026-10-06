# Running a Serverless Event Pipeline Locally with LocalStack and SAM

A cleaned-up guide based on documentation I wrote for an event-driven banking accelerator at
Slalom. Client names, account identifiers, credentials and internal URLs have been removed.
It explains how the system fits together and how a new engineer can run it on their own machine.

## The system in one picture

Customer actions (a deposit, a declined card, a direct debit set-up) flow through three services
connected by an event bus:

1. **Event ingestion**: Amazon API Gateway receives HTTP requests and passes them to AWS Lambda
   handlers: get all items, get by ID, put item, delete by ID. Items are stored in DynamoDB. After a
   PUT, the handler publishes the event to the event bus. Ingestion adds metadata but does not
   process customer data.
2. **Customer metadata service**: an *action* Lambda receives the event, creates a customer event
   object and stores it in DynamoDB. An *audit* Lambda records whether the event is pending or
   processed in a separate audit table. A *publish* Lambda sends the updated metadata back to the bus.
3. **Customer engagement service**: an action Lambda creates an engagement object and publishes to
   an SNS topic. A downstream Lambda hands the customer to a contact-centre queue (Amazon Connect)
   or a messaging channel.
4. **Event publisher**: a Lambda triggered from the bus writes the raw event as JSON to an S3
   bucket, the data lake.

## Prerequisites

- Node.js, Docker, Git
- AWS SAM CLI and the `aws-sam-cli-local` wrapper
- Python 3 and a virtual environment

## Setup

```bash
python3 -m venv samcli_env
source samcli_env/bin/activate
pip install aws-sam-cli-local localstack

# Docker must be running first
localstack start -d
```

Point the AWS profile at LocalStack in `samconfig.toml`:

```toml
[default]
profile = "localstack"

[profile localstack]
region = us-east-1
output = json
endpoint_url = http://localhost:4566
```

## Build and deploy each service

From each service folder (ingestion, metadata, engagement, publisher):

```bash
sam build
samlocal deploy
```

## Troubleshooting

- **Deploy fails on an API base path mapping resource.** Remove that resource and anything that
  depends on it in `template.yaml`, make the same change in the built template under
  `.aws-sam/build/`, then destroy and redeploy.
- **A Lambda cannot reach a table or bucket after you rename it.** Update the IAM policy for each
  Lambda role as well as the name in the code. Renaming tables or buckets means changing both.
- **A Lambda writes no logs.** Create the CloudWatch log group the function expects.

## Notes

I wrote the original documentation and the architecture diagram as part of the team's work. The
commands here follow that documentation and I have not re-run them for this public version, so
check versions before relying on them.
