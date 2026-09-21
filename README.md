# MealMate
Personalized meals, delivered through dialogue
What Inspired Me
I've been working on AI/ML projects and MLOps pipelines, and when I saw the Alexa+ hackathon opportunity, it clicked: this is exactly the kind of agentic system architecture I want to master. The chance to build a brand integration that actually does things—books appointments, checks orders, modifies deliveries—felt like the perfect intersection of my backend engineering skills and my interest in conversational AI.
What really drew me in was the Model Context Protocol (MCP) itself. After reading about the 2025-11-25 spec release and the shift to Streamable HTTP, I realized this wasn't just another chatbot wrapper—it was a proper tool-calling protocol with JSON-RPC semantics, session management, and real streaming support. That's the kind of infrastructure-level work that excites me as an engineer.
________________________________________
What I Learned
MCP Architecture Deep Dive
Building this project taught me several things I hadn't fully grasped before:
1.	Streamable HTTP vs. Legacy SSE Transport: The spec deprecates the old two-endpoint HTTP+SSE pattern (GET /sse + POST /messages) in favor of a single endpoint that handles both POST requests and optional GET streams. This simplifies deployment but requires understanding when to return application/json vs. text/event-stream.
2.	Tool Definition Semantics: MCP tools aren't just API endpoints—they're LLM-callable functions with JSON Schema input definitions. The inputSchema field determines what the Alexa+ core can infer and elicit from users.
3.	Latency Constraints Matter: Alexa+ requires <500ms round-trip latency for tool calls. This forced me to think about connection pooling, async I/O, and avoiding blocking operations in my tool handlers.
4.	Multi-Turn Conversation Design: MCP add-ons can be invoked across multiple turns, and users may shift topics mid-session. This means my tools need to handle partial information and slot-filling gracefully.
Mathematical Insight: Latency Budget
If we denote the total round-trip latency as L_{\text{total}}, it decomposes as:
L_{\text{total}} = L_{\text{network}} + L_{\text{server}} + L_{\text{tool\_execution}} + L_{\text{response\_serialization}}
For Alexa+ compliance, we need L_{\text{total}} < 500 \, \text{ms}. In practice, I aimed for:
•	L_{\text{network}} \approx 50-100 \, \text{ms} (cloud region proximity)
•	L_{\text{server}} \approx 20-50 \, \text{ms} (FastAPI/uvicorn overhead)
•	L_{\text{tool\_execution}} \approx 200-300 \, \text{ms} (external API calls)
•	L_{\text{response\_serialization}} \approx 10-20 \, \text{ms}
This left me a small buffer for retries and error handling.
________________________________________
How I Built My Project
Choice Summary
Decision	My Choice	Rationale
Brand Use Case	Food Delivery (reordering with modifications)	Familiar domain, clear tool boundaries, realistic API surface
Path	Production-ready MCP Server (not simulation)	Wanted to learn the real spec, not a mock
Language	Python (FastAPI + mcp SDK)	Faster iteration for JSON Schema tool definitions, async support built-in
Architecture Overview
[ Alexa+ Core ] ──POST /mcp──> [ My MCP Server (FastAPI) ]
                                     │
                                     ├── tools/list  → returns tool definitions
                                     ├── tools/call  → executes brand_book_order, brand_modify_order
                                     └── resources/read → optional menu catalog, order history
Step-by-Step Implementation
1. Set Up the MCP Server Skeleton
I used the official Python MCP SDK with Streamable HTTP transport:
from mcp.server.fastmcp import FastMCP
from mcp.server.streamable_http_manager import StreamableHTTPSessionManager

mcp = FastMCP("FoodBrandMCP")

@mcp.tool()
async def brand_book_order(
    date: str,
    serviceType: str,
    items: list[str],
    modifications: dict[str, str] | None = None
) -> dict:
    """Books a food order for the customer with the brand."""
    # Call internal order service
    order_id = await place_order(items, modifications, date)
    return {"orderId": order_id, "status": "confirmed"}
2. Implement Streamable HTTP Endpoint
The spec requires a single /mcp endpoint accepting both POST and GET:
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse, JSONResponse
import json

app = FastAPI()

@app.post("/mcp")
@app.get("/mcp")
async def mcp_endpoint(request: Request):
    if request.method == "POST":
        body = await request.json()
        result = await handle_mcp_request(body)
        # Return JSON or SSE stream depending on response type
        return JSONResponse(result)
    else:
        # GET opens SSE stream for server-initiated notifications
        return StreamingResponse(
            generate_sse_stream(),
            media_type="text/event-stream"
        )
3. Define Tool Schemas
The tool definition follows the MCP JSON-RPC structure with JSON Schema 2020-12 as the default dialect:
{
  "name": "brand_book_order",
  "description": "Books a food order for the customer with the brand, supporting item modifications.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "date": {"type": "string", "description": "ISO 8601 date string"},
      "serviceType": {"type": "string", "enum": ["delivery", "pickup"]},
      "items": {"type": "array", "items": {"type": "string"}},
      "modifications": {
        "type": "object",
        "additionalProperties": {"type": "string"}
      }
    },
    "required": ["date", "serviceType", "items"]
  }
}
4. Handle Multi-Turn Elicitation
When a user says "I want to reorder my usual Friday pizza," the LLM may not have all required fields. MCP supports URL mode elicitation and incremental scope consent:
@mcp.tool()
async def brand_modify_order(
    orderId: str,
    modifications: dict[str, str]
) -> dict:
    """Modifies an existing order (e.g., substitute mushrooms for extra cheese)."""
    if not await order_exists(orderId):
        # Return error that LLM can use to elicit correct orderId from user
        return {"error": "Order not found", "elicitationNeeded": True}
    
    updated = await apply_modifications(orderId, modifications)
    return {"orderId": orderId, "status": "modified", "details": updated}
5. Deploy with Latency Optimization
I deployed on a cloud instance in the same region as Alexa+ endpoints, used uvicorn with workers, and added connection pooling for external APIs:
uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4
________________________________________
Challenges I Faced
1. Streamable HTTP Debugging
The single-endpoint design is elegant but harder to debug than the old two-endpoint SSE pattern. I spent hours figuring out why my GET stream wasn't receiving notifications—turns out I needed to send priming events (empty SSE events with IDs) at stream start for protocol version ≥ 2025-11-25.
2. JSON Schema Validation Gaps
The MCP spec uses JSON Schema 2020-12, but some SDKs still default to Draft 7. I had to explicitly set the dialect and handle additionalProperties and unevaluatedProperties correctly, or the LLM would generate invalid tool calls.
3. Latency Budget Blowout
My first implementation averaged ~700ms round-trip—way over the 500ms limit. The culprit? Synchronous database lookups in my tool handlers. Switching to async SQLAlchemy with connection pooling brought it down to ~320ms.
4. Multi-Turn State Management
Alexa+ can invoke my add-on across multiple turns, and users may shift topics mid-session. I initially tried to maintain session state in memory, but that broke on redeploy. The fix: encode session context in the MCP-Session-Id header and persist minimal state to Redis.
5. Error Handling for LLM Self-Correction
The spec clarifies that input validation errors should be returned as Tool Execution Errors, not Protocol Errors, to enable model self-correction. I had to refactor my error responses to include structured error fields the LLM could parse and use to ask clarifying questions.
________________________________________
Final Thoughts
This project pushed me to think about agentic system design at a deeper level—not just "call an API," but "design tools an LLM can safely invoke with partial information." The MCP spec's emphasis on JSON Schema, streaming, and session management gave me a real production-grade framework to work with.
