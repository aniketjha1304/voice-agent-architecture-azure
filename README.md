Architecting Real-Time Voice Agents with Twilio, OpenAI Realtime, FastAPI, and Agent Builder
============================================================================================

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*IILdvPy8ANCT_Ayn755r7A.png)

[Reference](https://medium.com/@aniketjha1304/architecting-real-time-voice-agents-with-twilio-openai-realtime-fastapi-and-agent-builder-e2df8feb9375)

by [Aniket Jha](https://medium.com/?source=post_page---byline--e2df8feb9375---------------------------------------)






Voice is rapidly becoming the default interface for human-computer interaction. While speech recognition is a solved problem, building truly conversational, autonomous voice agents — ones that can act, call APIs, and end the call gracefully — is an engineering challenge.

The Twilio team made a brilliant leap integrating OpenAI Realtime into their telephony platform ([their reference implementation](https://www.twilio.com/code-exchange/ai-voice-assistant-openai-realtime-api) is an essential read). Building on top of that, we’ve architected a fully pluggable Voice Agent Framework for production:

*   Voice Agent composition.
*   Tool execution (so the agent can call APIs, end calls, escalate, etc.)
*   Explicit handling of telephony events

Below, I’ll show you how simple, robust, and extensible this setup can be using modern Python tools.

Real-Time Voice Architecture
----------------------------

This solution lets a user call your Twilio number and interact with an AI agent over the phone — one that can take action, respond naturally, and control the session.

Architecture Overview
---------------------

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*O6WzzYQKwxus6XnrU1g0rw.png)

Components in Context
---------------------

**User (Caller)**: Any phone, no app required.
**PSTN/SIP**: Brings the call to Twilio cloud.
**Twilio Edge**: Bridges the phone network to your backend, supports Media Streams.
**FastAPI Agent Engine**: Receives streams, orchestrates voice agent logic, and tools.
**OpenAI Realtime API**: Provides voice understanding and generation.

The Flow
--------

1.  User calls your Twilio number.
2.  Twilio hits your FastAPI endpoint (`/incoming-call`) for WebSocket setup instructions.
3.  FastAPI returns TwiML specifying a WebSocket URL (`/media-stream`).
4.  WebSocket opens: Twilio streams caller’s audio; FastAPI pipes this to your agent’s logic via Agent Builder.
5.  Agent interacts with OpenAI Realtime, receives intelligent responses.
6.  Audio response is streamed back (G.711, to match Twilio).
7.  The dialogue continues until the user or agent ends the call.

Why Agent Builder?
------------------

While the Twilio reference implementation shows how to connect the pieces, Agent Builder abstracts away the complexity of calling external APIs, managing conversational tools, and controlling call flow, in the same pattern you know from tool-based frameworks like LangChain (but focused on voice).
Refer: [https://github.com/Recall-Space/agent-builder](https://github.com/Recall-Space/agent-builder)

Setup
-----

Install everything:

```
pip install agent-builder fastapi[all] twilio openai uvicorn
```

Set your environment variables (`TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `OPENAI_API_KEY`, and other configs as needed).

Core FastAPI Handlers
---------------------

```
from fastapi import FastAPI, WebSocket, Request
from fastapi.responses import HTMLResponse
from twilio.twiml.voice_response import Connect, VoiceResponse
app = FastAPI()
@app.api_route("/incoming-call", methods=["POST"])
async def incoming_call(request: Request):
    host = request.url.hostname
    response = VoiceResponse()
    connect = Connect()
    connect.stream(url=f"wss://{host}/media-stream")
    response.append(connect)
    return HTMLResponse(str(response), media_type="application/xml")
```

Building the Agent: Prompt, Tools, and Orchestration
----------------------------------------------------

Agent Builder’s API makes it trivial to create specialized agents and attach “tools” that call APIs, databases, or even end the call.

Tool Example: Ending the Call
-----------------------------

```
from agent_builder.builders.tool_builder import ToolBuilder
from agent_builder.utils.call_end_exception import CallEndException
from pydantic import BaseModel
class EndCallSchema(BaseModel):
    """Schema for agent-initiated call termination."""
class EndCallHandler:
    def __init__(self, call_sid: str): self.call_sid = call_sid
    async def end_call(self) -> str:
        # You might use Twilio REST API to hang up here in production.
        raise CallEndException("End call triggered")
def create_end_call_tool(call_sid: str):
    handler = EndCallHandler(call_sid)
    tool = ToolBuilder()
    tool.set_name("end-call")
    tool.set_function(handler.end_call)
    tool.set_schema(EndCallSchema)
    tool.set_description(
        "Invoke this after confirming with the user that the conversation is complete."
    )
    return tool.build()
# ----------------------------
# Example Tool: FAQ Lookup
class FaqSchema(BaseModel):
    question: str = Field(description="User's question about the product or service.")
async def lookup_faq(question: str) -> str:
    # Stub: In production, perform a DB lookup, Elasticsearch query, etc.
    if "hours" in question.lower():
        return "Our support hours are 9am to 5pm, Monday through Friday."
    return "I'm sorry, I don't have an answer for that question."
def create_faq_tool():
    tool = ToolBuilder()
    tool.set_name("lookup-faq")
    tool.set_function(lookup_faq)
    tool.set_schema(FaqSchema)
    tool.set_description("Answer frequently asked questions about the service.")
    return tool.build()
```

Assemble the Agent with Streaming Voice
---------------------------------------

```
from agent_builder.builders.voice_agent_builder import VoiceAgentBuilder
OPENAI_REALTIME_API_URL = "https://your-openai-realtime-url"
OPENAI_REALTIME_API_KEY = "sk-..."
async def build_faq_voice_agent(
    call_sid: str,
    audio_format: str = "g711_ulaw",
    voice: str = "alloy"
):
    prompt = (
        "You are a professional AI voice assistant for a SaaS company. "
        "Answer FAQs, look up information, and end calls politely if requested."
    )
    # Tools: Add both FAQ lookup and end call tool
    tools = [        create_faq_tool(),
        create_end_call_tool(call_sid)
    ]
    # Compose the agent
    builder = (
        VoiceAgentBuilder()
        .set_api_key(OPENAI_REALTIME_API_KEY)
        .set_model_url(OPENAI_REALTIME_API_URL)
        .set_goal(prompt)
        .set_voice(voice)
        .set_input_audio_format(audio_format)
        .set_output_audio_format(audio_format)
        .set_tools(tools)
        .set_turn_detection({
            "type": "semantic_vad",
            "eagerness": "auto"
        })
    )
    return builder.build()
```

Streaming Audio & Handling Events
---------------------------------

A robust WebSocket loop hands audio and events between Twilio, your FastAPI app, and the agent:

```
import json
@app.websocket("/media-stream")
async def media_stream(websocket: WebSocket):
    await websocket.accept()
    call_sid = None
    try:
        while True:
            message = await websocket.receive_text()
            data = json.loads(message)
            if data.get("event") == "start":
                call_sid = data["start"]["callSid"]
                break
        agent = await build_faq_voice_agent(call_sid)
        async def input_audio_stream():
            # Initial handshake for OpenAI/agent
            yield '{"type": "response.create", ...}'
            async for msg in websocket.iter_text():
                yield msg
                if json.loads(msg).get("event") == "stop":
                    break
        async def handle_output_event(event_str: str):
            event = json.loads(event_str)
            if event.get("type") == "response.audio.delta":
                await websocket.send_text(json.dumps({
                    "event": "media",
                    "media": {"payload": event["delta"]},
                }))
            elif event.get("type") == "call.end":
                await websocket.close()
        await agent.ainvoke(input_audio_stream(), handle_output_event)
    except Exception as e:
        await websocket.close(code=1001, reason=f"Error: {str(e)}")
```

What Makes This Pattern Robust?
-------------------------------

*   Agent logic is purely declarative — define the goal and tools, not imperative flows.
*   Tools are drop-in actions the agent can take (API calls, escalation, end call, etc.).
*   Streaming & codecs handled using agent-builder and Twilio standards.
*   Clean call termination: The agent can end calls gracefully using a semantic tool, not side effects or callbacks.

Production Considerations
-------------------------

*   Use semantic VAD if supported for turn detection.
*   Store per-call logs and session state for compliance.
*   Always match audio codecs — Twilio expects G.711 µ-law.
*   Never leak stack traces to the user; surface only clean error responses.
*   Build in user opt-in/opt-out and account for privacy laws.

Final Thoughts
--------------

By abstracting the voice agent logic into composable, testable units — using agent-builder on top of a battle-tested Twilio + OpenAI streaming base — you can deliver voice experiences that are both powerful and maintainable. Your agent isn’t just a chatbot; it’s an API-enabled operator, acting with context and control.

Special thanks to the Twilio team for connecting OpenAI’s wakeful brains to the phone system, and to the open-source community advancing the state of conversational AI.

_Questions or want to contribute?_ [_Contact me_](mailto:aniket.jha@recall.space)_._
