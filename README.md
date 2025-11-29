# Teams Chatbot with n8n Workflow

A Microsoft Teams chatbot that processes mentions, retrieves reply context, intelligently selects custom or generic prompts, and responds using ChatGPT via n8n orchestration.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Prerequisites](#prerequisites)
- [High-Level Flow](#high-level-flow)
- [Setup Guide](#setup-guide)
- [Workflow Details](#workflow-details)
- [API Reference](#api-reference)
- [Prompt Strategy](#prompt-strategy)
- [Implementation Phases](#implementation-phases)
- [Configuration](#configuration)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
- [Future Enhancements](#future-enhancements)

## Overview

### Objectives

This project implements a **chatbot within Microsoft Teams** (using bot/team account) that can:

- **Receive messages** in Teams groups when the bot is **mentioned**
- **Read the full question** + **replied message content** (thread context)
- **Process content** through **n8n → AI Agent (OpenAI)** for analysis and response
- **Automatically select prompts**:
  - Uses **AI Agent** to intelligently analyze questions and select the best prompt
  - If AI determines a **custom prompt** matches → use custom prompt
  - Otherwise → use **generic Q&A prompt**
- **Reply back in Teams**, mention the correct user, and show **typing indicator** before responding

### Key Features

- ✅ Mention detection and filtering
- ✅ Thread context retrieval (reply-to-message support)
- ✅ Intelligent prompt selection (custom vs generic)
- ✅ n8n AI Agent integration for AI-powered responses (using OpenAI)
- ✅ Typing indicator for better UX
- ✅ Comprehensive logging and monitoring

## Architecture

```
┌─────────────┐
│   Teams     │
│   Group     │
└──────┬──────┘
       │ User mentions bot / replies
       ▼
┌─────────────────────────────────────┐
│         n8n Workflow                │
│  ┌───────────────────────────────┐  │
│  │ 1. HTTP Trigger (Webhook)     │  │
│  │ 2. Detect Mention             │  │
│  │ 3. Get Reply Context          │  │
│  │ 4. Send Typing Indicator      │  │
│  │ 5. AI Agent - Select Prompt   │  │
│  │ 6. IF/ELSE: Select Prompt     │  │
│  │ 7A/7B: Prepare Prompt         │  │
│  │ 8A/8B: AI Agent               │  │
│  │ 9. Merge Results              │  │
│  │ 10. Format Teams Message      │  │
│  │ 11. Send Reply to Teams       │  │
│  │ 12. Log Interaction           │  │
│  └───────────────────────────────┘  │
└──────┬──────────────────┬───────────┘
       │                  │
       ▼                  ▼
┌──────────────┐   ┌──────────────┐
│ Custom Prompt│   │   OpenAI     │
│     API     │   │   AI Agent   │
└──────────────┘   └──────────────┘
```

### Data Flow

1. **Teams** → Webhook → **n8n** (message payload)
2. **n8n** → **Microsoft Graph API** (get reply context, send typing, send reply)
3. **n8n** → **AI Agent** (intelligently analyze and select appropriate prompt)
4. **n8n** → **OpenAI ChatGPT** (generate response)
5. **n8n** → **Logging API** (store interaction data)
6. **n8n** → **Teams** (send formatted reply)

## Tech Stack

- **Microsoft Teams**: Message reception/sending, user mentions, typing indicators
- **n8n**: Workflow orchestration, prompt building, API integration, Teams communication
- **n8n AI Agent (OpenAI)**: AI agent node for content processing and response generation with enhanced capabilities (tool calling, better context handling, improved prompt management)
- **AI Agent (Prompt Selection)**: Uses AI to intelligently analyze questions and select the most appropriate prompt
- **Microsoft Graph API**: Teams integration (messages, mentions, typing)
- **Bitbucket**: Source code management (API, prompts, documentation, n8n workflows)

## Prerequisites

### Required Services & Accounts

1. **Microsoft Teams**
   - Bot account / App registration
   - Graph API permissions:
     - `Chat.ReadWrite`
     - `ChatMessage.Send`
     - `ChannelMessage.Read.All`
     - `ChannelMessage.Send`

2. **n8n Instance**
   - Self-hosted or n8n Cloud
   - Version 1.x or higher
   - Access to install nodes and configure credentials

3. **OpenAI Account**
   - API key
   - Access to GPT-4 or GPT-3.5-turbo models

4. **Prompt Templates** (stored in workflow)
   - Custom prompts are defined in the workflow code
   - AI Agent analyzes questions and selects from available prompts
   - No external API required

5. **Logging Service** (optional)
   - Database (MySQL/PostgreSQL) or API endpoint
   - For storing interaction logs

### Required Credentials

- Microsoft Graph API OAuth2 credentials
- OpenAI API key
- (Optional) External prompt API URL (if you want to load prompts from external source)
- Logging API endpoint URL (if applicable)

## High-Level Flow

1. **User sends message** in Teams group and **mentions bot** (or replies to a message and mentions bot)

2. **Teams → n8n** (Webhook/Graph API trigger):
   - n8n receives payload: message content, sender info, mention info, replyToId/thread info

3. **n8n Processing**:
   - **Detect mention bot** → only process if bot is mentioned
   - **Collect context**:
     - Current message content
     - If reply: fetch original message content (call Graph API for parent message)
   - Build **normalized chatInput** (question + reply content)

4. **n8n uses AI Agent** to analyze chatInput → AI intelligently decides:
   - Which custom prompt to use (if any)
   - Or use generic prompt

5. **n8n IF/ELSE Branch**:
   - If **custom prompt found** → build system prompt from custom → call AI Agent
   - If **not found** → use **generic QA prompt** → call AI Agent

6. **Receive response** from AI Agent → format message (add user mention)

7. **n8n sends reply** back to **Teams** in correct thread, mention the asking user

8. **Log result** (question content, prompt used, answer, timestamp) for debugging and improvement

**Bonus**: When n8n receives message from Teams → immediately send **"typing indicator"** to user in group.

## Setup Guide

### Step 1: Microsoft Teams Bot Registration

1. Go to [Azure Portal](https://portal.azure.com)
2. Create a new App Registration
3. Configure:
   - Name: `Teams Chatbot`
   - Supported account types: `Accounts in any organizational directory`
   - Redirect URI: `https://your-n8n-instance.com/webhook/teams-webhook`
4. Generate a client secret
5. Add API permissions:
   - Microsoft Graph → `Chat.ReadWrite`
   - Microsoft Graph → `ChatMessage.Send`
   - Microsoft Graph → `ChannelMessage.Read.All`
   - Microsoft Graph → `ChannelMessage.Send`
   - Microsoft Graph → `Subscription.ReadWrite.All` (for webhook subscriptions)
6. Grant admin consent
7. Note down:
   - Application (client) ID (this is your `TEAMS_BOT_ID`)
   - Directory (tenant) ID
   - Client secret value

### Step 2: Configure n8n

1. **Install n8n** (if not already installed):
   ```bash
   npm install n8n -g
   n8n start
   ```

2. **Import Workflow**:
   - Open n8n UI (usually `http://localhost:5678`)
   - Go to Workflows → Import from File
   - Select `n8n-workflows/teams_chatbot_main.json`

3. **Configure Credentials**:

   **Microsoft Graph API OAuth2**:
   - Credential name: `Microsoft Graph API`
   - Client ID: Your Azure App Registration Client ID
   - Client Secret: Your Azure App Registration Client Secret
   - Authorization URL: `https://login.microsoftonline.com/common/oauth2/v2.0/authorize`
   - Access Token URL: `https://login.microsoftonline.com/common/oauth2/v2.0/token`
   - Scope: `https://graph.microsoft.com/.default`

   **OpenAI API**:
   - Credential name: `OpenAI API`
   - API Key: Your OpenAI API key

4. **Set Environment Variables**:
   ```bash
   export TEAMS_BOT_ID="your-bot-id"
   export OPENAI_MODEL="gpt-4"
   export PROMPT_API_URL="https://your-api.com"
   export LOGGING_API_URL="https://your-logging-api.com/log"
   ```

### Step 3: Set Up Microsoft Graph API Subscription

The workflow uses **Microsoft Graph API webhook subscriptions** to receive notifications when new messages are created in Teams. This is the recommended approach for receiving messages with mentions.

#### Option A: Use Setup Workflow (Recommended)

1. **Import Setup Workflow**:
   - Import `n8n-workflows/setup_graph_subscription.json` into n8n
   - Activate the workflow

2. **Get Your Webhook URL**:
   - In the main workflow, open **"HTTP Trigger - Graph API Notification"** node
   - Copy the webhook URL (e.g., `https://your-n8n-instance.com/webhook/teams-notification`)
   - Set environment variable: `N8N_WEBHOOK_URL=https://your-n8n-instance.com`

3. **Create Subscription**:
   - Call the setup workflow endpoint: `POST https://your-n8n-instance.com/webhook/setup-subscription`
   - Or trigger it manually from n8n UI
   - Save the returned `subscriptionId` for renewal

4. **Subscription Renewal**:
   - Graph API subscriptions expire after **3 days maximum**
   - Set up a scheduled workflow to renew subscriptions before expiration
   - Use the same setup workflow or call Graph API directly:
     ```http
     PATCH https://graph.microsoft.com/v1.0/subscriptions/{subscriptionId}
     Content-Type: application/json
     
     {
       "expirationDateTime": "2024-01-04T00:00:00Z"
     }
     ```

#### Option B: Manual Setup via Graph API

1. **Create Subscription** (one-time setup):
   ```http
   POST https://graph.microsoft.com/v1.0/subscriptions
   Content-Type: application/json
   Authorization: Bearer {access_token}
   
   {
     "changeType": "created",
     "notificationUrl": "https://your-n8n-instance.com/webhook/teams-notification",
     "resource": "/teams/allMessages",
     "expirationDateTime": "2024-01-04T00:00:00Z",
     "clientState": "teams-chatbot-subscription"
   }
   ```

2. **Handle Validation**:
   - When Graph API first calls your webhook, it sends a validation request
   - The workflow automatically handles this and returns the validation token
   - No manual intervention needed

3. **Required Permissions**:
   - Application permission: `Subscription.ReadWrite.All`
   - Ensure admin consent is granted

#### Important Notes:

- **Subscription Expiration**: Subscriptions expire after 3 days. You must renew them regularly.
- **Webhook URL**: Must be publicly accessible (HTTPS required)
- **Resource Path**: `/teams/allMessages` subscribes to all team messages
- **Filtering**: The workflow filters messages to only process those mentioning the bot

### Step 4: Activate Main Workflow

1. **Activate the Main Workflow**:
   - Open `teams_chatbot_main.json` workflow in n8n
   - Click "Active" toggle to activate
   - Ensure webhook is accessible from internet

2. **Test Webhook**:
   - Graph API will send a validation request when subscription is created
   - Check workflow execution logs to verify validation is handled correctly

### Step 5: Configure Prompt Templates (Optional)

Prompt templates are stored directly in the workflow. You can:
- Modify prompt templates in the "Prepare Prompt Selection" node
- Add new custom prompts to the `availablePrompts` array
- Customize the AI Agent's prompt selection logic

### Step 6: Test Workflow

1. Activate the workflow in n8n
2. Send a test message in Teams mentioning the bot
3. Check n8n execution logs for any errors
4. Verify bot responds correctly

## Workflow Details

The n8n workflow consists of 12 nodes connected in a specific sequence:

### Node 1: HTTP Trigger - Graph API Notification
- **Type**: Webhook
- **Purpose**: Receives POST requests from Microsoft Graph API when new messages are created
- **Input**: Graph API notification payload (validation tokens or notification data)
- **Output**: Notification data

### Node 1: Validate & Extract Notification
- **Type**: Code (JavaScript)
- **Purpose**: 
  - Handles Graph API validation requests (returns validation token)
  - Extracts resource URL, message ID, team ID, channel ID from notifications
  - Filters for 'created' change type only
- **Output**: Resource URL, message/team/channel IDs, or validation token

### Node 1: Check Validation Request (IF Node)
- **Type**: IF
- **Purpose**: Branches based on whether request is validation or actual notification
- **True branch**: Return validation token (confirms subscription)
- **False branch**: Continue to fetch message

### Node 1: Return Validation Token
- **Type**: Respond to Webhook
- **Purpose**: Returns validation token to Graph API to confirm webhook subscription
- **Output**: HTTP 200 with validation token as plain text

### Node 1: Fetch Message from Graph API
- **Type**: HTTP Request
- **Purpose**: Fetches the actual message content from Graph API using resource URL
- **API**: `GET {resource}?$expand=mentions`
- **Output**: Full message object with mentions expanded

### Node 2: Detect Mention & Preprocess
- **Type**: Code (JavaScript)
- **Purpose**: 
  - Check if bot is mentioned (using Graph API mentions format)
  - Extract user information (ID, name) from Graph API message format
  - Clean message text (remove HTML/mention tags, decode entities)
  - Detect if message is a reply
  - Extract team and channel IDs
- **Input**: Graph API message format with expanded mentions
- **Output**: 
  - `userId`, `userName`, `plainText`
  - `hasReply`, `replyToId`, `conversationId`, `messageId`
  - `teamId`, `channelId`
- **Exit condition**: Returns `null` if bot not mentioned (stops workflow)

### Node 3: Check Has Reply (IF Node)
- **Type**: IF
- **Purpose**: Branch workflow based on whether message is a reply
- **True branch**: Go to "Get Reply Context"
- **False branch**: Skip to "Build Context Text"

### Node 3: Get Reply Context (Graph API)
- **Type**: HTTP Request
- **Purpose**: Fetch original message content that was replied to
- **API**: `GET /teams/{teamId}/channels/{channelId}/messages/{replyToId}`
- **Only executes**: If `hasReply == true`
- **Uses**: `teamId` and `channelId` from Graph API message format
- **Output**: Original message content

### Node 3: Build Context Text
- **Type**: Code (JavaScript)
- **Purpose**: Combine original message (if reply) with current question
- **Output**: `contextText` = "Original message: ... --- User question: ..."

### Node 4: Send Typing Indicator
- **Type**: HTTP Request
- **Purpose**: Show typing indicator in Teams (UX improvement)
- **API**: `POST /teams/{teamId}/channels/{channelId}/messages/{messageId}/sendTyping`
- **Note**: This shows users that the bot is processing their message

### Node 5: Prepare Prompt Selection
- **Type**: Code (JavaScript)
- **Purpose**: Prepares context and instructions for AI Agent to analyze and select prompt
- **Input**: `contextText` from previous node
- **Output**: 
  - List of available custom prompts with descriptions and keywords
  - Prompt selection instruction for AI Agent
  - Context data

### Node 5: AI Agent - Select Prompt
- **Type**: AI Agent (n8n)
- **Purpose**: Uses AI to intelligently analyze the user's question and determine which prompt to use
- **Model**: GPT-4 (configurable, lower temperature for consistent decisions)
- **Output**: JSON response with:
  ```json
  {
    "matched": true/false,
    "name_prompt": "hr_leave_policy_faq",
    "reason": "Question is about leave policy"
  }
  ```
- **Advantages**: 
  - More intelligent than keyword matching
  - Understands context and intent
  - No external API required
  - Can be improved with better prompts

### Node 5: Parse Prompt Selection
- **Type**: Code (JavaScript)
- **Purpose**: Parses AI Agent's JSON response and extracts prompt selection decision
- **Handles**: JSON parsing, error handling, fallback to generic prompt
- **Output**: `matched`, `name_prompt`, `reason`, `selectedPrompt`

### Node 6: Check Prompt Match (IF Node)
- **Type**: IF
- **Purpose**: Branch based on AI Agent's prompt selection decision
- **Condition**: `matched == true` (AI determined a custom prompt should be used)
- **True branch**: Use custom prompt (Node 7A → 8A)
- **False branch**: Use generic prompt (Node 7B → 8B)

### Node 7A: Prepare Custom Prompt
- **Type**: Code (JavaScript)
- **Purpose**: Build AI Agent request with custom system prompt based on AI's selection
- **Input**: Selected prompt name from AI Agent decision
- **Process**: 
  - Looks up prompt template from `promptTemplates` object
  - Maps prompt name to actual prompt content
- **Output**: `systemPrompt`, `userMessage`, `promptName`, `promptType: 'custom'`, `reason`

### Node 8A: AI Agent (Custom)
- **Type**: AI Agent (n8n)
- **Purpose**: Uses n8n AI Agent with custom system prompt
- **Model**: GPT-4 (configurable)
- **Features**: 
  - Enhanced context handling
  - Tool calling capabilities
  - Better prompt management
- **Output**: AI-generated response

### Node 7B: Prepare Generic Prompt
- **Type**: Code (JavaScript)
- **Purpose**: Build AI Agent request with generic system prompt
- **System Prompt**: "You are a helpful assistant for Ohmyhotel internal communication..."
- **Output**: `systemPrompt`, `userMessage`, `promptName: 'generic'`, `promptType: 'generic'`

### Node 8B: AI Agent (Generic)
- **Type**: AI Agent (n8n)
- **Purpose**: Uses n8n AI Agent with generic system prompt
- **Model**: GPT-4 (configurable)
- **Features**: 
  - Enhanced context handling
  - Tool calling capabilities
  - Better prompt management
- **Output**: AI-generated response

### Node 9: Merge Results
- **Type**: Merge
- **Purpose**: Combine outputs from both branches (only one will have data)
- **Mode**: Combine all

### Node 10: Format Teams Message
- **Type**: Code (JavaScript)
- **Purpose**: Format AI response for Teams
- **Format**: 
  - Add user mention: `<at id="userId">userName</at>`
  - Add answer text
  - Structure as Teams message with mentions array

### Node 11: Send Reply to Teams
- **Type**: HTTP Request
- **Purpose**: Send formatted reply back to Teams
- **API**: `POST /teams/{teamId}/channels/{channelId}/messages`
- **Uses**: `teamId` and `channelId` from Graph API message format
- **Body**: Formatted message with mentions (HTML format)

### Node 12: Log Interaction
- **Type**: HTTP Request
- **Purpose**: Log interaction for monitoring and analytics
- **Data logged**:
  - `timestamp`
  - `userId`, `userName`
  - `contextText`
  - `promptUsed` (name_prompt or "generic")
  - `promptType` (custom/generic)
  - `answerLength`
  - `modelName`
  - `promptSelectionReason` (AI's reasoning for prompt selection)
  - `aiSelectionUsed` (indicates AI Agent was used for selection)

## AI-Powered Prompt Selection

### How It Works

Instead of using a traditional API with keyword matching, the workflow uses **AI Agent** to intelligently analyze questions and select the most appropriate prompt.

### Process Flow

1. **Prepare Prompt Selection**:
   - Builds a list of available custom prompts with descriptions and keywords
   - Creates an instruction prompt for AI Agent
   - Includes the user's question context

2. **AI Agent Analysis**:
   - AI Agent receives the question and available prompts
   - Analyzes the question's intent, context, and domain
   - Makes an intelligent decision about which prompt to use

3. **Response Format**:
   ```json
   {
     "matched": true,
     "name_prompt": "hr_leave_policy_faq",
     "reason": "Question is about leave policy and vacation requests"
   }
   ```
   or
   ```json
   {
     "matched": false,
     "reason": "Question is general and doesn't match any specific domain"
   }
   ```

### Advantages of AI-Powered Selection

1. **Context Understanding**: AI understands intent, not just keywords
2. **Flexibility**: Can handle variations in phrasing and language
3. **No External API**: Everything runs within n8n workflow
4. **Easy to Improve**: Just update the AI Agent's instruction prompt
5. **Explainable**: AI provides reasoning for its decision

### Customizing Prompt Selection

To add or modify prompts, edit the `availablePrompts` array in the **"Prepare Prompt Selection"** node:

```javascript
const availablePrompts = [
  {
    name: 'your_new_prompt',
    description: 'Description of what this prompt handles',
    keywords: ['keyword1', 'keyword2', 'keyword3']
  },
  // ... more prompts
];
```

Then add the corresponding prompt template in the **"Prepare Custom Prompt"** node:

```javascript
const promptTemplates = {
  'your_new_prompt': `Your custom prompt content here...`,
  // ... more templates
};
```

### Improving Selection Accuracy

To improve AI Agent's prompt selection:

1. **Better Descriptions**: Write clear, detailed descriptions for each prompt
2. **Relevant Keywords**: Include comprehensive keywords that users might use
3. **Better Instructions**: Refine the instruction prompt in "Prepare Prompt Selection" node
4. **Examples**: Add example questions to the instruction prompt
5. **Temperature**: Lower temperature (0.3) ensures more consistent decisions

## Prompt Strategy

### Generic Prompt (Fallback)

Used for all questions that don't match any custom prompt.

**Example**:
```
You are a helpful assistant for Ohmyhotel internal communication. 
Answer users' questions clearly and concisely. 
If you don't know the answer, politely say so and suggest they contact 
the relevant department (HR, IT, etc.). 
Keep responses professional and friendly.
```

**Characteristics**:
- General-purpose Q&A
- Professional tone
- Admits uncertainty when appropriate
- Directs users to relevant departments

### Custom Prompts (Domain-Specific)

Examples of custom prompt domains:

1. **HR Leave Policy FAQ** (`hr_leave_policy_faq`)
   - Role: HR assistant
   - Focus: Leave policies, vacation requests, holidays
   - Tone: Formal, helpful

2. **B2C Booking Support** (`b2c_booking_support`)
   - Role: Customer support agent
   - Focus: Booking inquiries, cancellations, modifications
   - Tone: Friendly, solution-oriented

3. **DevOps CI/CD Helper** (`devops_ci_cd_helper`)
   - Role: Technical assistant
   - Focus: CI/CD pipelines, deployment, infrastructure
   - Tone: Technical, precise

4. **Company General Info** (`company_general_info`)
   - Role: Company information assistant
   - Focus: Company policies, office locations, general FAQs
   - Tone: Informative, concise

**Custom Prompt Structure**:
- Role definition
- Domain-specific context
- Tone and style guidelines
- Rules (e.g., "Don't make up data if unsure → ask user to contact HR")

## Implementation Phases

### Phase 1 – MVP (Bot Responds with Generic Prompt)

**Goal**: Basic bot functionality with generic responses.

**Tasks**:
- [ ] Create bot account / app registration for Teams + grant Graph permissions
- [ ] Set up n8n credentials for Graph API
- [ ] Build workflow:
  - Trigger to receive messages
  - Mention detection
   - Call AI Agent with generic prompt
  - Reply in thread + mention user
- [ ] Simple logging (file/DB)

**Deliverables**:
- Working bot that responds to mentions
- Basic logging functionality

### Phase 2 – Reply Context & Typing UX

**Goal**: Enhanced context understanding and better UX.

**Tasks**:
- [ ] Add logic to get `replyToId` → call Graph API → fetch original message content
- [ ] Build `contextText` = reply-content + user question
- [ ] Add node to send **typing indicator** when message received
- [ ] Improve reply formatting: line breaks, bullet lists

**Deliverables**:
- Bot understands thread context
- Typing indicator shows before response
- Better formatted responses

### Phase 3 – AI-Powered Prompt Selection

**Goal**: Intelligent prompt selection using AI Agent instead of API matching.

**Tasks**:
- [ ] Configure AI Agent for prompt selection
- [ ] Define custom prompt templates in workflow
- [ ] Create 3–5 initial custom prompts (HR, booking, DevOps)
- [ ] Test AI Agent's prompt selection accuracy
- [ ] Refine AI Agent instructions based on test results
- [ ] Log: `name_prompt` + `promptSelectionReason` for monitoring

**Deliverables**:
- AI Agent-based prompt selection
- Domain-specific prompt templates stored in workflow
- Intelligent prompt selection logic
- No external API required

### Phase 4 – Optimization & Monitoring

**Goal**: Production-ready system with monitoring and analytics.

**Tasks**:
- [ ] Add rate limiting / throttling
- [ ] Add metrics:
  - Questions per day
  - Custom vs generic prompt usage ratio
- [ ] Dashboard for AI usage (Elastic / Grafana / n8n + Notion/Sheet)
- [ ] Collect team feedback → update prompts & matching logic

**Deliverables**:
- Rate limiting implemented
- Analytics dashboard
- Improved prompts based on feedback

## Configuration

### Environment Variables

Set these in your n8n environment or `.env` file:

```bash
# Teams Configuration
TEAMS_BOT_ID="your-bot-application-id"  # Azure App Registration Client ID
MICROSOFT_APP_ID="your-bot-application-id"  # Alternative name (same value)

# OpenAI Configuration
OPENAI_MODEL="gpt-4"  # or "gpt-3.5-turbo"

# API Endpoints
PROMPT_API_URL="https://your-prompt-api.com"
LOGGING_API_URL="https://your-logging-api.com/log"

# n8n Webhook URL (for Graph API subscription setup)
N8N_WEBHOOK_URL="https://your-n8n-instance.com"

# Optional: Custom thresholds
PROMPT_MATCH_THRESHOLD=0.7
```

### n8n Credentials

1. **Microsoft Graph API OAuth2**:
   - Client ID: Azure App Registration Client ID
   - Client Secret: Azure App Registration Client Secret
   - Tenant ID: Your Azure Tenant ID
   - Scopes: `https://graph.microsoft.com/.default`

2. **OpenAI API**:
   - API Key: Your OpenAI API key
   - Organization ID (optional): Your OpenAI organization ID

### Workflow Settings

- **Execution Mode**: `v1` (default)
- **Error Workflow**: Configure error handling workflow (optional)
- **Webhook Settings**: Ensure webhook is publicly accessible

## Testing

### Manual Testing Steps

1. **Test Mention Detection**:
   - Send message in Teams: `@BotName Hello`
   - Verify workflow triggers
   - Check that bot responds

2. **Test Reply Context**:
   - Reply to an existing message: `@BotName What about this?`
   - Verify bot includes original message in context
   - Check response relevance

3. **Test Custom Prompt**:
   - Ask question matching a custom prompt (e.g., HR leave policy)
   - Verify custom prompt is used (check logs)
   - Verify response uses domain-specific tone

4. **Test Generic Prompt**:
   - Ask unrelated question
   - Verify generic prompt is used
   - Check response quality

5. **Test Typing Indicator**:
   - Send message to bot
   - Verify typing indicator appears before response

### Test Data

**Sample Messages**:
- `@BotName What is our leave policy?` (should match HR prompt)
- `@BotName How do I book a room?` (should match booking prompt)
- `@BotName What's the weather today?` (should use generic prompt)

### Debugging

1. **Check n8n Execution Logs**:
   - View execution history
   - Inspect data at each node
   - Check for errors

2. **Verify API Responses**:
   - Test AI Agent prompt selection with sample questions
   - Test OpenAI API directly
   - Check Graph API permissions

3. **Check Logs**:
   - Review logging endpoint data
   - Analyze prompt usage patterns
   - Identify common errors

## Troubleshooting

### Common Issues

#### Bot Not Responding

**Symptoms**: Bot doesn't reply to mentions

**Solutions**:
1. Check if workflow is activated in n8n
2. Verify webhook URL is correct and accessible
3. Check mention detection logic (verify `TEAMS_BOT_ID`)
4. Review n8n execution logs for errors

#### Graph API Permission Errors

**Symptoms**: 403 Forbidden errors when calling Graph API

**Solutions**:
1. Verify API permissions are granted in Azure Portal
2. Ensure admin consent is given
3. Check OAuth2 credentials are correct
4. Verify scopes include required permissions

#### Custom Prompt Not Matching

**Symptoms**: Always using generic prompt

**Solutions**:
1. Test AI Agent prompt selection with sample questions
2. Review AI Agent's reasoning in execution logs
3. Check if prompt selection instruction is clear
4. Verify available prompts list is complete
5. Review and refine AI Agent's system message for better decisions

#### AI Agent / OpenAI API Errors

**Symptoms**: AI Agent calls failing or no response generated

**Solutions**:
1. Verify OpenAI API key is valid in credentials
2. Check API rate limits
3. Verify model name is correct (e.g., `gpt-4`, `gpt-3.5-turbo`)
4. Check account balance/credits
5. Verify AI Agent node is properly configured:
   - Check system message is set correctly
   - Verify prompt text is being passed
   - Check node output format matches expected structure
6. Review n8n execution logs for AI Agent node errors

#### Typing Indicator Not Showing

**Symptoms**: No typing indicator appears

**Solutions**:
1. Verify typing indicator API endpoint (may vary by Teams version)
2. Check Graph API permissions for typing
3. Review Teams API documentation for latest typing indicator method

#### Graph API Subscription Issues

**Symptoms**: Not receiving message notifications, subscription expired

**Solutions**:
1. **Check Subscription Status**:
   ```http
   GET https://graph.microsoft.com/v1.0/subscriptions/{subscriptionId}
   ```
   - Verify subscription is active
   - Check expiration date

2. **Renew Expired Subscription**:
   - Graph API subscriptions expire after 3 days
   - Use setup workflow or manually renew:
     ```http
     PATCH https://graph.microsoft.com/v1.0/subscriptions/{subscriptionId}
     Content-Type: application/json
     
     {
       "expirationDateTime": "2024-01-07T00:00:00Z"
     }
     ```

3. **Verify Webhook URL**:
   - Ensure webhook URL is publicly accessible (HTTPS required)
   - Test webhook endpoint manually
   - Check firewall/network settings

4. **Check Validation**:
   - Graph API sends validation request when creating subscription
   - Verify workflow handles validation correctly (returns validation token)
   - Check n8n execution logs for validation requests

5. **Subscription Permissions**:
   - Ensure `Subscription.ReadWrite.All` permission is granted
   - Verify admin consent is given
   - Check application permissions (not delegated)

6. **Resource Path**:
   - Verify resource path: `/teams/allMessages`
   - For specific teams/channels, adjust resource path accordingly

## Future Enhancements

### Phase 5+ Ideas

1. **Multi-language Support**:
   - Detect language from message
   - Use language-specific prompts
   - Translate responses if needed

2. **Conversation Memory**:
   - Store conversation history
   - Maintain context across multiple messages
   - Reference previous interactions

3. **Advanced Analytics**:
   - User satisfaction tracking
   - Response quality metrics
   - Prompt effectiveness analysis

4. **Integration with Other Services**:
   - Connect to internal knowledge base
   - Integrate with ticketing systems
   - Link to company databases

5. **Proactive Notifications**:
   - Send reminders based on context
   - Alert users about important updates
   - Schedule follow-up messages

6. **Voice/Video Support**:
   - Process voice messages
   - Generate voice responses
   - Video call integration

## Contributing

This project is managed via Bitbucket. For contributions:

1. Create a feature branch: `feature/your-feature-name`
2. Make changes and test thoroughly
3. Update documentation as needed
4. Create a Pull Request with:
   - Description of changes
   - Screenshots of n8n workflow (if workflow changed)
   - Test results

## License

[Specify your license here]

## Contact

For questions or support, contact the development team.

---

**Last Updated**: 2024-01-01
