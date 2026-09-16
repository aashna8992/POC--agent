# POC--agent
SpecTrace
# Store Git Token
aws secretsmanager create-secret \
  --name "spectrace/git-token" \
  --secret-string '{"token":"<GIT_PAT>"}'

# Store Confluence Credentials
aws secretsmanager create-secret \
  --name "spectrace/confluence-creds" \
  --secret-string '{"email":"<YOUR_EMAIL>","token":"<CONFLUENCE_TOKEN>","domain":"<YOUR_DOMAIN>.atlassian.net"}'


————-
import json
import urllib.request
import boto3

def search_confluence(query):
    # Confluence CQL search endpoint
    url = f"https://{CONFLUENCE_DOMAIN}/wiki/rest/api/search?cql=text~\"{query}\"&limit=5"
    res = requests.get(url, auth=(USER, TOKEN))
    results = res.json().get("results", [])
    return [
        {"id": item["content"]["id"], "title": item["content"]["title"]}
        for item in results
    ]


def get_secret(secret_name):
    client = boto3.client('secretsmanager')
    res = client.get_secret_value(SecretId=secret_name)
    return json.loads(res['SecretString'])

def make_github_request(url, token):
    req = urllib.request.Request(url)
    req.add_header('Authorization', f'Bearer {token}')
    req.add_header('Accept', 'application/vnd.github.v3+json')
    req.add_header('User-Agent', 'SpecTrace-PoC')
    with urllib.request.urlopen(req) as response:
        return json.loads(response.read().decode())

def make_confluence_request(endpoint, creds):
    url = f"https://{creds['domain']}/wiki/rest/api/{endpoint}"
    req = urllib.request.Request(url)
    import base64
    auth = base64.b64encode(f"{creds['email']}:{creds['token']}".encode()).decode()
    req.add_header('Authorization', f'Basic {auth}')
    req.add_header('Accept', 'application/json')
    with urllib.request.urlopen(req) as response:
        return json.loads(response.read().decode())

def lambda_handler(event, context):
    action_group = event.get('actionGroup')
    api_path = event.get('apiPath')
    parameters = {p['name']: p['value'] for p in event.get('parameters', [])}
    
    git_token = get_secret('spectrace/git-token')['token']
    confluence_creds = get_secret('spectrace/confluence-creds')

    result_payload = {}

    if api_path == '/fetch-confluence-page':
        page_id = parameters.get('pageId')
        data = make_confluence_request(f"content/{page_id}?expand=body.storage", confluence_creds)
        result_payload = {
            "title": data.get('title'),
            "html_content": data.get('body', {}).get('storage', {}).get('value', '')[:10000] # trim to fit
        }

    elif api_path == '/get-file-tree':
        repo = parameters.get('repo')
        branch = parameters.get('branch', 'main')
        url = f"https://api.github.com/repos/{repo}/git/trees/{branch}?recursive=1"
        data = make_github_request(url, git_token)
        # Return file paths only, ignoring binaries or node_modules
        files = [item['path'] for item in data.get('tree', []) 
                 if item['type'] == 'blob' and not item['path'].startswith(('node_modules/', '.git/'))]
        result_payload = {"files": files[:300]}

    elif api_path == '/read-file':
        repo = parameters.get('repo')
        branch = parameters.get('branch', 'main')
        path = parameters.get('filePath')
        url = f"https://api.github.com/repos/{repo}/contents/{path}?ref={branch}"
        data = make_github_request(url, git_token)
        import base64
        content = base64.b64decode(data.get('content', '')).decode('utf-8', errors='ignore')
        result_payload = {"content": content[:25000]} # Cap for LLM context limits

    response_body = {
        'application/json': {
            'body': json.dumps(result_payload)
        }
    }

    return {
        'messageVersion': '1.0',
        'response': {
            'actionGroup': action_group,
            'apiPath': api_path,
            'httpMethod': event.get('httpMethod'),
            'httpStatusCode': 200,
            'responseBody': response_body
        }
    }


—————-

{
  "openapi": "3.0.0",
  "info": {
    "title": "SpecTrace Code and Doc Inspector API",
    "version": "1.0.0"
  },
  "paths": {
    "/fetch-confluence-page": {
      "get": {
        "summary": "Fetches Confluence PRD content by page ID",
        "operationId": "fetchConfluencePage",
        "parameters": [
          { "name": "pageId", "in": "query", "required": true, "schema": { "type": "string" } }
        ],
        "responses": { "200": { "description": "Confluence page text" } }
      }
    },
    "/get-file-tree": {
      "get": {
        "summary": "Lists all file paths in a given repository and branch",
        "operationId": "getFileTree",
        "parameters": [
          { "name": "repo", "in": "query", "required": true, "schema": { "type": "string" } },
          { "name": "branch", "in": "query", "required": false, "schema": { "type": "string" } }
        ],
        "responses": { "200": { "description": "List of repository file paths" } }
      }
    },
    "/read-file": {
      "get": {
        "summary": "Reads the exact text content of a single file from Git",
        "operationId": "readFile",
        "parameters": [
          { "name": "repo", "in": "query", "required": true, "schema": { "type": "string" } },
          { "name": "branch", "in": "query", "required": false, "schema": { "type": "string" } },
          { "name": "filePath", "in": "query", "required": true, "schema": { "type": "string" } }
        ],
        "responses": { "200": { "description": "Raw file text content" } }
      }
    }
  }
}

—-
[
  {
    "name": "fetchConfluencePage",
    "description": "Fetches Confluence PRD content by page ID",
    "inputSchema": {
      "type": "object",
      "properties": {
        "pageId": {
          "type": "string",
          "description": "The Confluence page ID"
        }
      },
      "required": ["pageId"]
    }
  },
  {
    "name": "getFileTree",
    "description": "Lists all file paths in a given repository and branch",
    "inputSchema": {
      "type": "object",
      "properties": {
        "repo": {
          "type": "string",
          "description": "The GitHub repo in 'owner/name' format"
        },
        "branch": {
          "type": "string",
          "description": "Branch name (default: main)"
        }
      },
      "required": ["repo"]
    }
  },
  {
    "name": "readFile",
    "description": "Reads the exact text content of a single code or SQL file from Git",
    "inputSchema": {
      "type": "object",
      "properties": {
        "repo": {
          "type": "string",
          "description": "The GitHub repo in 'owner/name' format"
        },
        "filePath": {
          "type": "string",
          "description": "Full path to the file in the repository"
        },
        "branch": {
          "type": "string",
          "description": "Branch name (default: main)"
        }
      },
      "required": ["repo", "filePath"]
    }
  }
]
————————
———-

You are an expert fintech software architect. Your job is to reverse-engineer features across Confluence, Git repositories, and Database scripts.

When given a feature request with a Confluence page ID and repository names:
1. Fetch the Confluence page using fetchConfluencePage to understand user requirements and domain terms.
2. Call getFileTree on the provided repositories to locate relevant controllers, route handlers, models, and SQL migration files.
3. Call readFile to inspect exact code logic, validations, and database procedure calls.
4. Synthesize your findings into a 4-part dossier:
   - Section 1: Business Story (Clear narrative of what the feature does, who uses it, business rules).
   - Section 2: Visual Data Flow (Mermaid sequence diagram starting from entry trigger: UI/Cron/Webhook to DB).
   - Section 3: Technical Catalog (Inventory of APIs, Step Functions, Database Names, Tables, Views, Stored Procedures).
   - Section 4: Gap & Ambiguity Ledger (MANDATORY: If a queue consumer, table, downstream API, or procedure was referenced but could not be located in the files, mark it as UNKNOWN and explicitly state the missing repository or dependency required to resolve it).

CRITICAL RULE: Never guess or hallucinate database tables or downstream consumers. If you cannot find the definition in the files, categorize it as UNKNOWN.
