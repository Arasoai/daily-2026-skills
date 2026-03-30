```markdown
---
name: leak-check-personal-info
description: Personal information leak detection API interface for checking if data has been exposed in breaches
triggers:
  - check if my data was leaked
  - personal information leak detection
  - use leak-check API
  - detect data breach exposure
  - install leak-check tool
  - check email in data breach
  - query leaked personal information
  - run leak check scan
---

# leak-check — Personal Information Leak Detection

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

## What It Does

`leak-check` is a personal information "leak" detection interface (`个人信息"泄漏"检测接口`) that allows you to query whether personal data (emails, phone numbers, usernames, etc.) has appeared in known data breaches or leaks. It provides a hosted API at [leak-check.garinasset.com](https://leak-check.garinasset.com) and a local installable tool.

## Installation

### One-line Install (recommended)

```bash
curl -LsSf https://raw.githubusercontent.com/garinasset/leak-check/refs/heads/main/install.sh | bash
```

### From Source

```bash
git clone https://github.com/garinasset/leak-check.git
cd leak-check
pip install -r requirements.txt
```

### Python Package

```bash
pip install requests  # primary dependency
```

## Configuration

Set your API credentials via environment variables:

```bash
export LEAK_CHECK_API_KEY="your_api_key_here"
export LEAK_CHECK_BASE_URL="https://leak-check.garinasset.com"
```

Or create a `.env` file:

```ini
LEAK_CHECK_API_KEY=your_api_key_here
LEAK_CHECK_BASE_URL=https://leak-check.garinasset.com
```

## API Usage

### Basic Query — Check an Email

```python
import os
import requests

API_KEY = os.environ["LEAK_CHECK_API_KEY"]
BASE_URL = os.environ.get("LEAK_CHECK_BASE_URL", "https://leak-check.garinasset.com")

def check_email(email: str) -> dict:
    """Check if an email address appears in known data leaks."""
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json",
    }
    response = requests.post(
        f"{BASE_URL}/api/check",
        json={"query": email, "type": "email"},
        headers=headers,
        timeout=10,
    )
    response.raise_for_status()
    return response.json()

result = check_email("user@example.com")
print(result)
```

### Check Phone Number

```python
def check_phone(phone: str) -> dict:
    """Check if a phone number appears in known data leaks."""
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json",
    }
    response = requests.post(
        f"{BASE_URL}/api/check",
        json={"query": phone, "type": "phone"},
        headers=headers,
        timeout=10,
    )
    response.raise_for_status()
    return response.json()

result = check_phone("+8613800138000")
print(result)
```

### Check Username

```python
def check_username(username: str) -> dict:
    """Check if a username appears in known data leaks."""
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json",
    }
    response = requests.post(
        f"{BASE_URL}/api/check",
        json={"query": username, "type": "username"},
        headers=headers,
        timeout=10,
    )
    response.raise_for_status()
    return response.json()
```

## LeakCheck Client Class

```python
import os
import requests
from typing import Literal, Optional

QueryType = Literal["email", "phone", "username", "ip"]

class LeakCheckClient:
    """Client for the leak-check personal information detection API."""

    def __init__(
        self,
        api_key: Optional[str] = None,
        base_url: Optional[str] = None,
        timeout: int = 10,
    ):
        self.api_key = api_key or os.environ["LEAK_CHECK_API_KEY"]
        self.base_url = (
            base_url
            or os.environ.get("LEAK_CHECK_BASE_URL", "https://leak-check.garinasset.com")
        ).rstrip("/")
        self.timeout = timeout
        self.session = requests.Session()
        self.session.headers.update(
            {
                "Authorization": f"Bearer {self.api_key}",
                "Content-Type": "application/json",
            }
        )

    def check(self, query: str, query_type: QueryType = "email") -> dict:
        """
        Check if a piece of personal information appears in known leaks.

        Args:
            query: The value to search for (email, phone, username, etc.)
            query_type: The type of data being queried

        Returns:
            dict with leak results
        """
        response = self.session.post(
            f"{self.base_url}/api/check",
            json={"query": query, "type": query_type},
            timeout=self.timeout,
        )
        response.raise_for_status()
        return response.json()

    def bulk_check(self, items: list[dict]) -> list[dict]:
        """
        Check multiple items in one request.

        Args:
            items: List of dicts with 'query' and 'type' keys

        Returns:
            List of results
        """
        response = self.session.post(
            f"{self.base_url}/api/bulk-check",
            json={"items": items},
            timeout=self.timeout,
        )
        response.raise_for_status()
        return response.json()

    def get_sources(self) -> list[dict]:
        """Retrieve the list of breach databases being checked."""
        response = self.session.get(
            f"{self.base_url}/api/sources",
            timeout=self.timeout,
        )
        response.raise_for_status()
        return response.json()


# Usage
client = LeakCheckClient()

# Single check
result = client.check("user@example.com", "email")
if result.get("found"):
    print(f"Found in {result.get('sources', [])} breach(es)")
else:
    print("No leaks detected")

# Bulk check
results = client.bulk_check([
    {"query": "user@example.com", "type": "email"},
    {"query": "+8613800138000", "type": "phone"},
])
```

## CLI Usage

After installation via the install script:

```bash
# Check an email
leak-check email user@example.com

# Check a phone number
leak-check phone +8613800138000

# Check a username
leak-check username johndoe

# Output as JSON
leak-check email user@example.com --json

# Use a specific API key
leak-check email user@example.com --api-key $LEAK_CHECK_API_KEY
```

## Handling API Responses

```python
def process_leak_result(result: dict) -> None:
    """Parse and display leak check results."""
    if not result.get("found", False):
        print("✅ No leaks detected")
        return

    print(f"⚠️  Found in {result.get('count', 0)} breach(es)")

    for source in result.get("sources", []):
        print(f"  - Source: {source.get('name', 'Unknown')}")
        print(f"    Date: {source.get('date', 'Unknown')}")
        print(f"    Fields: {', '.join(source.get('fields', []))}")


client = LeakCheckClient()
result = client.check("test@example.com", "email")
process_leak_result(result)
```

## Error Handling

```python
from requests.exceptions import HTTPError, ConnectionError, Timeout

def safe_check(client: LeakCheckClient, query: str, query_type: str) -> dict | None:
    try:
        return client.check(query, query_type)
    except HTTPError as e:
        if e.response.status_code == 401:
            print("Invalid API key — check LEAK_CHECK_API_KEY env var")
        elif e.response.status_code == 429:
            print("Rate limit exceeded — slow down requests")
        elif e.response.status_code == 400:
            print(f"Bad request: {e.response.json().get('message', 'unknown error')}")
        else:
            print(f"API error: {e}")
        return None
    except ConnectionError:
        print("Cannot reach leak-check server — check your network or BASE_URL")
        return None
    except Timeout:
        print("Request timed out — server may be slow")
        return None
```

## Batch Processing with Rate Limiting

```python
import time

def batch_check_emails(
    client: LeakCheckClient,
    emails: list[str],
    delay: float = 0.5,
) -> list[dict]:
    """Check a list of emails with rate limiting."""
    results = []
    for email in emails:
        result = safe_check(client, email, "email")
        if result is not None:
            results.append({"query": email, "result": result})
        time.sleep(delay)  # respect rate limits
    return results


emails = ["alice@example.com", "bob@example.com", "charlie@example.com"]
client = LeakCheckClient()
all_results = batch_check_emails(client, emails)

leaked = [r for r in all_results if r["result"].get("found")]
print(f"{len(leaked)}/{len(emails)} emails found in breaches")
```

## Integration with FastAPI

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, EmailStr
import os

app = FastAPI()
client = LeakCheckClient()


class CheckRequest(BaseModel):
    query: str
    type: str = "email"


@app.post("/check-leak")
async def check_leak(req: CheckRequest):
    try:
        result = client.check(req.query, req.type)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

## Troubleshooting

### "401 Unauthorized"
- Verify `LEAK_CHECK_API_KEY` is set and correct
- Check the API key hasn't expired on the dashboard

### "Connection refused" or DNS error
- Confirm `LEAK_CHECK_BASE_URL` points to the correct server
- Default: `https://leak-check.garinasset.com`

### "429 Too Many Requests"
- Add delays between requests (`time.sleep(1)`)
- Use the bulk endpoint for multiple queries

### Install script fails
```bash
# Manual install fallback
git clone https://github.com/garinasset/leak-check.git
cd leak-check
python -m pip install -r requirements.txt
python main.py --help
```

### SSL Certificate errors
```python
# Only for development/testing — never in production
response = requests.post(url, json=data, verify=False)
```

## Common Patterns

```python
# Pattern: Audit a list of employees' emails
client = LeakCheckClient()
employee_emails = load_emails_from_csv("employees.csv")  # your function

report = []
for email in employee_emails:
    result = client.check(email, "email")
    if result.get("found"):
        report.append({
            "email": email,
            "breaches": result.get("count", 0),
            "sources": result.get("sources", []),
        })

print(f"Security report: {len(report)} exposed accounts found")
```
```
