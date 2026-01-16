---
title: "Building a Custom MCP Server to Connect Gemini CLI with n8n for Workflow Automation"
date: 2026-01-16
draft: false
tags: 
  - n8n
  - automation
  - mcp
  - gemini-cli
  - python
  - traefik
  - self-hosted
  - homelab
  - api-integration
  - workflow-automation
categories:
  - Automation
  - Self-Hosted
  - Development
description: "Learn how to integrate Google's Gemini CLI with n8n using a custom Python MCP server to enable full workflow automation capabilities, bypassing n8n's read-only MCP limitations."
---

## Introduction

In this post, I'll walk you through my journey of integrating Google's Gemini CLI (v0.24.0) with my self-hosted n8n instance (v2.2.6) using the Model Context Protocol (MCP). While n8n provides built-in MCP support, it only exposes read-only actions. To unlock full workflow automation—creating, modifying, and deleting workflows via Gemini CLI—I built a custom Python MCP server that leverages n8n's API.

## Project Overview

**Goal:** Enable Gemini CLI to create, read, update, and delete n8n workflows programmatically.

**Challenge:** n8n's native MCP server only supports read operations, preventing workflow creation and modification.

**Solution:** Build a custom Python-based MCP server that uses n8n's API to provide full CRUD (Create, Read, Update, Delete) capabilities.

## Architecture

```text
Gemini CLI → Custom Python MCP Server → n8n API → n8n Instance (Docker)
                                              ↓
                                      Traefik Reverse Proxy
```

## Setup Steps

### 1. Enable n8n MCP and API Access

First, I enabled the necessary features in my n8n instance:

**Instance-level MCP Server:**
- This provided the MCP server URL and authentication key
- However, it only exposed read-only actions

**n8n API Access:**
- Enabled API access in n8n settings
- Generated an API key for authentication
- API endpoint: `https://n8n2.local.indushomelab.net/api/v1`

### 2. Configure Traefik Routing

My n8n instance runs behind Traefik as a reverse proxy. I had to update the Traefik configuration to allow the necessary API endpoints:

```yaml
n8n2:
  entryPoints:
    - "https"
  rule: "Host(`n8n2.local.indushomelab.net`) || (Host(`n8n2.local.indushomelab.net`) && PathPrefix(`/webhook`)) || (Host(`n8n2.local.indushomelab.net`) && PathPrefix(`/webhook-test`)) || (Host(`n8n2.local.indushomelab.net`) && PathPrefix(`/mcp-server/http`)) || (Host(`n8n2.local.indushomelab.net`) && PathPrefix(`/api/v1`))"
  tls: {}
  service: n8n2
```

**Key paths added:**
- `/api/v1` - n8n API access
- `/mcp-server/http` - n8n's built-in MCP server
- `/webhook` - Webhook endpoints
- `/webhook-test` - Testing webhooks

All routes use `PathPrefix` matching to ensure proper request forwarding.

### 3. Build the Python MCP Server

I created a custom MCP server using Python and the `fastmcp` library:

**Installation:**
```bash
python -m venv n8n-mcp-venv
source n8n-mcp-venv/bin/activate
pip install mcp requests
```

**Server Implementation (`server.py`):**

```python
import requests
from mcp.server.fastmcp import FastMCP

N8N_URL = "https://n8n2.local.indushomelab.net/api/v1"
N8N_API_KEY = "your_api_key_here"

mcp = FastMCP("n8n-mcp-python")

@mcp.tool()
def create_workflow(name: str, workflow: dict) -> dict:
    """Create a new n8n workflow"""
    body = {
        "name": name,
        "nodes": workflow.get("nodes", []),
        "connections": workflow.get("connections", {}),
        "settings": workflow.get("settings", {})
    }
    
    workflow.pop("active", None)
    
    response = requests.post(
        f"{N8N_URL}/workflows",
        headers={"X-N8N-API-KEY": N8N_API_KEY},
        json=body
    )
    if not response.ok:
        raise Exception(f"Failed: {response.text}")
    
    data = response.json()
    return {"workflow_id": data["id"]}

@mcp.tool()
def update_workflow(workflow_id: str, workflow: dict) -> dict:
    """Update an existing n8n workflow"""
    workflow.pop("active", None)
    
    response = requests.patch(
        f"{N8N_URL}/workflows/{workflow_id}",
        headers={"X-N8N-API-KEY": N8N_API_KEY},
        json=workflow
    )
    if not response.ok:
        raise Exception(f"Update failed: {response.text}")
    
    return response.json()

@mcp.tool()
def delete_workflow(workflow_id: str) -> dict:
    """Delete a workflow by ID"""
    response = requests.delete(
        f"{N8N_URL}/workflows/{workflow_id}",
        headers={"X-N8N-API-KEY": N8N_API_KEY}
    )
    if not response.ok:
        raise Exception(f"Delete failed: {response.text}")
    
    return {"message": "Workflow deleted"}

@mcp.tool()
def search_workflows(query: str) -> list:
    """Search workflows by name"""
    response = requests.get(
        f"{N8N_URL}/workflows",
        headers={"X-N8N-API-KEY": N8N_API_KEY}
    )
    workflows = response.json()
    filtered = [w for w in workflows if query.lower() in w["name"].lower()]
    return filtered

@mcp.tool()
def get_workflow_details(workflow_id: str) -> dict:
    """Get workflow details"""
    response = requests.get(
        f"{N8N_URL}/workflows/{workflow_id}",
        headers={"X-N8N-API-KEY": N8N_API_KEY}
    )
    if not response.ok:
        raise Exception("Workflow not found")
    
    return response.json()

@mcp.tool()
def execute_workflow(workflow_id: str, payload: dict = None) -> dict:
    """Execute workflow"""
    if payload is None:
        payload = {}
    
    response = requests.post(
        f"{N8N_URL}/workflows/{workflow_id}/execute",
        headers={"X-N8N-API-KEY": N8N_API_KEY},
        json=payload
    )
    if not response.ok:
        raise Exception(f"Execution failed: {response.text}")
    
    return response.json()

if __name__ == "__main__":
    import sys
    
    if len(sys.argv) > 1 and sys.argv[1] == "sse":
        mcp.run(transport="sse")
    else:
        mcp.run()
```

### 4. Configure Gemini CLI

Updated the Gemini CLI configuration file (`config.json`) to include both MCP servers:

```json
{
  "mcpServers": {
    "n8n-readonly": {
      "url": "https://n8n2.local.indushomelab.net/mcp-server/http",
      "headers": {
        "Authorization": "Bearer <n8n_mcp_auth_key>"
      }
    },
    "n8n-custom": {
      "command": "/home/indunil/n8n-mcp-venv/bin/python",
      "args": ["/mnt/storage/n8n_mcp_server/server.py"]
    }
  }
}
```

## Problems Faced and Solutions

### Problem 1: Read-Only MCP Server

**Issue:** n8n's native MCP server only exposes read operations.

**Solution:** Built a custom Python MCP server using n8n's REST API to enable full CRUD operations.

### Problem 2: MCP Server API Compatibility

**Issue:** Initial implementation used outdated `CallToolResult` objects and `schema` parameters, which caused errors with the latest FastMCP library.

**Error:**
```
TypeError: FastMCP.tool() got an unexpected keyword argument 'schema'
```

**Solution:** Updated to use Python type hints and direct return values instead of `CallToolResult` objects:

```python
# Old (broken)
@mcp.tool(schema={...})
def my_tool(input):
    return CallToolResult(output={...})

# New (working)
@mcp.tool()
def my_tool(param: str) -> dict:
    return {...}
```

### Problem 3: Traefik Routing

**Issue:** API endpoints `/api/v1`, `/mcp-server/http`, and `/webhook` returned 404 errors.

**Solution:** Added explicit PathPrefix rules in Traefik configuration to route these paths to the n8n service.

### Problem 4: Outdated Node Versions

**Issue:** Gemini CLI generated workflows using outdated n8n node versions with deprecated configuration options.

**Affected nodes:**
- HTTP Request
- Merge
- OpenAI
- And others

**Solution:** Manually updated node configurations within the n8n interface after workflow creation. This required reviewing each generated workflow and updating deprecated parameters to their current equivalents.

## Use Case Example

I created a workflow that automates trend extraction and social media content creation:

**Workflow Purpose:** 
This automation eliminates manual trend research and content generation by:
1. Fetching current trends from various sources
2. Scoring trends using AI (OpenAI)
3. Generating tailored content
4. Posting to multiple social media platforms

**Target Audience:**
- Social media managers
- Digital marketers
- Businesses looking to streamline content strategies

**Simple Test Workflow:**
I also created a basic "Hello World" workflow to verify the integration was working correctly.

## Technical Environment

- **n8n Version:** 2.2.6
- **Gemini CLI Version:** 0.24.0
- **Deployment:** Self-hosted Docker container on Ubuntu
- **Reverse Proxy:** Traefik
- **MCP Server:** Python 3.x with FastMCP library

## Key Learnings

1. **MCP Protocol Flexibility:** The Model Context Protocol allows building custom servers to extend capabilities beyond what native integrations provide.

2. **API-First Approach:** When a platform's MCP server is limited, falling back to REST APIs is a viable workaround.

3. **Type Hints Matter:** Modern FastMCP relies on Python type hints for automatic schema generation, making code cleaner and more maintainable.

4. **Reverse Proxy Configuration:** Don't forget to update your reverse proxy rules when exposing new API endpoints.

5. **Version Compatibility:** AI-generated workflows may use outdated node versions, requiring manual review and updates.

## Conclusion

By combining n8n's REST API with a custom Python MCP server, I successfully enabled Gemini CLI to create, modify, and delete workflows programmatically. While there were challenges with API compatibility and node versions, the end result is a powerful automation setup that lets me describe workflows in natural language and have them automatically created in n8n.

This approach can be adapted for other platforms that have limited MCP support but expose comprehensive REST APIs.

## Resources

- [n8n Documentation](https://docs.n8n.io/)
- [Model Context Protocol](https://modelcontextprotocol.io/)
- [FastMCP Library](https://github.com/jlowin/fastmcp)
- [Gemini CLI](https://ai.google.dev/gemini-api/docs/cli)

---

*Have you integrated AI assistants with your automation tools? Share your experiences in the comments below!*
