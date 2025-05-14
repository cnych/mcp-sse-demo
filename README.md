[![MseeP.ai Security Assessment Badge](https://mseep.net/pr/cnych-mcp-sse-demo-badge.png)](https://mseep.ai/app/cnych-mcp-sse-demo)

# 开发 SSE 类型的 MCP 服务

[MCP](https://www.claudemcp.com/) 支持两种通信传输方法：`STDIO`（标准输入/输出）或 `SSE`（服务器推送事件），两者都使用 `JSON-RPC 2.0` 进行消息格式化。`STDIO` 用于本地集成，而 `SSE` 用于基于网络的通信。

比如我们想直接在命令行中使用 MCP 服务，那么我们可以使用 `STDIO` 传输方法，如果我们要在 Web 页面中使用 MCP 服务，那么我们可以使用 `SSE` 传输方法。

接下来我们将为大家开发一个基于 MCP 的智能商城服务助手，使用 SSE 类型的 MCP 服务，具备以下核心功能：

- 实时访问产品信息和库存水平，支持定制订单。
- 根据客户偏好和可用库存推荐产品。
- 使用 MCP 工具服务器与微服务进行实时交互。
- 在回答产品询问时检查实时库存水平。
- 使用产品 ID 和数量促进产品购买。
- 实时更新库存水平。
- 通过自然语言查询提供订单交易的临时分析。

![](https://picdn.youdianzhishi.com/images/1743578609885.png)

> 这里我们使用 Anthropic Claude 3.5 Sonnet 模型作为 MCP 服务的 AI 助手，当然也可以选择其他支持工具调用的模型。

首先需要一个产品微服务，用于暴露一个产品列表的 API 接口。然后再提供一个订单微服务，用于暴露一个订单创建、库存信息等 API 接口。

接下来的核心就是核心的 MCP SSE 服务器，用于向 LLM 暴露产品微服务和订单微服务数据，作为使用 SSE 协议的工具。

最后就是使用 MCP 客户端，通过 SSE 协议连接到 MCP SSE 服务器，并使用 LLM 进行交互。

## 微服务

接下来我们开始开发产品微服务和订单微服务，并暴露 API 接口。

首先定义产品、库存和订单的类型。

```typescript
// types/index.ts
export interface Product {
  id: number;
  name: string;
  price: number;
  description: string;
}

export interface Inventory {
  productId: number;
  quantity: number;
  product?: Product;
}

export interface Order {
  id: number;
  customerName: string;
  items: Array<{ productId: number; quantity: number }>;
  totalAmount: number;
  orderDate: string;
}
```

然后我们可以用 Express 来暴露产品微服务和订单微服务，并提供 API 接口。由于是模拟数据，所以我们这里用更简单的内存数据来模拟，直接把数据通过下面的这些函数暴露出去。（生产环境中，还是需要使用微服务加数据库的方式来实现）

```typescript
// services/product-service.ts
import { Product, Inventory, Order } from "../types/index.js";

// 模拟数据存储
let products: Product[] = [
  {
    id: 1,
    name: "智能手表Galaxy",
    price: 1299,
    description: "健康监测，运动追踪，支持多种应用",
  },
  {
    id: 2,
    name: "无线蓝牙耳机Pro",
    price: 899,
    description: "主动降噪，30小时续航，IPX7防水",
  },
  {
    id: 3,
    name: "便携式移动电源",
    price: 299,
    description: "20000mAh大容量，支持快充，轻薄设计",
  },
  {
    id: 4,
    name: "华为MateBook X Pro",
    price: 1599,
    description: "14.2英寸全面屏，3:2比例，100% sRGB色域",
  },
];

// 模拟库存数据
let inventory: Inventory[] = [
  { productId: 1, quantity: 100 },
  { productId: 2, quantity: 50 },
  { productId: 3, quantity: 200 },
  { productId: 4, quantity: 150 },
];

let orders: Order[] = [];

export async function getProducts(): Promise<Product[]> {
  return products;
}

export async function getInventory(): Promise<Inventory[]> {
  return inventory.map((item) => {
    const product = products.find((p) => p.id === item.productId);
    return {
      ...item,
      product,
    };
  });
}

export async function getOrders(): Promise<Order[]> {
  return [...orders].sort(
    (a, b) => new Date(b.orderDate).getTime() - new Date(a.orderDate).getTime()
  );
}

export async function createPurchase(
  customerName: string,
  items: { productId: number; quantity: number }[]
): Promise<Order> {
  if (!customerName || !items || items.length === 0) {
    throw new Error("请求无效：缺少客户名称或商品");
  }

  let totalAmount = 0;

  // 验证库存并计算总价
  for (const item of items) {
    const inventoryItem = inventory.find((i) => i.productId === item.productId);
    const product = products.find((p) => p.id === item.productId);

    if (!inventoryItem || !product) {
      throw new Error(`商品ID ${item.productId} 不存在`);
    }

    if (inventoryItem.quantity < item.quantity) {
      throw new Error(
        `商品 ${product.name} 库存不足. 可用: ${inventoryItem.quantity}`
      );
    }

    totalAmount += product.price * item.quantity;
  }

  // 创建订单
  const order: Order = {
    id: orders.length + 1,
    customerName,
    items,
    totalAmount,
    orderDate: new Date().toISOString(),
  };

  // 更新库存
  items.forEach((item) => {
    const inventoryItem = inventory.find(
      (i) => i.productId === item.productId
    )!;
    inventoryItem.quantity -= item.quantity;
  });

  orders.push(order);
  return order;
}
```

然后我们可以通过 MCP 的工具来将这些 API 接口暴露出去，如下所示：

```typescript
// mcp-server.ts
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { z } from "zod";
import {
  getProducts,
  getInventory,
  getOrders,
  createPurchase,
} from "./services/product-service.js";

export const server = new McpServer({
  name: "mcp-sse-demo",
  version: "1.0.0",
  description: "提供商品查询、库存管理和订单处理的MCP工具",
});

// 获取产品列表工具
server.tool("getProducts", "获取所有产品信息", {}, async () => {
  console.log("获取产品列表");
  const products = await getProducts();
  return {
    content: [
      {
        type: "text",
        text: JSON.stringify(products),
      },
    ],
  };
});

// 获取库存信息工具
server.tool("getInventory", "获取所有产品的库存信息", {}, async () => {
  console.log("获取库存信息");
  const inventory = await getInventory();
  return {
    content: [
      {
        type: "text",
        text: JSON.stringify(inventory),
      },
    ],
  };
});

// 获取订单列表工具
server.tool("getOrders", "获取所有订单信息", {}, async () => {
  console.log("获取订单列表");
  const orders = await getOrders();
  return {
    content: [
      {
        type: "text",
        text: JSON.stringify(orders),
      },
    ],
  };
});

// 购买商品工具
server.tool(
  "purchase",
  "购买商品",
  {
    items: z
      .array(
        z.object({
          productId: z.number().describe("商品ID"),
          quantity: z.number().describe("购买数量"),
        })
      )
      .describe("要购买的商品列表"),
    customerName: z.string().describe("客户姓名"),
  },
  async ({ items, customerName }) => {
    console.log("处理购买请求", { items, customerName });
    try {
      const order = await createPurchase(customerName, items);
      return {
        content: [
          {
            type: "text",
            text: JSON.stringify(order),
          },
        ],
      };
    } catch (error: any) {
      return {
        content: [
          {
            type: "text",
            text: JSON.stringify({ error: error.message }),
          },
        ],
      };
    }
  }
);
```

这里我们一共定义了 4 个工具，分别是：

- `getProducts`：获取所有产品信息
- `getInventory`：获取所有产品的库存信息
- `getOrders`：获取所有订单信息
- `purchase`：购买商品

如果是 Stdio 类型的 MCP 服务，那么我们就可以直接在命令行中使用这些工具了，但是我们现在需要使用 SSE 类型的 MCP 服务，所以我们还需要一个 MCP SSE 服务器来暴露这些工具。

## MCP SSE 服务器

接下来我们开始开发 MCP SSE 服务器，用于暴露产品微服务和订单微服务数据，作为使用 SSE 协议的工具。

```typescript
// mcp-sse-server.ts
import express, { Request, Response, NextFunction } from "express";
import cors from "cors";
import { SSEServerTransport } from "@modelcontextprotocol/sdk/server/sse.js";
import { server as mcpServer } from "./mcp-server.js"; // 重命名以避免命名冲突

const app = express();
app.use(
  cors({
    origin: process.env.ALLOWED_ORIGINS?.split(",") || "*",
    methods: ["GET", "POST"],
    allowedHeaders: ["Content-Type", "Authorization"],
  })
);

// 存储活跃连接
const connections = new Map();

// 健康检查端点
app.get("/health", (req, res) => {
  res.status(200).json({
    status: "ok",
    version: "1.0.0",
    uptime: process.uptime(),
    timestamp: new Date().toISOString(),
    connections: connections.size,
  });
});

// SSE 连接建立端点
app.get("/sse", async (req, res) => {
  // 实例化SSE传输对象
  const transport = new SSEServerTransport("/messages", res);
  // 获取sessionId
  const sessionId = transport.sessionId;
  console.log(`[${new Date().toISOString()}] 新的SSE连接建立: ${sessionId}`);

  // 注册连接
  connections.set(sessionId, transport);

  // 连接中断处理
  req.on("close", () => {
    console.log(`[${new Date().toISOString()}] SSE连接关闭: ${sessionId}`);
    connections.delete(sessionId);
  });

  // 将传输对象与MCP服务器连接
  await mcpServer.connect(transport);
  console.log(`[${new Date().toISOString()}] MCP服务器连接成功: ${sessionId}`);
});

// 接收客户端消息的端点
app.post("/messages", async (req: Request, res: Response) => {
  try {
    console.log(`[${new Date().toISOString()}] 收到客户端消息:`, req.query);
    const sessionId = req.query.sessionId as string;

    // 查找对应的SSE连接并处理消息
    if (connections.size > 0) {
      const transport: SSEServerTransport = connections.get(
        sessionId
      ) as SSEServerTransport;
      // 使用transport处理消息
      if (transport) {
        await transport.handlePostMessage(req, res);
      } else {
        throw new Error("没有活跃的SSE连接");
      }
    } else {
      throw new Error("没有活跃的SSE连接");
    }
  } catch (error: any) {
    console.error(`[${new Date().toISOString()}] 处理客户端消息失败:`, error);
    res.status(500).json({ error: "处理消息失败", message: error.message });
  }
});

// 优雅关闭所有连接
async function closeAllConnections() {
  console.log(
    `[${new Date().toISOString()}] 关闭所有连接 (${connections.size}个)`
  );
  for (const [id, transport] of connections.entries()) {
    try {
      // 发送关闭事件
      transport.res.write(
        'event: server_shutdown\ndata: {"reason": "Server is shutting down"}\n\n'
      );
      transport.res.end();
      console.log(`[${new Date().toISOString()}] 已关闭连接: ${id}`);
    } catch (error) {
      console.error(`[${new Date().toISOString()}] 关闭连接失败: ${id}`, error);
    }
  }
  connections.clear();
}

// 错误处理
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  console.error(`[${new Date().toISOString()}] 未处理的异常:`, err);
  res.status(500).json({ error: "服务器内部错误" });
});

// 优雅关闭
process.on("SIGTERM", async () => {
  console.log(`[${new Date().toISOString()}] 接收到SIGTERM信号，准备关闭`);
  await closeAllConnections();
  server.close(() => {
    console.log(`[${new Date().toISOString()}] 服务器已关闭`);
    process.exit(0);
  });
});

process.on("SIGINT", async () => {
  console.log(`[${new Date().toISOString()}] 接收到SIGINT信号，准备关闭`);
  await closeAllConnections();
  process.exit(0);
});

// 启动服务器
const port = process.env.PORT || 8083;
const server = app.listen(port, () => {
  console.log(
    `[${new Date().toISOString()}] 智能商城 MCP SSE 服务器已启动，地址: http://localhost:${port}`
  );
  console.log(`- SSE 连接端点: http://localhost:${port}/sse`);
  console.log(`- 消息处理端点: http://localhost:${port}/messages`);
  console.log(`- 健康检查端点: http://localhost:${port}/health`);
});
```

这里我们使用 Express 来暴露一个 SSE 连接端点 `/sse`，用于接收客户端消息。使用 `SSEServerTransport` 来创建一个 SSE 传输对象，并指定消息处理端点为 `/messages`。

```typescript
const transport = new SSEServerTransport("/messages", res);
```

传输对象创建后，我们就可以将传输对象与 MCP 服务器连接起来，如下所示：

```typescript
// 将传输对象与MCP服务器连接
await mcpServer.connect(transport);
```

这样我们就可以通过 SSE 连接端点 `/sse` 来接收客户端消息，并使用消息处理端点 `/messages` 来处理客户端消息，当接收到客户端消息后，在 `/messages` 端点中，我们需要使用 `transport` 对象来处理客户端消息：

```typescript
// 使用transport处理消息
await transport.handlePostMessage(req, res);
```

也就是我们常说的列出工具、调用工具等操作。

## MCP 客户端

接下来我们开始开发 MCP 客户端，用于连接到 MCP SSE 服务器，并使用 LLM 进行交互。客户端我们可以开发一个命令行客户端，也可以开发一个 Web 客户端。

对于命令行客户端前面我们已经介绍过了，唯一不同的是现在我们需要使用 SSE 协议来连接到 MCP SSE 服务器。

```typescript
// 创建MCP客户端
const mcpClient = new McpClient({
  name: "mcp-sse-demo",
  version: "1.0.0",
});

// 创建SSE传输对象
const transport = new SSEClientTransport(new URL(config.mcp.serverUrl));

// 连接到MCP服务器
await mcpClient.connect(transport);
```

然后其他操作和前面介绍的命令行客户端是一样的，也就是列出所有工具，然后将用户的问题和工具一起发给 LLM 进行处理。LLM 返回结果后，我们再根据结果来调用工具，将调用工具结果和历史消息一起发给 LLM 进行处理，得到最终结果。

对于 Web 客户端的话，和命令行客户端也基本一致，只是现在我们将这些处理过程放到一些接口里面去实现，然后通过 Web 页面来调用这些接口即可。

我们首先要初始化 MCP 客户端，然后获取所有工具，并转换工具格式为 Anthropic 所需的数组形式，然后创建 Anthropic 客户端。

```typescript
// 初始化MCP客户端
async function initMcpClient() {
  if (mcpClient) return;

  try {
    console.log("正在连接到MCP服务器...");
    mcpClient = new McpClient({
      name: "mcp-client",
      version: "1.0.0",
    });

    const transport = new SSEClientTransport(new URL(config.mcp.serverUrl));

    await mcpClient.connect(transport);
    const { tools } = await mcpClient.listTools();
    // 转换工具格式为Anthropic所需的数组形式
    anthropicTools = tools.map((tool: any) => {
      return {
        name: tool.name,
        description: tool.description,
        input_schema: tool.inputSchema,
      };
    });
    // 创建Anthropic客户端
    aiClient = createAnthropicClient(config);

    console.log("MCP客户端和工具已初始化完成");
  } catch (error) {
    console.error("初始化MCP客户端失败:", error);
    throw error;
  }
}
```

接着就根据我们自身的需求俩开发 API 接口，比如我们这里开发一个聊天接口，用于接收用户的问题，然后调用 MCP 客户端的工具，将工具调用结果和历史消息一起发给 LLM 进行处理，得到最终结果，代码如下所示：

```typescript
// API: 聊天请求
apiRouter.post("/chat", async (req, res) => {
  try {
    const { message, history = [] } = req.body;

    if (!message) {
      console.warn("请求中消息为空");
      return res.status(400).json({ error: "消息不能为空" });
    }

    // 构建消息历史
    const messages = [...history, { role: "user", content: message }];

    // 调用AI
    const response = await aiClient.messages.create({
      model: config.ai.defaultModel,
      messages,
      tools: anthropicTools,
      max_tokens: 1000,
    });

    // 处理工具调用
    const hasToolUse = response.content.some(
      (item) => item.type === "tool_use"
    );

    if (hasToolUse) {
      // 处理所有工具调用
      const toolResults = [];

      for (const content of response.content) {
        if (content.type === "tool_use") {
          const name = content.name;
          const toolInput = content.input as
            | { [x: string]: unknown }
            | undefined;

          try {
            // 调用MCP工具
            if (!mcpClient) {
              console.error("MCP客户端未初始化");
              throw new Error("MCP客户端未初始化");
            }
            console.log(`开始调用MCP工具: ${name}`);
            const toolResult = await mcpClient.callTool({
              name,
              arguments: toolInput,
            });

            toolResults.push({
              name,
              result: toolResult,
            });
          } catch (error: any) {
            console.error(`工具调用失败: ${name}`, error);
            toolResults.push({
              name,
              error: error.message,
            });
          }
        }
      }

      // 将工具结果发送回AI获取最终回复
      const finalResponse = await aiClient.messages.create({
        model: config.ai.defaultModel,
        messages: [
          ...messages,
          {
            role: "user",
            content: JSON.stringify(toolResults),
          },
        ],
        max_tokens: 1000,
      });

      const textResponse = finalResponse.content
        .filter((c) => c.type === "text")
        .map((c) => c.text)
        .join("\n");

      res.json({
        response: textResponse,
        toolCalls: toolResults,
      });
    } else {
      // 直接返回AI回复
      const textResponse = response.content
        .filter((c) => c.type === "text")
        .map((c) => c.text)
        .join("\n");

      res.json({
        response: textResponse,
        toolCalls: [],
      });
    }
  } catch (error: any) {
    console.error("聊天请求处理失败:", error);
    res.status(500).json({ error: error.message });
  }
});
```

这里的核心实现也比较简单，和命令行客户端基本一致，只是现在我们将这些处理过程放到一些接口里面去实现了而已。

## 使用

下面是命令行客户端的使用示例：

![](https://picdn.youdianzhishi.com/images/1743580511504.png)

当然我们也可以在 Cursor 中来使用，创建 `.cursor/mcp.json` 文件，然后添加如下内容：

```json
{
  "mcpServers": {
    "products-sse": {
      "url": "http://localhost:8083/sse"
    }
  }
}
```

然后在 Cursor 的设置页面我们就可以看到这个 MCP 服务，然后就可以在 Cursor 中来使用这个 MCP 服务了。

![](https://picdn.youdianzhishi.com/images/1743580805254.png)

下面是我们开发的 Web 客户端的使用示例：

![](https://picdn.youdianzhishi.com/images/1743580945607.png)

![](https://picdn.youdianzhishi.com/images/1743581042949.png)

## 调试

同样我们可以使用 `npx @modelcontextprotocol/inspector` 命令来调试我们的 SSE 服务：

```bash
$ npx @modelcontextprotocol/inspector   
Starting MCP inspector...
⚙️ Proxy server listening on port 6277
🔍 MCP Inspector is up and running at http://127.0.0.1:6274 🚀
```

然后在浏览器中打开上面地址即可，选择 SSE，配置上我们的 SSE 地址即可测试：

![](https://picdn.youdianzhishi.com/images/1743675135602.png)

## 总结

当 LLM 决定触发对用户工具的调用时，工具描述的质量至关重要：

- **精确描述**：确保每个工具的描述清晰明确，包含关键词以便 LLM 正确识别何时使用该工具
- **避免冲突**：不要提供多个功能相似的工具，这可能导致 LLM 选择错误
- **测试验证**：在部署前使用各种用户查询场景测试工具调用的准确性

MCP 服务器可以使用多种技术实现：

- Python SDK
- TypeScript/JavaScript
- 其他编程语言

选择应基于团队熟悉度和现有技术栈。

另外将 AI 助手与 MCP 服务器集成到现有微服务架构中具有以下优势：

1. **实时数据**：通过 SSE（服务器发送事件）提供实时或近实时更新，对库存信息、订单状态等动态数据尤为重要
2. **可扩展性**：系统各部分可独立扩展，例如频繁使用的库存检查服务可单独扩容
3. **韧性**：单个微服务失败不会影响整个系统运行，确保系统稳定性
4. **灵活性**：不同团队可独立处理系统各部分，必要时使用不同技术栈
5. **高效通信**：SSE 比持续轮询更高效，只在数据变化时发送更新
6. **用户体验提升**：实时更新和快速响应提高客户满意度
7. **简化客户端**：客户端代码更简洁，无需复杂轮询机制，只需监听服务器事件

当然如果想要在生产环境中去使用，那么我们还需要考虑以下几点：

- 进行全面测试以识别潜在错误
- 设计故障恢复机制
- 实现监控系统跟踪工具调用性能和准确性
- 考虑添加缓存层减轻后端服务负担

通过以上实践，我们可以构建一个高效、可靠的基于 MCP 的智能商城服务助手，为用户提供实时、个性化的购物体验。
