# Ecom Chat Helper

![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat-square&logo=typescript&logoColor=white)
![React](https://img.shields.io/badge/React-61DAFB?style=flat-square&logo=react&logoColor=black)
![Node.js](https://img.shields.io/badge/Node.js-339933?style=flat-square&logo=nodedotjs&logoColor=white)
![Express](https://img.shields.io/badge/Express-000000?style=flat-square&logo=express&logoColor=white)
![LangChain](https://img.shields.io/badge/LangChain-1C3C3C?style=flat-square&logo=langchain&logoColor=white)
![LangGraph](https://img.shields.io/badge/LangGraph-1C3C3C?style=flat-square)
![MongoDB](https://img.shields.io/badge/MongoDB-47A248?style=flat-square&logo=mongodb&logoColor=white)
![Google Gemini](https://img.shields.io/badge/Google_Gemini-8E75B2?style=flat-square&logo=googlegemini&logoColor=white)

An e-commerce application with a conversational assistant for searching a furniture inventory.

The project uses React for the storefront, an Express API for handling chat requests, and a LangGraph agent powered by Google Gemini. The agent retrieves product information from MongoDB using vector similarity search and maintains conversation history across messages.

## Features

- Furniture storefront built with React.
- Conversational assistant using Google Gemini.
- LangGraph workflow with tool calling.
- Semantic product retrieval using MongoDB Atlas Vector Search.
- Text-search fallback when vector search returns no results.
- Persistent conversation history using MongoDB checkpointing.
- Rate-limit handling with exponential backoff.

## Tech Stack

| Component | Technologies |
|-----------|--------------|
| Frontend | React, JavaScript, CSS |
| Backend | Node.js, Express, TypeScript |
| Agent | LangChain, LangGraph |
| Language Model | Google Gemini 1.5 Flash |
| Embeddings | Google `text-embedding-004` |
| Database | MongoDB Atlas |
| Vector Search | MongoDB Atlas Vector Search |
| Conversation Memory | LangGraph MongoDB Checkpointer |

## Architecture

```text
             React Frontend
                   |
                   v
              Express API
                   |
                   v
              LangGraph Agent
                   |
                   v
            Google Gemini
                   |
                   v
           Tool Call Needed?
              /         \
            Yes          No
             |            |
             v            v
        item_lookup    Response
             |
             v
     MongoDB Atlas Vector Search
             |
             v
        Search Results
             |
             v
         Gemini Agent
             |
             v
       Final Response
```

Conversation state is persisted in MongoDB through a LangGraph checkpointer.

## How the Agent Works

The agent is implemented using LangGraph's `StateGraph` and consists of two nodes:

- **`agent`** — Invokes Gemini to process messages, decide whether to use a tool, and generate responses.
- **`tools`** — Executes tool calls requested by the model.

The workflow starts at the agent node. If Gemini requests a tool call, LangGraph executes it and passes the result back to the model. Otherwise, the workflow ends.

### Inventory Lookup

The agent has access to one tool: `item_lookup`.

This tool accepts a search query and an optional number of results to retrieve.

When called, it:

1. Connects to the `items` collection in MongoDB.
2. Checks whether the inventory contains documents.
3. Generates a query embedding using Google's `text-embedding-004` model.
4. Performs similarity search using MongoDB Atlas Vector Search.
5. Falls back to a case-insensitive text search if no vector results are found.
6. Returns the retrieved products to the agent.

The vector search uses the following MongoDB fields:

| Field | Purpose |
|-------|---------|
| `embedding_text` | Text representation of a product |
| `embedding` | Stored vector embedding |
| `vector_index` | Atlas Vector Search index |

The text-search fallback checks `item_name`, `item_description`, `categories`, and `embedding_text`.

If the lookup fails, the tool returns an error message that the agent can use when generating its response.

### Conversation Memory

The agent uses `MongoDBSaver` to persist conversation state.

Each conversation is identified by a `thread_id`, allowing subsequent requests to reuse previous messages.

For example:

```text
User: Show me some sofas.
Agent: [Searches the inventory and returns matching items]

User: Do you have anything cheaper?
Agent: [Uses the conversation history to interpret the follow-up]
```

### Rate-Limit Handling

Requests to Gemini are wrapped in a retry function that handles HTTP `429` errors using exponential backoff.

The implementation allows up to three attempts before returning an error.

## Project Structure

```text
ecom-chat-helper/
├── client/
│   ├── public/
│   ├── src/
│   │   ├── components/
│   │   │   ├── ChatWidget.js
│   │   │   └── EcommerceStore.js
│   │   ├── App.js
│   │   ├── App.css
│   │   ├── index.js
│   │   └── index.css
│   └── package.json
│
├── server/
│   ├── agent.ts
│   ├── index.ts
│   ├── seed-database.ts
│   ├── tsconfig.json
│   └── package.json
│
└── README.md
```

### Main Files

| File | Description |
|------|-------------|
| `client/src/components/ChatWidget.js` | Chat interface |
| `client/src/components/EcommerceStore.js` | Furniture storefront |
| `server/agent.ts` | LangGraph agent and inventory lookup tool |
| `server/index.ts` | Express API and chat endpoints |
| `server/seed-database.ts` | Inventory data generation and seeding |

## Getting Started

### Prerequisites

- Node.js 18+
- npm
- MongoDB Atlas database
- Google AI API key

### 1. Clone the Repository

```bash
git clone https://github.com/YOUR_USERNAME/ecom-chat-helper.git
cd ecom-chat-helper
```

Replace `YOUR_USERNAME` with your GitHub username.

### 2. Install Backend Dependencies

```bash
cd server
npm install
```

### 3. Configure Environment Variables

Create a `.env` file inside the `server` directory:

```env
MONGODB_URI=your_mongodb_connection_string
GOOGLE_API_KEY=your_google_api_key
PORT=8000
```

| Variable | Description |
|----------|-------------|
| `MONGODB_URI` | MongoDB connection string |
| `GOOGLE_API_KEY` | API key for Gemini and Google embeddings |
| `PORT` | Express server port |

Do not commit the `.env` file to Git.

### 4. Set Up MongoDB Atlas

The application expects a database named:

```text
inventory_database
```

With a collection named:

```text
items
```

The collection must contain product documents with the fields used by the inventory lookup tool.

For vector search, configure an Atlas Vector Search index named:

```text
vector_index
```

The index must reference the `embedding` field and use dimensions compatible with the embeddings stored in the collection.

The repository also contains a database seeding script:

```bash
cd server
npm run seed
```

Run this from the repository root. The command assumes the `seed` script is defined in `server/package.json`.

### 5. Start the Backend

From the `server` directory:

```bash
npm run dev
```

The API uses port `8000` by default.

### 6. Install and Start the Frontend

Open another terminal:

```bash
cd client
npm install
npm start
```

The frontend is expected to run at:

```text
http://localhost:3000
```

## API Endpoints

The Express backend exposes the following endpoints:

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/` | Health check |
| `POST` | `/chat` | Start a new conversation |
| `POST` | `/chat/:threadId` | Continue an existing conversation |

### Start a Conversation

```bash
curl -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "Show me some sofas"}'
```

Example response:

```json
{
  "threadId": "1703123456789",
  "response": "..."
}
```

### Continue a Conversation

```bash
curl -X POST http://localhost:8000/chat/1703123456789 \
  -H "Content-Type: application/json" \
  -d '{"message": "Do you have anything cheaper?"}'
```

The existing thread ID allows the agent to retrieve the conversation history.

## Notes

- The agent uses Gemini 1.5 Flash for response generation and tool calling.
- Inventory retrieval uses semantic search rather than relying exclusively on keyword matching.
- The text-search fallback is triggered when vector search returns no results, not when the vector search operation fails.
- Conversation state is stored in MongoDB using LangGraph checkpointing.
- The agent currently supports inventory lookup only. Cart management, checkout, and order processing are not implemented as agent tools.
- MongoDB Atlas Vector Search requires a configured index and product embeddings.
- The application depends on external MongoDB and Google AI services.

## License

See the repository's `LICENSE` file for licensing information.