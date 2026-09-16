    # Extract action name from AgentCore Gateway context or payload
    action = ""
    try:
        if context and context.client_context and context.client_context.custom:
            raw_tool = context.client_context.custom.get("bedrockAgentCoreToolName", "")
            action = raw_tool.split("___")[-1] if "___" in raw_tool else raw_tool
    except Exception:
        pass

    if not action:
        if "filePath" in event or "filepath" in event:
            action = "readFile"
        elif "pageId" in event or "pageid" in event:
            action = "fetchConfluencePage"
        elif "query" in event:
            action = "searchConfluencePages"
        elif "repo" in event:
            action = "getFileTree"

    action_lower = action.lower()


———
return {"content": [{"type": "text", "text": json.dumps({"files": paths[:800]})}]}

—-
return {"content": [{"type": "text", "text": file_content}]}

—-
return {"content": [{"type": "text", "text": json.dumps({"pages": results})}]}
—-
return {"content": [{"type": "text", "text": body}]}
—-
return {"content": [{"type": "text", "text": f"Unknown tool: {action}"}], "isError": True}

——-
