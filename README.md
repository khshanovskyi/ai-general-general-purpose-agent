# General Purpose Agent

**Equipped with:**
- WEB Search (DuckDuckGo MCP Server. Performs WEB search and content fetching)
- Python Code Interpreter (MCP Server. A stateful Python code execution environment with Jupyter kernel support)
- Image Generation (ImageGen model within DIAL Core)
- File Content Extractor (Extract content from file (PDF, TXT, CSV). Supports basic pagination)
- RAG Search (Makes RAG search. Indexed files preserve during conversation in Cache)

## AFTER ALL THE TASKS DONE - DON'T FORGET TO REMOVE API KEYs FROM core/config.json

## Flow
<img src="agent_flow.png">

## Step 1
1. Provide implementation for the [agent.py](task/agent.py) according to TODO
2. Provide implementation for the [app.py](task/app.py) according to TODO. Tools initialization can be skipped, we will add them later
3. Create System prompt [prompts.py](task/prompts.py)

4. Configure [DIAL core config](core/config.json):
   - Add our Agent configuration to `applications`: 
     - Key is `general-purpose-agent` with such configurations:
       - displayName = `General Purpose Agent`
       - description = `eneral Purpose Agent. Equipped with: WEB search (DuckDuckGo via MCP), RAG search (supports PDF, TXT, CSV files), Python Code Interpreter (via MCP), Image Generation (model).`
       - endpoint = `http://host.docker.internal:5030/openai/deployments/general-purpose-agent/chat/completions`
       - inputAttachmentTypes = array of `image/png` and `image/jpeg`
       - forwardAuthToken = `true`
   - Add to `models`:
     - Key is `gpt-4o` with such configurations:
         - displayName = `GPT 4o`
         - endpoint = `http://adapter-dial-openai:5000/openai/deployments/gpt-4o/chat/completions`
         - iconUrl = `http://localhost:3001/gpt4.svg`
         - type = `chat`
         - upstreams = array with dict of:
            - endpoint = `https://api.openai.com/v1/chat/completions`
            - key = ${YOUR_OPENAI_API_KEY} 

5. Run [docker-compose](docker-compose.yml)
6. Start [app.py](task/app.py), open http://localhost:3000/ in browser and:
   - Firstly, in Marketplace check if General Purpose Agent is present and GPT 4o as well
   - Test that GPT 4o is working: `Hi, what can you do?`
   - Then test that General Purpose Agent is working: `Hi, what can you do?`

## Step 2
**Time to add first tool to extract content from files**
1. Provide implementation for the [tools/base.py](task/tools/base.py) according to TODO
2. Now is time to provide first tool implementation -> [tools/files/file_content_extraction_tool.py](task/tools/files/file_content_extraction_tool.py)
3. Add this tool to [app.py](task/app.py)
4. Add to [DIAL core config](core/config.json) new supported types `"application/pdf", "text/html", "text/plain", "text/csv"`
5. Restart [docker-compose](docker-compose.yml)
6. Test it:
   - `What can you do?` - should mention that have tool to extract file content
   - Attach [report.csv](tests/report.csv) and ask: `What is top sale for category A?` - should get file content and provide '1700 on 2025-10-05'
   - Attach [microwave_manual.txt](tests/microwave_manual.txt) and ask: `How should I clean the plate?` - should use pagination (2-3 tool calls) to fetch content and then should provide the response


## Step 3
**As you have seen full content extraction from files is not efficient, we need to add RAG search capability to enhance our Agent**
1. Provide implementation for the [tools/base.py](task/tools/rag/rag_tool.py) according to TODO
2. Add this tool to [app.py](task/app.py)
3. Test it:
   - Attach [microwave_manual.txt](tests/microwave_manual.txt) and ask: `How should I clean the plate?` - should call RAG tool. The main criteria here is that this tool will be called, usually it will try to call the `file_content_extraction_tool`, but after fetching 1st page and seeing that there are more paged it should call RAG tool, also user can indicate to model that RAG tool should be called
   - Pay attention that tool SYSTEM prompt matters, tool description as well. Configure it to achieve best result!

## Step 3
**Now let's add Image generation tool**

**ℹ️ In DIAL we name models and applications as deployments**

1. Add to [DIAL core config](core/config.json) dall-e-3 model:
   - Key is `dall-e-3` with such configurations:
     - displayName = `DALL-E-3`
     - endpoint = `http://adapter-dial-openai:5000/openai/deployments/dall-e-3/chat/completions`
     - iconUrl = `http://localhost:3001/gpt3.svg`
     - type = `chat`
     - upstreams = array with dict of:
     - endpoint = `https://api.openai.com/v1/images/generations`
     - key = {YOUR_OPENAI_API_KEY}
2. Provide implementation for the [tools/deployment/base.py](task/tools/deployment/base.py) according to TODO
3. Provide implementation for the [image_generation_tool.py](task/tools/deployment/image_generation_tool.py) according to TODO
4. Add this tool to [app.py](task/app.py)
5. **Optionally**, in the [tools/deployment/](task/tools/deployment) folder you can provide WEB search deployment tool, see it https://github.com/khshanovskyi/ai-simple-agent/blob/completed/task/tools/web_search.py
6. Restart [docker-compose](docker-compose.yml)
7. Test it:
   - `Generate picture with smiling cat` - expected result that you will see in stage all request parameters and attached revised prompt and picture, also, generated picture must be shown as content part in choice
   - If you created WEB search as deployment tool, test with such query: `Search what is the weather in Kyiv now and based on result generate picture that will represent it`

## Step 4
**Time to add WEB search (if you added as deployment tool - it is okay, but its not for free, we will make it for free)**

1. Add to [docker-compose](docker-compose.yml) new service:
    ```
    ddg-search:
      image: khshanovskyi/ddg-mcp-server:latest
      ports:
        - "8051:8000"
      environment:
        LOG_LEVEL: "INFO"
        MCP_TRANSPORT: "streamable-http"
      restart: unless-stopped
      mem_limit: 512M
      cpus: 0.5
    ```
    Sources: https://github.com/khshanovskyi/duckduckgo-mcp-server
2. Provide implementation for the [tools/deployment/mcp_client.py](task/tools/mcp/mcp_client.py) according to TODO
3. Provide implementation for the [tools/deployment/mcp_tool.py](task/tools/mcp/mcp_tool.py) according to TODO
4. Add all MCP tools from `http://localhost:8051/mcp` to [app.py](task/app.py) as MCPTool
5. Restart [docker-compose](docker-compose.yml)
6. Test it:
   - `Search what is the weather in Kyiv now`
   - `Who is Arkadiy Dobkin?`

## Step 5
**Time to add Code Interpreter**

LLMs cannot perform real calculations, they are 'assuming'. Also, sometimes we need to analyze data щк make some chart. 
Because of that ChatGPT has Python Code Interpreter that is running on the ChatGPT side, in Claude it is JS Code Interpreter that is running in browser.
We will try to implement similar to ChatGPT Python Code Interpreter tool. 

We will use already implemented [PyInterpreter MCP Server](https://github.com/khshanovskyi/mcp-python-code-interpreter/tree/main). 
Flow: <img src="py_interpreter_flow.png">

1. Add to [docker-compose](docker-compose.yml) new service:
    ```
      python-interpreter:
        image: khshanovskyi/python-code-interpreter-mcp-server:latest
        ports:
          - "8050:8000"
        environment:
          LOG_LEVEL: "INFO"
        restart: unless-stopped
        mem_limit: 2G
        cpus: 2.0
    ```
2. Provide implementation for the [tools/deployment/mcp_tool.py](task/tools/mcp/interpreter/python_code_interpreter_tool.py) according to TODO
3. Add Python Code Interpreter tool to [app.py](task/app.py), url http://localhost:8050/mc
4. Restart [docker-compose](docker-compose.yml)
5. Test it:
    - `What is the sin of 5682936329203?` Should call PyInterpreter and show result
    - Attach [report.csv](tests/report.csv) and ask: `I need chart bar from this data` - should get file content and then call PyInterpreter, in response should be generated file as attachment that will be able to see
---

## AFTER ALL THE TASKS DONE - DON'T FORGET TO REMOVE API KEYs FROM core/config.json

Congratulate you, that is all with General Purpose Agent 🎉
