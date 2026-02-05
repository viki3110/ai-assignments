# Customer Support Email Agent

An intelligent customer support email processing system built with LangGraph and Google Gemini. This agent automatically classifies incoming emails, routes them through appropriate workflows, drafts responses, and integrates human review for critical cases.

## Features

- **Intelligent Email Classification**: Automatically categorizes emails by intent (question, bug, billing, feature, complex) and urgency (low, medium, high, critical)
- **Dynamic Routing**: Routes emails through different workflows based on classification
- **Knowledge Base Integration**: Searches documentation to provide accurate responses
- **Bug Tracking Integration**: Automatically creates bug tickets for reported issues
- **Human-in-the-Loop**: Pauses for human review on critical or billing issues
- **Persistent State**: Uses checkpointer for conversation persistence across sessions
- **Retry Logic**: Built-in retry policies for transient failures
- **Structured Output**: Uses TypedDict for type-safe classification results

## Architecture

The agent uses a LangGraph state machine with the following workflow:

```
START → read_email → classify_intent → [Route based on classification]
                                              │
                    ┌─────────────────────────┼─────────────────────────┐
                    │                         │                         │
                    ▼                         ▼                         ▼
            search_documentation        bug_tracking          human_review
                    │                         │                         │
                    └─────────────┬───────────┴───────────────┬─────────┘
                                  │                           │
                                  ▼                           ▼
                            draft_response ←──────────────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    │                           │
                    ▼                           ▼
            human_review (if needed)      send_reply
                    │                           │
                    └───────────────┬───────────┘
                                    │
                                    ▼
                                   END
```

### Workflow Nodes

1. **read_email**: Extracts and parses email content
2. **classify_intent**: Uses LLM to classify email intent and urgency, then routes accordingly
3. **search_documentation**: Searches knowledge base for relevant information (for questions/features)
4. **bug_tracking**: Creates bug tracking tickets (for bug reports)
5. **draft_response**: Generates response using context and classification
6. **human_review**: Pauses for human approval/editing (for critical/billing/complex issues)
7. **send_reply**: Sends the final email response

## Installation

### Prerequisites

- Python 3.8+
- Google API Key for Gemini

### Setup

1. Clone the repository and navigate to the customer support agent directory:
```bash
cd customer_support_agent
```

2. Install dependencies:
```bash
pip install -r requirements.txt
```

3. Create a `.env` file in the `customer_support_agent` directory:
```env
GOOGLE_API_KEY=your_google_api_key_here
```

## Configuration

### Environment Variables

- `GOOGLE_API_KEY`: Required. Your Google API key for accessing Gemini models.

### LLM Configuration

The agent uses Google Gemini 2.5 Flash with the following settings:
- Model: `gemini-2.5-flash`
- Temperature: `0` (for consistent classification)

You can modify these settings in the `llm` initialization in `agent.py`:

```python
llm = ChatGoogleGenerativeAI(
    model="gemini-2.5-flash",
    api_key=os.environ["GOOGLE_API_KEY"],
    temperature=0
)
```

## Usage

### Basic Example

```python
from agent import app

# Initialize state with email data
initial_state = {
    "email_content": "I was charged twice for my subscription! This is urgent!",
    "sender_email": "customer@example.com",
    "email_id": "email_123",
    "messages": []
}

# Run with a thread_id for persistence
config = {"configurable": {"thread_id": "customer_123"}}
result = app.invoke(initial_state, config)

# Check if human review is needed
if result.get('__interrupt__'):
    print("Human review required:", result['__interrupt__'])
```

### Handling Human Review

When the workflow pauses for human review, you can resume it with approval/edits:

```python
from langgraph.types import Command

# Provide human input to resume
human_response = Command(
    resume={
        "approved": True,
        "edited_response": "We sincerely apologize for the double charge. I've initiated an immediate refund..."
    }
)

# Resume execution
final_result = app.invoke(human_response, config)
print("Email sent successfully!")
```

### Running the Example

The `agent.py` file includes a complete example at the bottom. Run it with:

```bash
python agent.py
```

## State Structure

### EmailAgentState

The state object passed between nodes:

```python
{
    # Raw email data
    "email_content": str,           # The email body text
    "sender_email": str,             # Sender's email address
    "email_id": str,                 # Unique identifier for the email
    
    # Classification result
    "classification": EmailClassification | None,  # See below
    
    # Raw search/API results
    "search_results": list[str] | None,    # List of raw document chunks
    "customer_history": dict | None,       # Raw customer data from CRM
    
    # Generated content
    "draft_response": str | None,         # Generated response text
    "messages": list[str] | None          # Processing messages
}
```

### EmailClassification

Structured classification result:

```python
{
    "intent": Literal["question", "bug", "billing", "feature", "complex"],
    "urgency": Literal["low", "medium", "high", "critical"],
    "topic": str,      # Main topic/subject of the email
    "summary": str     # Brief summary of the email content
}
```

## Routing Logic

The agent routes emails based on classification:

| Intent | Urgency | Route |
|--------|---------|-------|
| billing | any | → human_review |
| any | critical | → human_review |
| question | low/medium/high | → search_documentation → draft_response |
| feature | low/medium/high | → search_documentation → draft_response |
| bug | any | → bug_tracking → draft_response |
| complex | any | → draft_response → human_review |

After `draft_response`, the workflow checks:
- If urgency is `high` or `critical` → `human_review`
- If intent is `complex` → `human_review`
- Otherwise → `send_reply`

## Node Functions

### `read_email(state: EmailAgentState) -> dict`

Extracts and parses email content. In production, this would connect to your email service (Gmail, Outlook, etc.).

**Returns**: Updates `messages` with processing notification

### `classify_intent(state: EmailAgentState) -> Command`

Uses structured LLM output to classify email intent and urgency. Routes to:
- `search_documentation` for questions/features
- `bug_tracking` for bugs
- `human_review` for billing or critical issues
- `draft_response` for other cases

**Returns**: `Command` with classification update and routing decision

### `search_documentation(state: EmailAgentState) -> Command`

Searches knowledge base using classification topic and intent. Currently uses mock data - replace with your search API.

**Returns**: `Command` with search results and route to `draft_response`

**Retry Policy**: 3 attempts for transient failures

### `bug_tracking(state: EmailAgentState) -> Command`

Creates or updates bug tracking ticket. Currently uses mock ticket ID - integrate with your bug tracking system (Jira, GitHub Issues, etc.).

**Returns**: `Command` with ticket information and route to `draft_response`

### `draft_response(state: EmailAgentState) -> Command`

Generates response using:
- Original email content
- Classification (intent, urgency)
- Search results (if available)
- Customer history (if available)

Routes to `human_review` if urgency is high/critical or intent is complex, otherwise to `send_reply`.

**Returns**: `Command` with draft response and routing decision

### `human_review(state: EmailAgentState) -> Command`

Pauses workflow using `interrupt()` for human review. Human can:
- Approve the draft response
- Edit the response
- Reject (workflow ends, human handles directly)

**Returns**: `Command` with updated response (if approved) and route to `send_reply` or `END`

### `send_reply(state: EmailAgentState) -> dict`

Sends the email response. Currently prints to console - integrate with your email service.

**Returns**: Empty dict (workflow complete)

## Error Handling

- **Search Failures**: The `search_documentation` node has a retry policy (3 attempts) and gracefully handles errors by storing error messages in state
- **Transient Failures**: Retry policies can be added to any node using `RetryPolicy(max_attempts=N)`
- **Human Review**: Critical errors automatically route to human review for manual handling

## Extending the System

### Integrating Real Email Service

Replace the `read_email` function:

```python
def read_email(state: EmailAgentState) -> dict:
    # Connect to your email service
    email = email_service.fetch_email(state['email_id'])
    return {
        "email_content": email.body,
        "sender_email": email.sender,
        "messages": [HumanMessage(content=f"Processing email: {email.body}")]
    }
```

### Integrating Real Search API

Replace the mock search in `search_documentation`:

```python
def search_documentation(state: EmailAgentState) -> Command:
    classification = state.get('classification', {})
    query = f"{classification.get('intent', '')} {classification.get('topic', '')}"
    
    # Use your search API (e.g., Elasticsearch, Pinecone, etc.)
    search_results = search_api.query(query, limit=5)
    
    return Command(
        update={"search_results": [r.content for r in search_results]},
        goto="draft_response"
    )
```

### Integrating Bug Tracking System

Replace the mock ticket creation in `bug_tracking`:

```python
def bug_tracking(state: EmailAgentState) -> Command:
    # Create ticket via API
    ticket = bug_tracker.create_ticket(
        title=state['classification']['topic'],
        description=state['email_content'],
        priority=state['classification']['urgency']
    )
    
    return Command(
        update={"search_results": [f"Bug ticket {ticket.id} created"]},
        goto="draft_response"
    )
```

### Integrating Email Sending

Replace the print statement in `send_reply`:

```python
def send_reply(state: EmailAgentState) -> dict:
    email_service.send_email(
        to=state['sender_email'],
        subject=f"Re: {state['email_id']}",
        body=state['draft_response']
    )
    return {}
```

## Persistence

The agent uses `MemorySaver` checkpointer for state persistence. Each workflow run uses a `thread_id` to maintain conversation context:

```python
config = {"configurable": {"thread_id": "customer_123"}}
```

For production, consider using a persistent checkpointer like `PostgresSaver` or `SqliteSaver`:

```python
from langgraph.checkpoint.postgres import PostgresSaver

checkpointer = PostgresSaver.from_conn_string("postgresql://...")
app = workflow.compile(checkpointer=checkpointer)
```

## Notes

- The workflow is compiled with a checkpointer for persistence. If running with LangGraph Local Server, you may need to compile without a checkpointer.
- The `interrupt()` function must be called first in a node - any code before it will re-run on resume.
- Classification uses structured output for type safety and consistent results.
- All state updates use raw data structures - formatting happens on-demand in prompts.

## Requirements

See `requirements.txt` for full dependency list. Key dependencies:

- `langgraph`: State machine framework
- `langchain-google-genai`: Google Gemini integration
- `python-dotenv`: Environment variable management

## License

[Add your license here]

## Contributing

[Add contribution guidelines here]
