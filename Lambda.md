import base64
import json
import urllib.request
import boto3

def get_secret(secret_name):
    client = boto3.client("secretsmanager")
    resp = client.get_secret_value(SecretId=secret_name)
    val = resp.get("SecretString", "")
    try:
        return json.loads(val)
    except Exception:
        return val

def http_get(url, headers):
    req = urllib.request.Request(url, headers=headers, method="GET")
    with urllib.request.urlopen(req, timeout=20) as r:
        return json.loads(r.read().decode("utf-8"))

def lambda_handler(event, context):
    print("Received event:", json.dumps(event))

    # 1. Extract action from AgentCore Gateway context
    action = ""
    try:
        if context and context.client_context and context.client_context.custom:
            raw_tool = context.client_context.custom.get("bedrockAgentCoreToolName", "")
            action = raw_tool.split("___")[-1] if "___" in raw_tool else raw_tool
    except Exception as e:
        print("Context read error:", str(e))

    # 2. Fallback parameter detection if context is omitted
    if not action:
        if "filePath" in event or "filepath" in event:
            action = "readFile"
        elif "pageId" in event or "pageid" in event:
            action = "fetchConfluencePage"
        elif "query" in event:
            action = "searchConfluencePages"
        elif "repo" in event:
            action = "getFileTree"

    print(f"Routing to action: '{action}'")
    action_lower = action.lower()

    # 3. Retrieve Git Token
    git_token_data = get_secret("spectrace/git-token")
    if isinstance(git_token_data, dict):
        git_token = (
            git_token_data.get("token") 
            or git_token_data.get("git-token") 
            or list(git_token_data.values())[0]
        )
    else:
        git_token = git_token_data

    # 4. Handle Tool Actions (returning MCP format)
    try:
        if "getfiletree" in action_lower:
            repo = event.get("repo")
            branch = event.get("branch", "main")
            url = f"https://api.github.com/repos/{repo}/git/trees/{branch}?recursive=1"
            headers = {
                "Authorization": f"Bearer {git_token}",
                "Accept": "application/vnd.github.v3+json",
                "User-Agent": "SpecTrace-Agent"
            }
            res = http_get(url, headers)
            paths = [item["path"] for item in res.get("tree", []) if item.get("type") == "blob"]
            return {
                "content": [{"type": "text", "text": json.dumps({"files": paths[:800]})}]
            }

        elif "readfile" in action_lower:
            repo = event.get("repo")
            file_path = (event.get("filePath") or event.get("filepath", "")).lstrip("/")
            branch = event.get("branch", "main")
            url = f"https://api.github.com/repos/{repo}/contents/{file_path}?ref={branch}"
            headers = {
                "Authorization": f"Bearer {git_token}",
                "Accept": "application/vnd.github.v3+json",
                "User-Agent": "SpecTrace-Agent"
            }
            res = http_get(url, headers)
            file_content = base64.b64decode(res.get("content", "")).decode("utf-8", errors="replace")
            return {
                "content": [{"type": "text", "text": file_content}]
            }

        elif "searchconfluence" in action_lower:
            conf_creds = get_secret("spectrace/confluence-creds")
            auth_raw = f"{conf_creds['username']}:{conf_creds['api_token']}".encode("utf-8")
            auth_b64 = base64.b64encode(auth_raw).decode("utf-8")
            query = event.get("query", "")
            url = f"https://{conf_creds['domain']}/wiki/rest/api/content/search?cql=text~\"{query}\"&limit=10"
            headers = {
                "Authorization": f"Basic {auth_b64}",
                "Accept": "application/json"
            }
            res = http_get(url, headers)
            results = [{"id": p["id"], "title": p["title"]} for p in res.get("results", [])]
            return {
                "content": [{"type": "text", "text": json.dumps({"pages": results})}]
            }

        elif "fetchconfluence" in action_lower:
            conf_creds = get_secret("spectrace/confluence-creds")
            auth_raw = f"{conf_creds['username']}:{conf_creds['api_token']}".encode("utf-8")
            auth_b64 = base64.b64encode(auth_raw).decode("utf-8")
            page_id = event.get("pageId") or event.get("pageid")
            url = f"https://{conf_creds['domain']}/wiki/rest/api/content/{page_id}?expand=body.storage"
            headers = {
                "Authorization": f"Basic {auth_b64}",
                "Accept": "application/json"
            }
            res = http_get(url, headers)
            body = res.get("body", {}).get("storage", {}).get("value", "")
            return {
                "content": [{"type": "text", "text": body}]
            }

        else:
            return {
                "content": [{"type": "text", "text": f"Unknown tool: {action}"}],
                "isError": True
            }

    except Exception as e:
        return {
            "content": [{"type": "text", "text": f"Execution error: {str(e)}"}],
            "isError": True
        }
