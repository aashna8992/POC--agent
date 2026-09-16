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




            
