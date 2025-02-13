FastAPI LLM Chatbot with LangGraph & Groq
This repository contains a FastAPI-based chatbot powered by LangGraph, Groq, and LangChain. It integrates a state-based AI workflow with human review, tool execution, and LLM-powered conversations.

🚀 Features
FastAPI Backend: Provides a REST API to process messages.
LangGraph Workflow: Implements a structured AI interaction flow.
Groq LLM: Uses Llama 3.3 70B for intelligent responses.
Tool Integration: Supports external tools like weather search.
Human Review System: Enables intervention in AI decision-making.
Memory Checkpoints: Saves conversation states using MemorySaver.
CORS Enabled: Allows cross-origin requests for frontend integration.
🛠️ Setup & Installation
1️⃣ Clone the Repository
git clone https://github.com/your-username/your-repo-name.git
cd your-repo-name
2️⃣ Install Dependencies
pip install -r requirements.txt
3️⃣ Set Environment Variables
export GROQ_API_KEY="your_groq_api_key"
export LANGCHAIN_API_KEY="your_langchain_api_key"
4️⃣ Run the FastAPI Server
uvicorn main:app --host 0.0.0.0 --port 53140 --reload
📡 API Endpoints
🔹 Chat with AI
Endpoint:

POST /chat
Request Body:

{
  "messages": [
    {"role": "user", "content": "What is the weather in New York?"}
  ],
  "thread_id": "12345"
}
Response:

{
  "response": [
    {"role": "tool", "name": "weather_search", "content": "The weather in New York is Sunny!"}
  ]
}

📌 Project Structure
graphql

📂 your-repo-name
 ├── main.py         # FastAPI application with LangGraph & Groq integration
 ├── requirements.txt # Required Python dependencies
 ├── README.md       # Project documentation
 
⚡ Tools & Technologies
FastAPI: Lightweight Python web framework.
LangGraph: AI state management with tools & human-in-the-loop control.
Groq LLM: Powered by Llama 3.3-70B model.
LangChain: Enables tool integration & memory management.
Uvicorn: ASGI server for running FastAPI.

🛡️ Security & Best Practices
Never hardcode API keys – use environment variables.
Restrict CORS in production (update allow_origins).
Validate user input to prevent security vulnerabilities.
