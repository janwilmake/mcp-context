# Complete Guide: Making Your MCP Server Tasks-Compatible

<!-- https://contextarea.com/rules-httpsuithu-hp86fniyc1dsev -->

## Overview

Tasks enable clients to execute long-running tool calls asynchronously with polling-based result retrieval. This guide covers implementing full task support in your MCP server.

## 1. Capability Declaration

### Declare Task Support in `initialize`

```typescript
// Server initialization response
{
  "capabilities": {
    "tasks": {
      "list": {},      // Support tasks/list
      "cancel": {},    // Support tasks/cancel
      "requests": {
        "tools": {
          "call": {}   // Support task-augmented tools/call
        }
      }
    },
    "tools": {
      "listChanged": true
    }
  }
}
```

## 2. Tool-Level Task Configuration

### Configure Each Tool's Task Support

```typescript
// In tools/list response
{
  "tools": [
    {
      "name": "long_running_analysis",
      "title": "Long Running Analysis",
      "description": "Analyzes large datasets",
      "inputSchema": {
        "type": "object",
        "properties": {
          "dataset_id": { "type": "string" }
        }
      },
      "execution": {
        "taskSupport": "optional"  // "required" | "optional" | "forbidden"
      }
    }
  ]
}
```

**Task Support Options:**

- `"forbidden"` (default): Tool doesn't support tasks
- `"optional"`: Client may use task augmentation
- `"required"`: Client MUST use task augmentation

## 3. Task State Management

### Core Task Data Structure

```typescript
interface TaskState {
  taskId: string;
  status: "working" | "input_required" | "completed" | "failed" | "cancelled";
  statusMessage?: string;
  createdAt: string; // ISO 8601
  lastUpdatedAt: string; // ISO 8601
  ttl: number | null; // milliseconds or null for unlimited
  pollInterval?: number; // suggested ms between polls

  // Internal state
  toolName: string;
  arguments: Record<string, unknown>;
  result?: CallToolResult;
  error?: Error;
}

class TaskManager {
  private tasks = new Map<string, TaskState>();

  createTask(
    toolName: string,
    args: Record<string, unknown>,
    ttl?: number,
  ): TaskState {
    const taskId = crypto.randomUUID();
    const now = new Date().toISOString();

    const task: TaskState = {
      taskId,
      status: "working",
      createdAt: now,
      lastUpdatedAt: now,
      ttl: ttl ?? 3600000, // Default 1 hour
      pollInterval: 5000, // Suggest 5s polling
      toolName,
      arguments: args,
    };

    this.tasks.set(taskId, task);
    this.scheduleCleanup(task);

    return task;
  }

  private scheduleCleanup(task: TaskState) {
    if (task.ttl) {
      setTimeout(() => {
        this.tasks.delete(task.taskId);
      }, task.ttl);
    }
  }
}
```

## 4. Handle Task-Augmented `tools/call`

### Detect and Process Task Requests

```typescript
async function handleToolCall(
  params: CallToolRequestParams,
): Promise<JSONRPCResponse> {
  const tool = findTool(params.name);

  // Check task support compatibility
  if (params.task) {
    // Client wants task execution
    if (tool.execution?.taskSupport === "forbidden") {
      return {
        jsonrpc: "2.0",
        id: requestId,
        error: {
          code: -32601,
          message: `Tool '${params.name}' does not support task execution`,
        },
      };
    }

    // Create task and return immediately
    const task = taskManager.createTask(
      params.name,
      params.arguments ?? {},
      params.task.ttl,
    );

    // Start async execution
    executeToolAsync(task);

    return {
      jsonrpc: "2.0",
      id: requestId,
      result: {
        task: {
          taskId: task.taskId,
          status: task.status,
          statusMessage: "Tool execution started",
          createdAt: task.createdAt,
          lastUpdatedAt: task.lastUpdatedAt,
          ttl: task.ttl,
          pollInterval: task.pollInterval,
        },
        _meta: {
          "io.modelcontextprotocol/model-immediate-response": `Task ${task.taskId} created. The analysis is running in the background.`,
        },
      },
    };
  } else {
    // Non-task execution
    if (tool.execution?.taskSupport === "required") {
      return {
        jsonrpc: "2.0",
        id: requestId,
        error: {
          code: -32601,
          message: `Tool '${params.name}' requires task execution`,
        },
      };
    }

    // Execute synchronously
    const result = await executeTool(params.name, params.arguments);
    return { jsonrpc: "2.0", id: requestId, result };
  }
}
```

## 5. Async Tool Execution

### Execute Tool and Update Task State

```typescript
async function executeToolAsync(task: TaskState) {
  try {
    // Update to working
    updateTaskStatus(task, "working", "Processing request...");

    // Execute the actual tool logic
    const result = await executeTool(task.toolName, task.arguments);

    // Store result and mark complete
    task.result = result;
    updateTaskStatus(
      task,
      "completed",
      "Tool execution completed successfully",
    );

    // Optionally send status notification
    sendTaskStatusNotification(task);
  } catch (error) {
    task.error = {
      code: -32603,
      message: error.message,
    };
    updateTaskStatus(task, "failed", `Tool execution failed: ${error.message}`);
    sendTaskStatusNotification(task);
  }
}

function updateTaskStatus(
  task: TaskState,
  status: TaskStatus,
  statusMessage?: string,
) {
  task.status = status;
  task.statusMessage = statusMessage;
  task.lastUpdatedAt = new Date().toISOString();
}
```

## 6. Implement `tasks/get`

### Poll Task Status

```typescript
function handleTasksGet(params: { taskId: string }): GetTaskResult {
  const task = taskManager.tasks.get(params.taskId);

  if (!task) {
    return {
      jsonrpc: "2.0",
      id: requestId,
      error: {
        code: -32602,
        message: "Task not found or expired",
      },
    };
  }

  return {
    jsonrpc: "2.0",
    id: requestId,
    result: {
      taskId: task.taskId,
      status: task.status,
      statusMessage: task.statusMessage,
      createdAt: task.createdAt,
      lastUpdatedAt: task.lastUpdatedAt,
      ttl: task.ttl,
      pollInterval: task.pollInterval,
    },
  };
}
```

## 7. Implement `tasks/result`

### Block Until Complete, Return Tool Result

```typescript
async function handleTasksResult(params: {
  taskId: string;
}): Promise<JSONRPCResponse> {
  const task = taskManager.tasks.get(params.taskId);

  if (!task) {
    return {
      jsonrpc: "2.0",
      id: requestId,
      error: {
        code: -32602,
        message: "Task not found or expired",
      },
    };
  }

  // Block until terminal status
  await waitForTerminalStatus(task);

  // Return the actual tool result
  if (task.status === "completed") {
    return {
      jsonrpc: "2.0",
      id: requestId,
      result: {
        ...task.result,
        _meta: {
          "io.modelcontextprotocol/related-task": {
            taskId: task.taskId,
          },
        },
      },
    };
  } else if (task.status === "failed") {
    return {
      jsonrpc: "2.0",
      id: requestId,
      error: task.error,
    };
  } else if (task.status === "cancelled") {
    return {
      jsonrpc: "2.0",
      id: requestId,
      error: {
        code: -32603,
        message: "Task was cancelled",
      },
    };
  }
}

async function waitForTerminalStatus(task: TaskState): Promise<void> {
  const terminalStatuses = ["completed", "failed", "cancelled"];

  while (!terminalStatuses.includes(task.status)) {
    await new Promise((resolve) => setTimeout(resolve, 100));
  }
}
```

## 8. Implement `tasks/list`

### Return All Tasks with Pagination

```typescript
function handleTasksList(params: { cursor?: string }): ListTasksResult {
  const tasks = Array.from(taskManager.tasks.values());
  const pageSize = 50;
  const startIndex = params.cursor ? parseInt(params.cursor) : 0;
  const endIndex = startIndex + pageSize;

  const page = tasks.slice(startIndex, endIndex);
  const hasMore = endIndex < tasks.length;

  return {
    jsonrpc: "2.0",
    id: requestId,
    result: {
      tasks: page.map((task) => ({
        taskId: task.taskId,
        status: task.status,
        statusMessage: task.statusMessage,
        createdAt: task.createdAt,
        lastUpdatedAt: task.lastUpdatedAt,
        ttl: task.ttl,
        pollInterval: task.pollInterval,
      })),
      nextCursor: hasMore ? endIndex.toString() : undefined,
    },
  };
}
```

## 9. Implement `tasks/cancel`

### Cancel Active Tasks

```typescript
function handleTasksCancel(params: { taskId: string }): CancelTaskResult {
  const task = taskManager.tasks.get(params.taskId);

  if (!task) {
    return {
      jsonrpc: "2.0",
      id: requestId,
      error: {
        code: -32602,
        message: "Task not found",
      },
    };
  }

  // Reject if already terminal
  const terminalStatuses = ["completed", "failed", "cancelled"];
  if (terminalStatuses.includes(task.status)) {
    return {
      jsonrpc: "2.0",
      id: requestId,
      error: {
        code: -32602,
        message: `Cannot cancel task: already in terminal status '${task.status}'`,
      },
    };
  }

  // Cancel the task
  cancelTaskExecution(task);
  updateTaskStatus(task, "cancelled", "Task cancelled by client request");
  sendTaskStatusNotification(task);

  return {
    jsonrpc: "2.0",
    id: requestId,
    result: {
      taskId: task.taskId,
      status: task.status,
      statusMessage: task.statusMessage,
      createdAt: task.createdAt,
      lastUpdatedAt: task.lastUpdatedAt,
      ttl: task.ttl,
      pollInterval: task.pollInterval,
    },
  };
}
```

## 10. Optional: Status Notifications

### Proactively Notify Status Changes

```typescript
function sendTaskStatusNotification(task: TaskState) {
  const notification = {
    jsonrpc: "2.0",
    method: "notifications/tasks/status",
    params: {
      taskId: task.taskId,
      status: task.status,
      statusMessage: task.statusMessage,
      createdAt: task.createdAt,
      lastUpdatedAt: task.lastUpdatedAt,
      ttl: task.ttl,
      pollInterval: task.pollInterval,
    },
  };

  sendNotification(notification);
}
```

## 11. Handle `input_required` Status

### Request Input During Execution

```typescript
async function executeToolWithInput(task: TaskState) {
  try {
    // Start processing
    updateTaskStatus(task, "working");

    // Discover need for input
    const needsApproval = await checkIfNeedsApproval(task);

    if (needsApproval) {
      // Move to input_required
      updateTaskStatus(task, "input_required", "Waiting for user approval");
      sendTaskStatusNotification(task);

      // Send elicitation request with related task metadata
      const elicitResponse = await sendElicitationRequest({
        mode: "form",
        message: "Please approve this operation",
        requestedSchema: {
          type: "object",
          properties: {
            approve: { type: "boolean" },
          },
        },
        _meta: {
          "io.modelcontextprotocol/related-task": {
            taskId: task.taskId,
          },
        },
      });

      if (
        elicitResponse.action === "accept" &&
        elicitResponse.content?.approve
      ) {
        // Resume processing
        updateTaskStatus(task, "working", "Resuming after approval");
        await continueExecution(task);
      } else {
        updateTaskStatus(task, "cancelled", "User declined approval");
      }
    }
  } catch (error) {
    updateTaskStatus(task, "failed", error.message);
  }
}
```

## 12. Security Best Practices

### Access Control and Isolation

```typescript
class SecureTaskManager {
  private tasks = new Map<string, TaskState>();

  // Bind tasks to authorization context
  createTask(
    authContext: string,
    toolName: string,
    args: Record<string, unknown>,
  ): TaskState {
    const task = {
      ...createBaseTask(toolName, args),
      authContext, // Store auth context
    };
    this.tasks.set(task.taskId, task);
    return task;
  }

  // Verify access before operations
  getTask(taskId: string, authContext: string): TaskState | null {
    const task = this.tasks.get(taskId);
    if (!task || task.authContext !== authContext) {
      return null; // Access denied
    }
    return task;
  }

  // Generate cryptographically secure IDs
  generateTaskId(): string {
    return crypto.randomUUID(); // 122 bits of entropy
  }

  // Enforce resource limits
  createTaskWithLimits(authContext: string, ...params): TaskState {
    const userTasks = this.countUserTasks(authContext);
    if (userTasks >= MAX_CONCURRENT_TASKS) {
      throw new Error("Maximum concurrent tasks exceeded");
    }

    const requestedTtl = params.ttl;
    const actualTtl = Math.min(requestedTtl, MAX_TTL_MS);

    return this.createTask(authContext, ...params, actualTtl);
  }
}
```

## 13. Complete Implementation Example

```typescript
class TaskEnabledMCPServer {
  private taskManager = new TaskManager();

  async handleRequest(request: JSONRPCRequest): Promise<JSONRPCResponse> {
    switch (request.method) {
      case "initialize":
        return this.handleInitialize();

      case "tools/list":
        return this.handleToolsList();

      case "tools/call":
        return this.handleToolCall(request.params);

      case "tasks/get":
        return this.handleTasksGet(request.params);

      case "tasks/result":
        return await this.handleTasksResult(request.params);

      case "tasks/list":
        return this.handleTasksList(request.params);

      case "tasks/cancel":
        return this.handleTasksCancel(request.params);
    }
  }

  private handleInitialize() {
    return {
      protocolVersion: "2025-11-25",
      capabilities: {
        tasks: {
          list: {},
          cancel: {},
          requests: {
            tools: { call: {} },
          },
        },
        tools: { listChanged: true },
      },
      serverInfo: {
        name: "task-enabled-server",
        version: "1.0.0",
      },
    };
  }

  private handleToolsList() {
    return {
      tools: [
        {
          name: "analyze_data",
          title: "Analyze Data",
          description: "Long-running data analysis",
          inputSchema: {
            type: "object",
            properties: {
              dataset: { type: "string" },
            },
          },
          execution: {
            taskSupport: "optional",
          },
        },
      ],
    };
  }
}
```

## Key Requirements Checklist

✅ **Capability Negotiation:**

- Declare `tasks` capability in `initialize`
- Set tool-level `execution.taskSupport`

✅ **Task Creation:**

- Return `CreateTaskResult` immediately
- Generate unique, secure task IDs
- Respect/override client TTL requests

✅ **Status Lifecycle:**

- Start in `working` status
- Follow valid state transitions
- Update `lastUpdatedAt` on changes

✅ **Task Operations:**

- `tasks/get`: Return current status
- `tasks/result`: Block until terminal, return tool result
- `tasks/list`: Support pagination
- `tasks/cancel`: Validate state, prevent terminal cancellation

✅ **Metadata:**

- Include `io.modelcontextprotocol/related-task` in results
- Provide `io.modelcontextprotocol/model-immediate-response` for streaming responses

✅ **Security:**

- Bind tasks to auth context
- Validate access on all operations
- Use cryptographically secure IDs
- Enforce resource limits

This guide covers all task features. Implement these patterns for a fully tasks-compatible MCP server!
