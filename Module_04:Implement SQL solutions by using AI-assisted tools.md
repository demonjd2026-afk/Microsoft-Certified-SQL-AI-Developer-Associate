# DP-800 Exam Notes — Implement SQL Solutions by Using AI-Assisted Tools

> **Module:** Implement SQL solutions by using AI-assisted tools
> **Level:** Intermediate | **Roles:** DBA, Developer, Data Engineer
> **Platforms Covered:** SQL Server, Azure SQL Database, Azure SQL Managed Instance, Microsoft Fabric

---

## Module Overview

This module covers 6 core areas across AI-assisted database development:

| Unit | Topic |
|---|---|
| 1 | Describe AI-assisted development tools for Microsoft SQL platforms |
| 2 | Interpret the security impact of using AI-assisted tools |
| 3 | Enable GitHub Copilot and Fabric Copilot |
| 4 | Configure model and MCP tool options in Copilot chat sessions |
| 5 | Create and configure GitHub Copilot instruction files |
| 6 | Connect to MCP server endpoints (SQL Server and Fabric Lakehouse) |

---

## Unit 1 — Describe AI-Assisted Development Tools

### What Are AI-Assisted Development Tools?

- Use **Large Language Models (LLMs)** to help write, understand, and optimise code.
- Analyse code context, database schema, and natural language prompts to generate relevant suggestions.
- Two primary tools for Microsoft SQL platforms:
  - **GitHub Copilot** — works inside development environments (VS Code, Visual Studio, SSMS).
  - **Fabric Copilot** — integrated into Microsoft Fabric workspaces.

---

### GitHub Copilot

- Works directly in **VS Code**, **Visual Studio**, and **SQL Server Management Studio (SSMS)**.
- Provides code completions, answers questions about the database, and generates T-SQL queries by understanding schema and usage patterns.
- Understands the database context through connected schema information.

### Fabric Copilot

- Integrated into **Microsoft Fabric workspaces**.
- Provides AI assistance for data engineering, data science, and analytics workloads.
- Helps explore data, generate queries, and understand Lakehouse structures.
- Available within the Fabric portal for quick data exploration.

---

### Key Capabilities of Both Tools

| Capability | Description |
|---|---|
| **Code completion** | Suggests complete T-SQL statements, column names, and table references as you type |
| **Natural language to SQL** | Describe requirements in plain English; tool generates corresponding T-SQL |
| **Code explanation** | Highlight existing code and ask for explanation — useful for unfamiliar stored procedures |
| **Query optimisation** | Suggests performance improvements based on best practices and your specific schema |
| **Error assistance** | Diagnoses query failures and suggests fixes |

---

### AI Assistants by Platform

| Platform | AI Tool Available |
|---|---|
| SQL Server | GitHub Copilot (via SSMS and VS Code) |
| Azure SQL Database | GitHub Copilot (via SSMS and VS Code) |
| Azure SQL Managed Instance | GitHub Copilot (via SSMS and VS Code) |
| SQL databases in Microsoft Fabric | **Both** Fabric Copilot (in portal) and GitHub Copilot (external tools) |

> SQL databases in Fabric support **dual access** — use Fabric Copilot for quick portal-based exploration, or GitHub Copilot for intensive IDE-based development work.

---

### Model Context Protocol (MCP) — Role Overview

- An **open standard** that allows AI assistants to connect directly to data sources.
- When MCP is configured, the AI can **query the actual database schema, sample data, and metadata in real time** rather than relying only on visible code context.
- Produces more accurate and contextually relevant suggestions.
- AI understands specific tables, columns, relationships, and data types — not just generic SQL knowledge.

---

### Responsible AI Considerations

- Both GitHub Copilot and Fabric Copilot are designed with **responsible AI principles** and safeguards.
- Prompts and responses are **NOT used to train the underlying AI models**.
- Data is **encrypted in transit and at rest**.

> ⚠️ **Critical:** AI-generated code suggestions must **always be reviewed before execution**, especially for queries that modify data or affect database security. You remain responsible for validating generated code meets requirements and follows organisational standards.

---

## Unit 2 — Interpret Security Impact

### How AI Assistants Process Your Data

**GitHub Copilot transmits:**
- Code currently being written or edited.
- Content from open editor files providing context.
- Database schema information when connected.
- Natural language prompts and questions.

**Fabric Copilot processes:**
- Similar data within the Fabric workspace context.
- Metadata about lakehouses, warehouses, and semantic models.

---

### Organisational Policy Considerations

Before enabling AI-assisted tools, evaluate these areas:

| Area | Key Question |
|---|---|
| **Data classification** | Does the database contain PII, financial records, healthcare data, or regulated information? Some organisations restrict AI tools for sensitive databases. |
| **Compliance requirements** | Do cloud-based AI services align with your compliance obligations (GDPR, HIPAA, etc.)? |
| **Intellectual property** | Are database schemas or code considered trade secrets that shouldn't be processed externally? |
| **Access controls** | Who should have access to AI tools? Align AI tool licensing with existing database access permissions. |

---

### GitHub Copilot Security Features

| Feature | Description |
|---|---|
| **Content filtering** | Filters suggestions to avoid harmful code patterns, SQL injection vulnerabilities, and security anti-patterns |
| **Organisation policies** | Enterprise admins can control which repositories can use Copilot, and whether suggestions can include code matching public repositories |
| **Audit logging** | GitHub Enterprise Cloud organisations can track Copilot usage through audit logs |

---

### Protecting Credentials and Connection Strings

> ⚠️ **Critical Security Rule:** Never paste actual credentials or passwords into AI prompts.

```sql
-- AVOID: Hardcoded credentials in AI-visible files
-- CREATE LOGIN username WITH PASSWORD = 'actual_password';

-- RECOMMENDED: Use parameterised approaches
-- CREATE LOGIN username WITH PASSWORD = $(password);
```

**Best practices:**
- **Never** paste credentials into prompts — do not ask AI to help with connection strings containing actual passwords or keys.
- Store connection information in **environment variables**, not hardcoded in files the AI can see.
- **Review before committing** — check AI-generated code for accidentally included sensitive information before source control commit.

---

### Evaluating AI Suggestions Critically

| Area | What to Check |
|---|---|
| **Review permissions** | When Copilot suggests `GRANT` or `REVOKE` statements, verify permissions align with the least-privilege security model |
| **Validate dynamic SQL** | AI-generated dynamic SQL may be vulnerable to injection if not parameterised — always check for appropriate use of `sp_executesql` with parameters |
| **Check data exposure** | Review `SELECT` statements to ensure they don't inadvertently expose more data than intended, especially in views or stored procedures used by applications |

> **Tip:** Treat AI-generated code as a **first draft requiring human review**. The assistant optimises for functionality and common patterns, but you know your specific security requirements.

---

### MCP Server Security Considerations

| Area | Guidance |
|---|---|
| **Authentication** | Use least-privilege principle — create dedicated service accounts with read-only access when AI only needs schema information |
| **Network security** | For sensitive environments, require private endpoints or VPN connections; consider whether MCP traffic should traverse public networks |
| **Data sampling** | Understand what data might be transmitted when MCP sampling is enabled; ensure it aligns with data handling policies |

---

## Unit 3 — Enable GitHub Copilot and Fabric Copilot

### Prerequisites

**For GitHub Copilot:**
- A GitHub account with Copilot access (Individual, Business, or Enterprise subscription).
- A supported development environment: **Visual Studio Code**, **Visual Studio**, or **SQL Server Management Studio 22+**.
- Active internet connection for AI model communication.

**For Fabric Copilot:**
- A Microsoft Fabric capacity: **F2 or higher**, or Power BI Premium capacity **P1 or higher**.
- Appropriate permissions in your Fabric workspace.
- Copilot features **enabled at the tenant level** by your administrator.

---

### Enable GitHub Copilot in SSMS

SSMS 22+ includes native support for GitHub Copilot.

**Installation steps:**
1. Launch **Visual Studio Installer**.
2. Locate your SSMS installation → **Modify**.
3. On the Workloads tab, select **AI Assistance**.
4. Select **Modify** to install GitHub Copilot components.

**Sign-in steps:**
1. Select the GitHub Copilot badge in SSMS (upper-right corner).
2. Choose **Open Chat Window to Sign In**.
3. Sign in with your GitHub account that has Copilot access.
4. If no subscription, select **Sign up for Copilot Free**.

**SSMS Copilot badge states:** Active | Inactive | Unavailable | Not Installed

**Verification:** Open Copilot Chat window (`Ctrl+\, Ctrl+C`) → Ask "What tables are in this database?" → Verify contextual response.

---

### Enable GitHub Copilot in VS Code

**Installation steps:**
1. Open VS Code → Extensions view (`Ctrl+Shift+X` / `Cmd+Shift+X`).
2. Install **GitHub Copilot** extension.
3. Install **GitHub Copilot Chat** extension.
4. Sign in to GitHub account when prompted.

**For SQL database development also install:**
1. Search for `mssql` in Extensions.
2. Install **SQL Server (mssql)** extension by Microsoft.
3. Connect to database using the Connections view.

**Key feature:** Right-click a connected database → **"Chat with this database"** → starts a context-aware conversation.

**Verification:** Open Copilot Chat panel (`Ctrl+Alt+I`) → Type `-- Get all customers from Seattle` → Observe if Copilot suggests relevant SQL.

---

### Enable Fabric Copilot

**Tenant-level configuration** (requires Fabric Administrator):
1. Navigate to the **Fabric Admin portal**.
2. Go to **Tenant settings**.
3. Under the Copilot and Azure OpenAI Service section, enable **"Users can use Copilot and other features powered by Azure OpenAI"**.
4. Configure which security groups have access.

**Once tenant-enabled, users can access Copilot in:**
- Data Engineering (notebooks, pipelines).
- Data Warehouse (SQL queries, semantic models).
- Data Science (notebooks, experiments).

**To use Copilot in a Fabric SQL database:**
1. Open SQL database in the Fabric portal.
2. Navigate to the query editor.
3. Look for the Copilot button or icon in the toolbar.
4. Start typing natural language queries or use the chat interface.

**Verification:** Open Fabric SQL database → Ask "Describe the tables in this database" → Verify response reflects actual schema.

---

### Enterprise Setup Best Practices

- **Standardise subscriptions** — ensure all team members have the same Copilot access level.
- **Document approved uses** — create guidelines for when/how to use AI assistance, especially for production database changes.
- **Set up shared instruction files** — custom instruction files guide Copilot behaviour for team coding standards (see Unit 5).

---

## Unit 4 — Configure Model and MCP Tool Options

### Understanding Model Selection

GitHub Copilot supports multiple AI models with different capabilities:

| Model | Strengths |
|---|---|
| **GPT models** | Advanced reasoning, excellent for complex T-SQL generation and database design |
| **Claude models** | Strong at explaining code and providing detailed documentation |
| **Gemini models** | Available in some configurations with different strengths |

**To change models in SSMS:**
1. Open the Copilot Chat window.
2. Use the model selector dropdown at the top of the chat.

**To change models in VS Code:**
1. Open Copilot Chat panel (`Ctrl+Alt+I`).
2. Use the model picker in the chat interface.

> **Tip:** Use advanced models for complex query optimisation or database design. Use faster models for quick code completions (lower latency).

---

### What Is Model Context Protocol (MCP)?

- An **open standard** enabling AI assistants to connect directly to external data sources and tools.
- Instead of relying only on visible code context, MCP enables the AI to **query your actual database schema, sample data, and metadata in real time**.
- Produces more accurate, reliable, and contextually aware suggestions.

**Without MCP:** AI works only with what's visible in the editor — generic SQL knowledge plus open files.
**With MCP:** AI can query your database directly — knows your specific tables, columns, relationships, and data types.

#### MCP Architecture

```
MCP Host (GitHub Copilot / Claude / etc.)
    ↓
MCP Client (protocol client in your tool)
    ↓
MCP Server (service exposing data sources/tools)
    ↓
SQL Database / Fabric Lakehouse
```

| Component | Role |
|---|---|
| **MCP Host** | Your AI environment (GitHub Copilot, Claude, etc.) |
| **MCP Client** | The protocol client that connects to MCP servers |
| **MCP Server** | A service that exposes specific data sources or tools |

---

### Configure MCP Tools in GitHub Copilot (VS Code)

#### Step 1 — Enable Agent Mode
- Agent mode allows Copilot to use external tools including MCP servers.
- Open Copilot Chat panel → Look for mode selector → Switch to **Agent** mode.

#### Step 2 — Add an MCP Server
1. Open Command Palette (`Ctrl+Shift+P`).
2. Type **MCP: Add Server** and select it.
3. Choose server type: **HTTP** or **Stdio**.
4. Enter the server URL or configuration.

#### Step 3 — Manage MCP Tools
1. In Agent mode, select the **tools icon** in the chat panel.
2. View available MCP servers and their tools.
3. Enable or disable specific tools as needed.

---

### MCP Server Options for SQL Databases

| MCP Server | Connects To | Notes |
|---|---|---|
| **SQL MCP Server** | SQL Server and Azure SQL | Microsoft open-source solution built on Data API builder; secure, managed schema exposure |
| **Microsoft Fabric MCP Server** | Fabric lakehouses, warehouses, SQL databases | Connects via Fabric data agents as MCP servers |
| **Azure MCP Server** | Broader Azure resources including SQL databases | Provides broader Azure resource integration |

**Configuring SQL MCP server in VS Code (`mcp.json`):**
```json
{
  "servers": {
    "sql-server": {
      "type": "http",
      "url": "https://your-mcp-endpoint/mcp"
    }
  }
}
```
- Create `.vscode/mcp.json` in your workspace.
- VS Code detects the configuration and prompts to start the server.
- Authenticate when prompted.

---

### Configure MCP in Fabric Copilot

To publish a Fabric data agent as an MCP server:
1. Create and configure a **data agent** in your Fabric workspace.
2. Publish the data agent.
3. Go to agent's **Settings** → Open the **Model Context Protocol** tab.
4. Copy the **MCP server URL**.
5. Use this URL in VS Code or other tools.

> This allows you to use the **same data agent** from both the Fabric portal and external development environments.

---

### MCP Configuration Best Practices

| Practice | Detail |
|---|---|
| **Start with defaults** | Begin with default settings, then adjust based on experience |
| **Match models to tasks** | Advanced models for complex design; simpler models for routine completion |
| **Limit MCP scope** | Least-privilege access — if AI only needs schema information, don't grant data read permissions |
| **Test configurations** | After changing settings, verify the assistant still provides accurate suggestions |

> ⚠️ MCP servers establish **direct connections to your databases**. Ensure network security policies allow connections and authentication follows security best practices.

---

## Unit 5 — Create and Configure GitHub Copilot Instruction Files

### What Are Custom Instruction Files?

- **Markdown documents** containing guidance for GitHub Copilot.
- Placed in specific locations so Copilot includes their contents as context when generating suggestions.
- Influence how Copilot responds **without modifying each individual prompt**.

---

### Two Types of Instruction Files

| Type | File Name | Location | Scope |
|---|---|---|---|
| **Repository instructions** | `copilot-instructions.md` | `.github/` folder at repository root | Everyone working in the repository |
| **Prompt files** | `*.prompt.md` | `.github/prompts/` folder | Reusable prompts for specific tasks; referenced in chat conversations |

---

### Repository Instruction File (`copilot-instructions.md`)

**How to create:**
1. Navigate to your repository root folder.
2. Create a `.github` folder if it doesn't exist.
3. Create a file named `copilot-instructions.md`.
4. Add instructions in markdown format.

**Example instruction file for a database project:**

```markdown
# Project Guidelines for Copilot

## Database Development Standards

This project uses SQL Server 2022 with the following conventions:

### Naming Conventions
- Tables: PascalCase, singular (Customer, OrderDetail)
- Columns: PascalCase (FirstName, OrderDate)
- Stored procedures: usp_ActionEntity (usp_GetCustomerOrders)
- Views: vw_EntityName (vw_ActiveCustomers)
- Indexes: IX_TableName_ColumnName

### T-SQL Style
- Use explicit column lists in SELECT statements (avoid SELECT *)
- Always include schema prefix (dbo.TableName)
- Use ANSI JOIN syntax, not comma-separated tables
- Include error handling in all stored procedures
- Use TRY...CATCH blocks for data modification operations

### Security Requirements
- Never generate GRANT statements to public
- Use parameterised queries, never concatenate user input
- Avoid dynamic SQL when possible

### Performance Guidelines
- Suggest appropriate indexes when creating tables
- Prefer SET NOCOUNT ON in stored procedures
- Use EXISTS instead of COUNT for existence checks
```

---

### Configuring Instruction File Settings

**In VS Code:**
1. Open Settings (`Ctrl+,`).
2. Search for "GitHub Copilot: Instructions".
3. Configure which instruction files to include automatically.

**In Visual Studio:**
1. Go to **Tools** > **Options**.
2. Navigate to **GitHub** > **Copilot**.
3. Configure custom instruction preferences.

You can also reference instruction files explicitly in chat: Command Palette (`Ctrl+Shift+P`) → **Chat: Configure Instructions**.

---

### Reusable Prompt Files (`.prompt.md`)

- Define templates for **common tasks**.
- Stored in `.github/prompts/` folder.
- Referenced in chat using the `#` symbol or prompt picker.

**Use cases for database development:**
- Generating stored procedure templates.
- Creating data migration scripts.
- Reviewing query performance.
- Documenting existing code.

**Example prompt file (`create-procedure.prompt.md`):**

```markdown
# Create Stored Procedure

Generate a stored procedure with the following specifications:

- Procedure name: {{procedureName}}
- Purpose: {{description}}
- Parameters: {{parameters}}

Follow these requirements:
- Include TRY...CATCH error handling
- Add SET NOCOUNT ON at the beginning
- Include appropriate comments
- Follow our naming conventions (usp_ prefix)
- Use explicit transactions for data modifications
```

---

### Team-Wide Instruction Strategies

| Strategy | Description |
|---|---|
| **Centralised standards** | Maintain a master instruction file in a shared repository; copy or link to project repositories |
| **Layered instructions** | Organisation-level instructions for general standards + repository-specific additions |
| **Version control** | Track instruction file changes in git — review how AI guidance evolves over time |

---

### Best Practices for Instruction Files

| Practice | Detail |
|---|---|
| **Be specific** | Vague instructions → vague results. Specify exact naming patterns, not just "use good naming" |
| **Provide examples** | Include code snippets demonstrating preferred patterns |
| **Prioritise** | Put the most important guidelines first |
| **Test iteratively** | After creating files, test various prompts to see if Copilot follows guidance; adjust wording if needed |
| **Keep current** | Update instructions when standards change — outdated guidance leads to inconsistent suggestions |

---

## Unit 6 — Connect to MCP Server Endpoints

### Connect to SQL Server MCP Endpoints

**Using SQL MCP Server (recommended for SQL Server / Azure SQL):**
- Microsoft's open-source solution built on **Data API builder**.
- Enables AI agents to interact with SQL databases.

**Setup steps:**
1. Install and configure SQL MCP Server for your database.
2. Enable the MCP endpoint in the SQL MCP Server configuration.
3. Note the MCP endpoint URL (typically ending in `/mcp`).
4. Add the server to VS Code using the **MCP: Add Server** command.

**Direct SQL Server connection (simpler alternative):**
- Use the MSSQL extension's built-in integration in VS Code.
- Right-click connected database → **"Chat with this database"** → Schema-aware conversations without a separate MCP server.

---

### Connect to Azure SQL MCP Endpoints

**Using Azure MCP Server:**
1. Install the Azure MCP Server extension in VS Code.
2. Authenticate with your Azure account.
3. Configure access to Azure SQL resources.
4. Use Agent mode to query databases.

**Network configuration required for Azure SQL:**
- Ensure client IP is allowed through the **Azure SQL firewall**.
- For private endpoints, configure network to allow MCP traffic.
- Consider using **service principals** for automated or shared MCP access.

**Example `mcp.json` with authentication input prompt:**

```json
{
  "servers": {
    "azure-sql-mcp": {
      "type": "http",
      "url": "https://your-api-endpoint.azurewebsites.net/mcp",
      "headers": {
        "Authorization": "Bearer ${input:azure_token}"
      }
    }
  },
  "inputs": [
    {
      "id": "azure_token",
      "type": "promptString",
      "description": "Azure access token",
      "password": true
    }
  ]
}
```

---

### Connect to Fabric Lakehouse MCP Endpoints

**Publishing a Fabric data agent as MCP server:**
1. Create a data agent in your Fabric workspace.
2. Configure the data agent with access to your lakehouse or warehouse.
3. Publish the data agent.
4. Navigate to agent's **Settings** → **Model Context Protocol** tab.
5. Copy the **MCP server URL** and tool description.

**Configuring VS Code to connect (`.vscode/mcp.json`):**

```json
{
  "servers": {
    "fabric-lakehouse": {
      "type": "http",
      "url": "https://your-fabric-mcp-endpoint-url"
    }
  }
}
```

**When VS Code detects the configuration:**
1. Select **Add Server** when prompted.
2. Authenticate with your Microsoft account.
3. Fabric MCP server appears in available tools.

**Alternative:** Install the **Microsoft Fabric MCP Server extension** from VS Code marketplace for a streamlined experience.

---

### Authentication Methods for MCP Endpoints

| Method | How It Works | Best For |
|---|---|---|
| **Interactive authentication** | Sign in through a browser prompt when connecting | Development environments |
| **Service principal** | Configure a Microsoft Entra ID application with appropriate permissions | Automated scenarios and shared environments |
| **API keys** | Some MCP servers support API key authentication | Various — store keys securely using environment variables or VS Code's input prompts |

> ⚠️ **Never store credentials directly in configuration files that might be committed to source control.** Use environment variables, input prompts, or secure credential stores.

---

### Testing MCP Connections

**In VS Code Agent mode:**
1. Switch to Agent mode in the Copilot Chat panel.
2. Click the **tools icon** to see available MCP servers.
3. Verify configured servers appear with a **green status**.
4. Ask "What tables are available in the database?"
5. Confirm response reflects actual database schema.

---

### Troubleshooting MCP Connection Issues

| Issue | Likely Cause | Solution |
|---|---|---|
| Server not listed | Configuration file syntax error | Validate JSON in `mcp.json` |
| Authentication failed | Expired credentials | Re-authenticate or refresh tokens |
| Connection timeout | Network/firewall blocking | Check firewall rules and network access |
| Empty schema results | Insufficient permissions | Verify database permissions for authenticated user |

---

### Best Practices for MCP Endpoint Management

| Practice | Detail |
|---|---|
| **Use least-privilege access** | Create dedicated accounts for MCP connections with only required permissions (typically read-only schema access) |
| **Separate environments** | Configure different MCP endpoints for dev, test, and production — avoid connecting AI assistants to production data during routine development |
| **Monitor connections** | Track MCP server usage through logging and monitoring |
| **Keep configurations consistent** | For team environments, share MCP configurations through version control so everyone connects to the same endpoints |

> **Tip:** Start with a **development or test database** when learning MCP configuration. Once comfortable, extend to additional environments.

---

## Quick Reference Summary

| Topic | Key Points |
|---|---|
| GitHub Copilot tools | Works in SSMS 22+, VS Code, Visual Studio |
| Fabric Copilot tools | Available in Fabric portal; requires F2+ or P1+ capacity |
| MCP role | Enables AI to query real-time schema/metadata — more accurate suggestions |
| Data sent to AI | Code context, open files, schema info, natural language prompts |
| Data NOT used for | Training AI models — prompts and responses excluded from training |
| Credentials rule | NEVER paste credentials into AI prompts; use environment variables |
| AI code rule | Always review AI-generated code before executing |
| SSMS Copilot prerequisite | GitHub account + Copilot subscription + SSMS 22+ + AI Assistance workload |
| Fabric Copilot prerequisite | F2+ or P1+ capacity + tenant-level admin enablement |
| MCP architecture | Host → Client → Server → Database |
| Agent mode | Required in VS Code to use MCP tools with Copilot |
| `mcp.json` location | `.vscode/mcp.json` in workspace folder |
| Repository instruction file | `.github/copilot-instructions.md` |
| Prompt files | `.github/prompts/*.prompt.md` |
| MCP auth best practice | Least-privilege; dedicated read-only accounts; never commit credentials |
| SQL MCP Server | Built on Data API builder — Microsoft open-source solution |
| Fabric MCP Server | Published via data agent Settings → Model Context Protocol tab |
| MCP troubleshooting | Validate JSON syntax → Check auth → Check firewall → Verify permissions |

---

## Key Facts to Remember for the Exam

1. **Two primary AI tools:** GitHub Copilot (SSMS, VS Code, Visual Studio) and Fabric Copilot (Fabric portal). Fabric SQL databases support **both**.
2. Prompts and responses are **NOT used to train** the underlying AI models. Data is **encrypted in transit and at rest**.
3. AI-generated code must **always be reviewed before execution** — you remain responsible.
4. **Never paste actual credentials or passwords** into AI prompts — use environment variables instead.
5. GitHub Copilot **content filtering** prevents generation of SQL injection vulnerabilities and harmful security patterns.
6. **Audit logging** for Copilot usage is available with **GitHub Enterprise Cloud**.
7. GitHub Copilot in SSMS requires: GitHub account with Copilot access + **SSMS 22+** + **AI Assistance workload** installed via Visual Studio Installer.
8. Fabric Copilot requires: **F2 or higher** Fabric capacity (or P1+ Power BI Premium) + tenant-level administrator enablement.
9. MCP (Model Context Protocol) is an **open standard** — not proprietary — enabling real-time schema/data access.
10. MCP architecture: **Host → Client → Server → Database**. Host = AI environment, Server = service exposing data.
11. **Agent mode** must be enabled in VS Code Copilot Chat to use MCP tools.
12. `mcp.json` is stored at **`.vscode/mcp.json`** in the workspace folder.
13. SQL MCP Server is built on **Data API builder** (Microsoft open-source).
14. Fabric MCP Server URL is found in data agent **Settings → Model Context Protocol tab**.
15. **Repository instruction file:** `.github/copilot-instructions.md` — applies to everyone in the repository.
16. **Prompt files:** `.github/prompts/*.prompt.md` — reusable templates for specific tasks.
17. Instruction files use **markdown format**; be specific and provide code examples for best results.
18. MCP connections should use **least-privilege** access — typically read-only for schema information.
19. Always use **separate MCP endpoints** for dev, test, and production — avoid connecting AI to production during routine development.
20. Dynamic SQL generated by AI should always be checked for **injection vulnerabilities** — verify `sp_executesql` with parameters is used.
21. When Copilot suggests `GRANT`/`REVOKE` statements, verify they align with the **least-privilege security model**.
22. The shortcut to open Copilot Chat in SSMS is **`Ctrl+\, Ctrl+C`**. In VS Code it is **`Ctrl+Alt+I`**.

---

*End of DP-800 Module 4 Notes — Implement SQL Solutions by Using AI-Assisted Tools*