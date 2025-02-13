```python
import os
import logging
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Dict
from langchain_groq import ChatGroq
from langgraph.graph import StateGraph, START, END, MessagesState
from langgraph.checkpoint.memory import MemorySaver
from langgraph.types import Command, interrupt
from langchain_core.tools import tool
from fastapi.middleware.cors import CORSMiddleware  
from langsmith import Client  # Import LangSmith Client

# Configure logging
logging.basicConfig(level=logging.INFO)

# Set API Keys
os.environ["GROQ_API_KEY"] = "API_KEY"
os.environ["LANGCHAIN_API_KEY"] = "API_KEY"

# Initialize LangSmith Client
langsmith_client = Client()

# FastAPI app
app = FastAPI()

# Add CORS Middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # Allows all origins (Modify in production)
    allow_credentials=True,
    allow_methods=["*"],  # Allows all HTTP methods
    allow_headers=["*"],  # Allows all headers
)

# Define a tool
@tool
def weather_search(city: str):
    """Search for the weather"""
    logging.info(f"Searching for weather in: {city}")
    return f"The weather in {city} is Sunny!"

# Initialize the model
model = ChatGroq(model_name="llama-3.3-70b-versatile").bind_tools([weather_search])

# Define state
class State(MessagesState):
    """State to track messages"""

# Call LLM
def call_llm(state):
    try:
        response = model.invoke(state["messages"])
        return {"messages": [response]}
    except Exception as e:
        logging.error(f"Error calling LLM: {str(e)}")
        raise HTTPException(status_code=500, detail="Error processing request")

# Human review node
def human_review_node(state) -> Command:
    last_message = state["messages"][-1]
    tool_call = last_message.tool_calls[-1]
    
    human_review = interrupt({
        "question": "Is this correct?",
        "tool_call": tool_call,
    })

    review_action = human_review["action"]
    review_data = human_review.get("data")

    if review_action == "continue":
        return Command(goto="run_tool")
    elif review_action == "update":
        updated_message = {
            "role": "ai",
            "content": last_message.content,
            "tool_calls": [{
                "id": tool_call["id"],
                "name": tool_call["name"],
                "args": review_data,
            }],
            "id": last_message.id,
        }
        return Command(goto="run_tool", update={"messages": [updated_message]})
    elif review_action == "feedback":
        tool_message = {
            "role": "tool",
            "content": review_data,
            "name": tool_call["name"],
            "tool_call_id": tool_call["id"],
        }
        return Command(goto="call_llm", update={"messages": [tool_message]})

# Run tool function
def run_tool(state):
    new_messages = []
    tools = {"weather_search": weather_search}
    tool_calls = state["messages"][-1].tool_calls

    for tool_call in tool_calls:
        tool = tools.get(tool_call["name"])
        if tool:
            result = tool.invoke(tool_call["args"])
            new_messages.append({
                "role": "tool",
                "name": tool_call["name"],
                "content": result,
                "tool_call_id": tool_call["id"],
            })
    return {"messages": new_messages}

# Routing function
def route_after_llm(state) -> str:
    if len(state["messages"][-1].tool_calls) == 0:
        return END
    else:
        return "human_review_node"

# Create the state graph
builder = StateGraph(State)
builder.add_node(call_llm)
builder.add_node(run_tool)
builder.add_node(human_review_node)
builder.add_edge(START, "call_llm")
builder.add_conditional_edges("call_llm", route_after_llm)
builder.add_edge("run_tool", "call_llm")

# Set up memory
memory = MemorySaver()
graph = builder.compile(checkpointer=memory)

# Request model for API
class RequestModel(BaseModel):
    messages: List[Dict[str, str]]
    thread_id: str

# POST API Endpoint for LLM chat
@app.post("/chat")
async def chat(request: RequestModel):
    """API to process messages using the AI model"""
    try:
        thread = {"configurable": {"thread_id": request.thread_id}}
        input_data = {"messages": request.messages}

        response_events = []
        for event in graph.stream(input_data, thread, stream_mode="updates"):
            response_events.append(event)

        return {"response": response_events}
    except Exception as e:
        logging.error(f"Error in /chat API: {str(e)}")
        raise HTTPException(status_code=500, detail="Internal Server Error")

# Run Uvicorn server (if running this file directly)
if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=53140)
```