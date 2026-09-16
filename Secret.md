    # Retrieve and safely unwrap Git Token
    git_secret_raw = get_secret("spectrace/git-token")
    if isinstance(git_secret_raw, dict):
        git_token = git_secret_raw.get("token", "").strip()
    else:
        try:
            parsed = json.loads(git_secret_raw)
            git_token = parsed.get("token", "").strip()
        except Exception:
            git_token = str(git_secret_raw).strip()


            ———
                headers = {
        "Authorization": f"Bearer {git_token}",
        "Accept": "application/vnd.github.v3+json",
        "User-Agent": "SpecTrace-Agent"
    }



————
confluence 

————
    elif "searchconfluence" in action_lower:
        import urllib.parse

        # 1. Safely parse credentials into a dictionary
        conf_creds_raw = get_secret("spectrace/confluence-creds")
        if isinstance(conf_creds_raw, str):
            try:
                conf_creds = json.loads(conf_creds_raw)
            except Exception:
                conf_creds = {}
        else:
            conf_creds = conf_creds_raw

        # 2. Extract values safely
        username = conf_creds.get("username") or conf_creds.get("email") or ""
        api_token = conf_creds.get("api_token") or conf_creds.get("token") or ""
        domain = conf_creds.get("domain", "").replace("https://", "").rstrip("/")

        # 3. Build Basic Auth
        auth_raw = f"{username}:{api_token}".encode("utf-8")
        auth_b64 = base64.b64encode(auth_raw).decode("utf-8")

        # 4. Clean and encode query
        query = event.get("query", "").strip()
        cql_raw = f'type=page and (title ~ "{query}" or text ~ "{query}")'
        cql_encoded = urllib.parse.quote(cql_raw)

        url = f"https://{domain}/wiki/rest/api/content/search?cql={cql_encoded}&limit=10"
        headers = {
            "Authorization": f"Basic {auth_b64}",
            "Accept": "application/json"
        }

        try:
            res = http_get(url, headers)
            results = [
                {"id": p.get("id"), "title": p.get("title")} 
                for p in res.get("results", [])
            ]
            return {
                "content": [{"type": "text", "text": json.dumps({"pages": results})}]
            }
        except Exception as e:
            return {
                "content": [{"type": "text", "text": f"Confluence search error: {str(e)}"}],
                "isError": True
            }


————-fetch confluence———-

    elif "fetchconfluence" in action_lower:
        # 1. Unpack credentials safely to avoid string index errors
        creds = get_secret("spectrace/confluence-creds")
        if isinstance(creds, str):
            try:
                creds = json.loads(creds)
                if isinstance(creds, str):
                    creds = json.loads(creds)
            except Exception:
                creds = {}

        username = creds.get("username") or creds.get("email", "")
        api_token = creds.get("api_token") or creds.get("token", "")
        domain = creds.get("domain", "").replace("https://", "").rstrip("/")

        # 2. Build Basic Auth
        auth_raw = f"{username}:{api_token}".encode("utf-8")
        auth_b64 = base64.b64encode(auth_raw).decode("utf-8")

        # 3. Extract pageId safely
        page_id = event.get("pageId") or event.get("pageid") or event.get("id", "")

        url = f"https://{domain}/wiki/rest/api/content/{page_id}?expand=body.storage"
        headers = {
            "Authorization": f"Basic {auth_b64}",
            "Accept": "application/json"
        }

        try:
            res = http_get(url, headers)
            # Safely navigate dictionary structure
            body = ""
            if isinstance(res, dict):
                body = res.get("body", {}).get("storage", {}).get("value", "")

            return {
                "content": [{"type": "text", "text": body}]
            }
        except Exception as e:
            return {
                "content": [{"type": "text", "text": f"Confluence fetch error: {str(e)}"}],
                "isError": True
            }





            
