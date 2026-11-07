# Executive Summary

For a Gmail-facing AI assistant in 2026, the recommended architecture is a serverless, event-driven system on AWS, triggered by Google's native push notifications. This pattern is superior to continuous polling, as it is more scalable, efficient, and provides near-real-time responsiveness. The structure's foundation is the Gmail API's `watch` feature, which sends notifications to a Google Cloud Pub/Sub topic. These events are then pushed across clouds to an AWS API Gateway endpoint. State management for complex email triage workflows—such as receiving, forwarding, waiting for user instructions, and drafting replies—is best handled by AWS Step Functions, particularly its Standard Workflows, which are designed for long-running processes involving human-in-the-loop steps. State data, including the last processed `historyId` from Gmail, message statuses, and encrypted user OAuth2 refresh tokens, should be persisted in a database like Amazon DynamoDB. For AI integration with Claude on Amazon Bedrock, the optimal pattern is to use the Bedrock Responses API with server-side tool use, where actions like 'send email' or 'create draft' are implemented as discrete, secure Lambda functions. This approach is more robust and maintainable than using bash scripts or a monolithic Python sidecar. Thread correlation for replies is achieved by programmatically setting the `threadId`, matching the `Subject` header, and correctly populating the `In-Reply-To` and `References` headers in compliance with RFC 2822.

# Recommended Architecture Overview

The proposed architecture is a robust, serverless, and event-driven system primarily hosted on AWS, designed to react to events from a user's Gmail account. The system's components work in concert as follows:

1.  **Event Ingress (Google Cloud & AWS):** The process begins by setting a `watch` on the user's Gmail mailbox via the Gmail API. This directs Google to send push notifications for any changes to a specified Google Cloud Pub/Sub topic. The Pub/Sub topic is configured for authenticated push, sending a POST request containing a JSON payload to a secure HTTPS endpoint on AWS API Gateway. This payload includes a `message.data` field which is a base64url-encoded string containing the user's email address and the new mailbox `historyId`.

2.  **Asynchronous Processing & Decoupling (AWS):** The API Gateway endpoint is integrated with an AWS Lambda function. This initial Lambda's primary roles are to verify the OIDC authentication token sent by Pub/Sub to ensure the request is legitimate, and then to place the notification payload into an Amazon SQS (Simple Queue Service) queue. Using SQS decouples the ingestion of events from their processing, providing durability and resilience against downstream failures.

3.  **Workflow Orchestration (AWS):** AWS Step Functions serves as the brain of the operation, orchestrating the entire email triage workflow. A Standard Workflow is triggered by messages appearing in the SQS queue. This state machine manages the sequence of tasks, including fetching email details, invoking the AI model, waiting for human input, and executing actions like drafting a reply.

4.  **State Management (AWS):** Amazon DynamoDB is used as the persistent state store. It holds several key pieces of information across different tables: a `Users` table for storing encrypted OAuth2 refresh tokens, a `Mailboxes` table to track the `lastHistoryId` processed for each user and the `watch` expiration time, and a `Messages` table to log the status of each email (e.g., `received`, `forwarded`, `drafted`) and prevent duplicate processing.

5.  **AI and Tool Integration (AWS):** The core AI logic is handled by Amazon Bedrock, running a model like Claude. Within the Step Functions workflow, a task state invokes Bedrock's Responses API. The model is provided with the email content and a list of available server-side tools. These tools are implemented as individual AWS Lambda functions that encapsulate specific actions, such as fetching a full email body, creating a draft via the Gmail API, or interacting with a calendar. This server-side tool-use pattern keeps the agent's capabilities modular, secure, and scalable.

# Architectural Diagram Description

This describes the data flow and component interactions from a new email's arrival to the creation of an AI-generated draft:

1.  **Origin (Gmail):** An email arrives in the user's inbox. The Gmail API, configured with a `watch`, detects this change.

2.  **Event Notification (Google Cloud):** The Gmail backend publishes a notification message to a designated Google Cloud Pub/Sub topic. This message contains the user's email address and a new `historyId`.

3.  **Cross-Cloud Bridge (Pub/Sub -> API Gateway):** The Pub/Sub topic pushes the notification via an authenticated HTTPS POST request to a predefined endpoint on AWS API Gateway.

4.  **Ingestion & Verification (API Gateway -> Lambda -> SQS):** The API Gateway triggers a 'Verifier' Lambda function. This function validates the OIDC token in the request header to authenticate the source as Google. Upon successful validation, it places the notification payload into an SQS queue for reliable, asynchronous processing.

5.  **Workflow Trigger (SQS -> Step Functions):** The SQS queue is configured as an event source for an AWS Step Functions state machine. The arrival of a new message in the queue initiates a new workflow execution.

6.  **State Sync & Data Fetching (Step Functions -> DynamoDB & Gmail API):** The first step of the workflow is a task that retrieves the user's last known `historyId` from a DynamoDB table. It then calls the Gmail API's `users.history.list` method to get all changes since that point. It iterates through the changes, and for each new message, it calls `users.messages.get` to fetch the full message content. It updates DynamoDB with the new `historyId` to mark the sync point.

7.  **AI Reasoning (Step Functions -> Bedrock):** The workflow passes the fetched email content and thread context to Amazon Bedrock. The model (Claude) analyzes the email and, based on its instructions, decides on a course of action, such as 'draft a reply'.

8.  **Action Execution (Bedrock -> Step Functions -> Lambda -> Gmail API):** The model's decision is returned to the Step Functions workflow. If the action is to draft a reply, the workflow invokes a dedicated 'Drafting' Lambda function. This Lambda constructs a valid MIME message, ensuring the `threadId`, `Subject`, `In-Reply-To`, and `References` headers are correctly set for proper threading. It then calls the `users.drafts.create` method of the Gmail API, using the user's stored OAuth token.

9.  **Final State Update (Lambda -> Step Functions -> DynamoDB):** The 'Drafting' Lambda returns a success status and the ID of the created draft to the Step Functions workflow. The workflow's final step is to update the message's status in the DynamoDB `Messages` table to 'drafted'. The workflow execution then completes successfully.

# Gmail Interaction Strategy

## Recommended Method

Utilize Google's push notification system by calling the Gmail API's `users.watch()` method to monitor mailbox changes. This setup directs notifications to a Google Cloud Pub/Sub topic, which then pushes a JSON payload to a specified HTTPS endpoint (e.g., an AWS API Gateway). The payload's `message.data` field is a base64url-encoded string containing the user's email address and the new mailbox `historyId`.

## Reasoning

This push-based approach is significantly more efficient and scalable than polling. It provides near-real-time updates on mailbox changes without repeatedly querying the API, thereby avoiding potential API quota issues. By using `history.list` only when a change notification is received, the application minimizes API calls and operates more cost-effectively.

## Key Api Calls

The primary API calls are `users.watch()` to establish the push notification channel and `users.history.list()` to retrieve all mailbox changes (new messages, label changes, etc.) that have occurred since the `startHistoryId` provided in the call. The `history.list` response will contain a list of history records, where messages typically only have `id` and `threadId` fields populated, requiring further API calls to fetch full details if needed.

## Alternative Method Critique

Server-side polling, such as repeatedly calling `users.messages.list`, is not recommended for a scalable solution. While it might be feasible for a single, low-volume personal inbox, it is inefficient, consumes API quota rapidly, and introduces latency between when an email arrives and when the application detects it. For an autonomous agent requiring timely responses, this delay and quota risk make polling an inferior choice.


# Authentication And Token Management

## Authorization Flow

The recommended authorization method is the OAuth 2.0 web server flow. This flow is designed for applications that run on a server and need to access Google APIs on behalf of a user, including when the user is offline.

## Required Access Type

The application must specify `access_type=offline` during the initial authorization request. This parameter instructs Google's authorization server to return a refresh token in addition to the access token. The refresh token is essential for maintaining long-term access without requiring the user to re-authenticate frequently.

## Token Storage Recommendation

The long-lived refresh token must be stored securely on the server. The recommended practice on AWS is to use a dedicated secret management service like AWS Secrets Manager or to store it in a DynamoDB table with envelope encryption, using AWS Key Management Service (KMS) to protect the data encryption keys.

## Token Refresh Process

The application uses the stored refresh token to make a request to Google's token endpoint to obtain a new, short-lived access token whenever the current one expires. The application must be built to handle potential `invalid_grant` errors, which indicate the refresh token is no longer valid (e.g., due to user revocation, password change, or inactivity for six months). In such cases, the application must trigger a re-consent flow to get a new refresh token from the user.


# Email Threading And Drafting Logic

## Thread Identification Method

To associate a new draft or message with an existing conversation, the `threadId` of that conversation must be specified in the `message` resource when calling methods like `users.drafts.create` or `users.messages.insert`.

## Required Rfc2822 Headers

The new message must contain `In-Reply-To` and `References` headers that are set in compliance with the RFC 2822 standard. The `In-Reply-To` header should contain the `Message-ID` of the message being replied to, and the `References` header should contain the `Message-ID`s of the previous messages in the thread.

## Subject Header Requirement

The `Subject` header of the new message or draft must exactly match the `Subject` headers of the other messages already in the target thread. Any deviation in the subject line will cause Gmail to create a new thread.


# Ai Agent Integration Pattern

## Recommended Service

Amazon Bedrock is the primary AWS service recommended for building the AI agent logic, specifically utilizing a model like Claude that supports tool use.

## Integration Method

The recommended integration method is server-side tool calling using the Amazon Bedrock Responses API. This feature allows the model to request the execution of a tool, and the application's backend code then executes it and returns the result to the model to inform its next step.

## Tool Implementation

The tools that the agent can use (e.g., for sending emails via the Gmail API, checking a calendar, etc.) should be implemented as custom AWS Lambda functions. This approach provides a serverless, scalable, and well-governed way to extend the agent's capabilities. An alternative mentioned is using the AgentCore Gateway as an integration type.

## Reasoning

This pattern is superior to alternatives like a Python sidecar service or bash scripts because it is the native, idiomatic way to build agents on Bedrock. Implementing tools as Lambda functions is more robust, secure, and scalable than managing bash scripts or a separate service. It keeps all tool execution within the governed AWS account, leverages a serverless architecture that handles scaling automatically, and avoids the complexities of managing inter-service communication and state associated with a sidecar pattern.


# Workflow State Management

## Recommended Service

AWS Step Functions is the best-suited service for orchestrating the multi-step email triage workflow, providing an explicit state machine to manage the process.

## Workflow Type

Step Functions Standard Workflows are recommended because they are designed for long-running processes (up to one year) and are ideal for workflows that require human intervention. For shorter, high-volume, idempotent steps within the main workflow, nesting an Express Workflow can be an effective pattern.

## Example Workflow States

A typical email triage workflow would include states such as: 'ReceiveEmail' (triggered from an SQS queue), 'TriagePriority' (using the Bedrock agent to classify), 'CheckForHumanApproval', 'WaitForInstruction' (pausing the workflow), 'GenerateDraftReply' (calling the Bedrock agent), 'SendDraft', and 'UpdateLabels'.

## Human In The Loop Pattern

The specific pattern for implementing human-in-the-loop steps is the `.waitForTaskToken` service integration. When the workflow reaches this state, it pauses and provides a unique task token. This token can be sent to a human via SNS or a UI, and the workflow will only resume once it receives a `SendTaskSuccess` or `SendTaskFailure` API call containing that token. It is a best practice to use heartbeats and timeouts with this pattern to prevent workflows from being stuck indefinitely.


# Core Architecture Components

## Component Name

Gmail API Push Notifications + Google Cloud Pub/Sub

## Provider

Google Cloud

## Role In Architecture

This is the event source for the entire pipeline. It uses the Gmail API's `watch()` method to subscribe to changes in a user's mailbox. Instead of polling, Google's recommended pattern is to have the Gmail API send push notifications to a Google Cloud Pub/Sub topic. These notifications are near-real-time and contain a base64url-encoded JSON payload with the user's email address and a new `historyId`. This `historyId` is then used to efficiently fetch only the changes since the last notification, avoiding costly and quota-intensive full mailbox scans.

## Component Name

AWS API Gateway

## Provider

AWS

## Role In Architecture

Serves as the secure, public-facing HTTPS endpoint that receives the push notifications from Google Cloud Pub/Sub. It is configured to accept POST requests from Google and is responsible for triggering an AWS Lambda function to process the incoming notification. This provides a serverless, scalable entry point into the AWS environment.

## Component Name

AWS Lambda (Ingress Verifier)

## Provider

AWS

## Role In Architecture

This function is triggered by API Gateway upon receiving a notification. Its first critical role is security: it verifies the OIDC (OpenID Connect) JSON Web Token (JWT) included in the authenticated push from Pub/Sub to ensure the request is authentic and originated from Google. After successful validation, it places the notification payload (containing the `historyId`) into an SQS queue for durable and asynchronous processing.

## Component Name

AWS SQS (Simple Queue Service)

## Provider

AWS

## Role In Architecture

Acts as a durable and scalable message queue that decouples the event ingress layer (API Gateway/Lambda) from the core processing logic. It buffers the incoming Gmail change notifications, ensuring that no events are lost if the downstream worker service is busy or temporarily unavailable. This enhances the overall resilience of the system.

## Component Name

Python Worker Service (on EC2/ECS/Fargate)

## Provider

AWS

## Role In Architecture

This is the main processing engine of the assistant. It can be deployed on EC2, ECS, or Fargate. The service continuously polls the SQS queue for new notifications. Upon receiving a message, it uses the stored OAuth2 refresh token to get a fresh access token, then calls the Gmail API's `users.history.list` method with the `historyId` from the notification to fetch message deltas. It then retrieves full message content, updates the state in DynamoDB, and initiates the Step Functions workflow for triage and response.

## Component Name

AWS DynamoDB

## Provider

AWS

## Role In Architecture

Serves as the high-performance, scalable NoSQL database for all state management. It would contain several tables: a `Users` table to store user-specific information including their encrypted Google OAuth refresh tokens; a `Mailboxes` table to track the `lastHistoryId` processed for each user and the expiration of their `watch` subscription; and a `Messages` table to track the state of each email (e.g., received, forwarded, awaiting_reply, sent) and ensure idempotency by logging processed message and history IDs.

## Component Name

AWS Step Functions (Standard Workflow)

## Provider

AWS

## Role In Architecture

Orchestrates the complex, multi-step email triage workflow. A Standard Workflow is used because the process can be long-running and involve human interaction. The workflow would define states for classifying the email, forwarding it if necessary, and pausing to wait for user instructions using the `.waitForTaskToken` integration pattern. Once instructions are received, it proceeds to generate a draft, wait for final approval, and trigger the send action.

## Component Name

Amazon Bedrock (with Claude)

## Provider

AWS

## Role In Architecture

This is the core intelligence of the AI assistant. The Claude model on Bedrock is invoked via the Responses API to perform natural language tasks such as summarizing email threads, classifying the intent of a message, and generating context-aware draft replies. The architecture leverages Bedrock's server-side tool use capability, allowing the model to securely call other AWS services (via Lambda functions) to perform actions.

## Component Name

AWS Lambda (as Bedrock Tools)

## Provider

AWS

## Role In Architecture

These functions serve as the 'tools' that the Claude model can invoke. They act as a secure and governed bridge between the LLM and external APIs. For example, there would be a Lambda tool for `send_email` which encapsulates the logic of composing the MIME message and calling the Gmail API, and another for `create_calendar_event`. This pattern is preferred over bash scripts as it is more robust, scalable, and secure.

## Component Name

AWS Secrets Manager / KMS

## Provider

AWS

## Role In Architecture

Provides secure storage for sensitive credentials, most importantly the per-user Google OAuth2 refresh tokens. Refresh tokens are long-lived and powerful, so they must be stored encrypted at rest. Secrets Manager, backed by AWS Key Management Service (KMS), is the ideal service for this, handling encryption, access control, and rotation policies.


# End To End Data Flow

Here is a step-by-step walkthrough of the process, from an email's arrival to the generation of a draft reply:

1.  **Email Arrival & Notification:** A new email is delivered to the user's Gmail inbox. The pre-configured Gmail API `watch` on the mailbox detects this change and sends a push notification to a Google Cloud Pub/Sub topic. The notification payload contains the user's email address and the new `historyId` for their mailbox, encoded in base64url format.

2.  **Secure Ingestion into AWS:** The Pub/Sub topic immediately forwards this notification via an authenticated HTTPS POST request to an AWS API Gateway endpoint. A Lambda function triggered by the gateway verifies the request's OIDC token to confirm its origin, then places the decoded `historyId` and user email into an SQS queue.

3.  **Syncing Mailbox History:** A worker process, triggered by the SQS message, retrieves the user's last synced `historyId` from a DynamoDB table. It then calls the Gmail API `users.history.list` with this ID as `startHistoryId`. This API call returns a list of all changes (new messages, label changes, etc.) that have occurred since the last sync.

4.  **Fetching New Message Content:** The worker iterates through the history list. When it identifies a new message, it uses the message ID to call the `users.messages.get` API to retrieve the full message content, including headers, body, and sender information. To prevent reprocessing, it uses the Gmail message ID as a deduplication key when updating its state in DynamoDB.

5.  **Initiating Triage Workflow:** For each new, relevant email, the worker initiates an execution of the main AWS Step Functions state machine, passing the message content and metadata as input. The message's status is marked as `processing` in DynamoDB.

6.  **AI Analysis and Decision:** A task within the Step Functions workflow invokes Amazon Bedrock. It sends the email content, relevant thread history, and a prompt instructing the Claude model to analyze the message and decide on the next action. The model might classify the email's intent and determine that a reply is warranted.

7.  **Human-in-the-Loop (If Necessary):** If the workflow requires confirmation, the Step Function can enter a wait state using a task token. It could, for example, use SNS to send a notification to the user asking for approval to proceed. The workflow pauses until it receives a signal to continue.

8.  **Drafting the Reply:** Once approved (or if no approval is needed), the model invokes a server-side tool, which is a Lambda function responsible for creating drafts. This function constructs a complete RFC 2822 MIME message. Crucially, it sets the `threadId` to match the original conversation, sets the `In-Reply-To` header to the `Message-ID` of the email being replied to, and ensures the `Subject` line matches. It then base64url-encodes this MIME message.

9.  **Creating the Draft in Gmail:** The drafting Lambda calls the `users.drafts.create` Gmail API endpoint, providing the encoded MIME message. The Gmail API creates the draft in the user's account, correctly associated with the existing thread.

10. **Finalizing the Process:** The Lambda function returns the ID of the newly created draft to the Step Functions workflow. The workflow's final action is to update the message's status in DynamoDB to `drafted`, storing the draft ID for reference. The user can now find the AI-generated draft in their Gmail interface, ready to be reviewed, edited, and sent.

# Security Considerations

## Area

Webhook and API Authentication

## Best Practice

All incoming events must be authenticated to ensure they originate from a trusted source. For cross-cloud communication, such as receiving Gmail push notifications from Google Cloud Pub/Sub to an AWS endpoint, a standard-based authentication mechanism like OpenID Connect (OIDC) should be used.

## Implementation Detail

The recommended architecture uses Google Cloud Pub/Sub's authenticated push feature to send notifications to an AWS API Gateway HTTPS endpoint. This feature includes a signed OIDC JSON Web Token (JWT) in the 'Authorization' header of the POST request. The backend AWS Lambda function triggered by API Gateway must validate this token. The validation process involves checking the token's signature against Google's public keys, verifying the issuer ('iss') is 'https://accounts.google.com', and ensuring the audience ('aud') claim matches the URL of the receiving service. This confirms the request is a legitimate notification from the configured Pub/Sub subscription and not a malicious attempt to trigger the workflow.


# Scalability And Api Quota Management

The proposed event-driven, serverless architecture is inherently scalable and designed to efficiently manage Gmail API quotas. Instead of constant polling, which is inefficient and prone to hitting API rate limits, the architecture uses the Google-recommended push notification pattern. This is achieved by setting a 'watch' on the user's mailbox, which directs the Gmail API to send a notification to a Google Cloud Pub/Sub topic whenever a change occurs. This notification is a lightweight JSON payload containing the user's email address and a `historyId`. The backend service then makes a single `users.history.list` call with the last known `historyId` to retrieve only the new changes, drastically reducing API call volume compared to repeatedly listing all messages. This push-based model provides near-real-time updates while staying well within API quota limits. The architecture's scalability is further enhanced by its use of AWS serverless components. The API Gateway, Lambda functions, SQS queue, and Step Functions workflows automatically scale with the volume of incoming email notifications, ensuring the system can handle both low and high traffic without performance degradation or the need for manual infrastructure management.

# Cost Analysis Overview

The cost profile of this architecture is highly efficient due to its reliance on serverless, pay-per-use components. Unlike a traditional server-side daemon running 24/7 on an EC2 instance, which incurs costs even when idle, this model's expenses are directly tied to the volume of email processing. The primary AWS services used—API Gateway, Lambda, SQS, Step Functions, and Amazon Bedrock—all follow a pay-per-use pricing model. Costs are incurred for each API call received by API Gateway, for the compute time (in milliseconds) of Lambda function executions, per state transition in Step Functions, and for the number of input/output tokens processed by the Claude model on Amazon Bedrock. Data storage and access costs for DynamoDB and SQS are also based on usage. This means that if no emails are received, the cost is near zero. As email volume increases, the cost scales linearly. This model is significantly more cost-effective for a personal agent application, where workload can be sporadic, as it eliminates the expense of maintaining idle compute resources.

# Open Source Reference Architectures

## Project Name

Gmail API Push Notification Guide

## Url

https://developers.google.com/workspace/gmail/api/guides/push

## Description

Official Google documentation explaining the recommended pattern for receiving near-real-time updates from a Gmail mailbox. It details how to set up a `watch` on a mailbox, receive notifications via Google Cloud Pub/Sub, and use the `historyId` to sync changes. This is the foundational piece for the event ingress part of the architecture.

## Project Name

Gmail API Threading and Drafts Guides

## Url

https://developers.google.com/workspace/gmail/api/guides/threads

## Description

Official Google documentation that explains the specific criteria for creating a draft or message that correctly becomes part of an existing email thread. It covers the mandatory requirements for `threadId`, `References`/`In-Reply-To` headers, and matching `Subject` lines, which is critical for the 'reply-all' functionality.

## Project Name

AWS Bedrock Samples for Agents and Function Calling

## Url

https://github.com/aws-samples/amazon-bedrock-samples/tree/main/agents-and-function-calling

## Description

A collection of sample code and notebooks from AWS that demonstrates how to use Amazon Bedrock with function calling (tool use). It provides practical examples of creating agents and integrating them with custom tools (like Lambda functions), which is the core pattern for enabling the Claude-based agent to interact with the Gmail API and other services.

## Project Name

Google Identity - Web Server OAuth 2.0 Flow

## Url

https://developers.google.com/workspace/gmail/api/auth/web-server

## Description

The official guide for implementing server-side OAuth2 with Google APIs. It explains how to obtain and, crucially, how to use and manage offline refresh tokens, which are essential for a long-running daemon that needs to access user data when the user is not present. It also covers token expiration and revocation scenarios.

## Project Name

AWS Step Functions Best Practices

## Url

https://docs.aws.amazon.com/step-functions/latest/dg/sfn-best-practices.html

## Description

AWS documentation outlining best practices for using Step Functions. This is relevant for designing the email triage workflow, especially regarding the use of timeouts, heartbeats for long-running tasks, and the `.waitForTaskToken` pattern for implementing human-in-the-loop interactions.


# Analysis Of Alternative Patterns

A comparative analysis of the patterns mentioned in the query reveals why the recommended modern, event-driven architecture is superior for a 2026 implementation. 

1.  **Server-Side Polling Daemon vs. Event-Driven Push Notifications**: The context strongly advises against a continuous polling daemon. Polling the Gmail API (e.g., via `users.messages.list`) is inefficient, introduces latency between an email's arrival and its detection, and is highly prone to exhausting API rate limits, especially as the user base scales. The recommended push-based architecture, using Gmail API `watch` and `history.list`, is far superior. It provides near-real-time notifications, is significantly more efficient in API quota consumption, and is more scalable as it only processes events as they occur.

2.  **Python Sidecar vs. Bash Tools vs. Native Bedrock Tool Integration**: The query considers different ways to implement the agent's actions. 
    *   **Bash Tools**: The context correctly identifies that bash scripts are unsuitable for this task. They lack the robustness needed for complex API interactions, secure OAuth token management, MIME message composition, and error handling (e.g., exponential backoff).
    *   **Python Service/Sidecar**: A dedicated Python service (running on EC2, ECS, or Fargate) is a viable and robust pattern. It can effectively encapsulate all the logic for interacting with the Gmail API, managing state, and composing replies. This is a significant improvement over bash scripts.
    *   **Native Bedrock Tool Integration (Recommended)**: The most modern and idiomatic pattern for 2026, as highlighted in the context, is to use Amazon Bedrock's server-side tool use capabilities. This involves defining the agent's tools (e.g., 'send_email', 'create_draft') and implementing them as AWS Lambda functions. When the Claude model decides to use a tool, Bedrock securely invokes the corresponding Lambda function. This approach is superior because it is serverless (no infrastructure to manage), inherently scalable, keeps tool execution within a governed AWS environment, and represents the standard for building capable, function-calling AI agents on AWS.
