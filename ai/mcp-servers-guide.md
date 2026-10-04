# Understanding Model Context Protocol (MCP) Servers and Spring Boot Integration

The **Model Context Protocol (MCP)** is an open standard introduced by Anthropic that creates a universal way for AI models (hosted in environments like Claude Desktop, IDEs, or custom agents) to securely connect to external data sources and tools. 

If the LLM acts as the **"brain,"** the MCP server acts as the **"hands."** Instead of hardcoding custom tool integrations for every LLM provider, developers build a single MCP server. The protocol standardizes how models discover available functions, pass arguments, and receive structured outputs.

---

## How MCP Works

MCP uses a modular client-server architecture:

* **MCP Host:** The AI-powered application or development environment the user interacts with.
* **MCP Client:** The internal component maintaining a 1:1 connection with an MCP server.
* **MCP Server:** A lightweight service (such as a Spring Boot application) exposing capabilities as **Tools** (executable functions), **Resources** (static or dynamic data), or **Prompts**.
* **Transport Protocol:** Communication relies on **JSON-RPC** messages transmitted either locally via **STDIO** (standard input/output) or remotely via **Server-Sent Events (SSE)**.

---

## Example in Spring Boot (with Spring AI)

Spring AI provides native support for building MCP servers. By using simple annotations, you can turn any standard Spring service into an AI-discoverable tool.

### 1. Define the Service with `@Tool`
Annotate Java methods so the framework knows they are accessible to AI models:

```java
@Service
public class TaskService {
    private final List<String> tasks = new ArrayList<>(List.of("Review pull request", "Update documentation"));

    @Tool(name = "get_tasks", description = "Get the current list of tasks")
    public List<String> getTasks() {
        return tasks;
    }

    @Tool(name = "add_task", description = "Add a new task to the list")
    public String addTask(@ToolParam(description = "The task description") String task) {
        tasks.add(task);
        return "Successfully added task: " + task;
    }
}
```

### 2. Register the Tool Callbacks
Expose the annotated service components as MCP tool callbacks in your main application configuration:

```java
@SpringBootApplication
public class McpServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(McpServerApplication.class, args);
    }

    @Bean
    public List<ToolCallback> taskTools(TaskService taskService) {
        return List.of(ToolCallbacks.from(taskService));
    }
}
```

### 3. Configure Properties for STDIO Transport
For local desktop clients (like Claude Desktop), configure the application to run synchronously over standard input/output without starting an embedded web server:

```properties
spring.main.web-application-type=none
spring.ai.mcp.server.type=SYNC
logging.pattern.console=
```

Once packaged as a JAR, you can register this Spring Boot app in your client configuration file. The AI model will dynamically inspect the `get_tasks` and `add_task` definitions and call them autonomously whenever you prompt it in natural language.