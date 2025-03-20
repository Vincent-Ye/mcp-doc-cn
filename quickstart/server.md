---
标题: "针对服务器开发者"
描述: "开始构建你自己的服务器，用于在 Claude for Desktop 和其他客户端中使用。"
---

在本教程中，我们将构建一个简单的 MCP 天气服务器，并将其连接到一个主机（Claude for Desktop）。我们将从基本设置开始，然后逐步实现更复杂的用例。

### 我们将构建什么

许多大型语言模型 (LLM) 目前无法获取天气预报和恶劣天气警报。让我们使用 MCP 来解决这个问题！

我们将构建一个服务器，公开两个工具：`get-alerts` 和 `get-forecast`。然后我们将服务器连接到 MCP 主机（在本例中是 Claude for Desktop）：

<Frame>
  <img src="/images/weather-alerts.png" />
</Frame>
<Frame>
  <img src="/images/current-weather.png" />
</Frame>

<Note>
服务器可以连接到任何客户端。为了方便起见，这里我们选择了 Claude for Desktop，但我们也提供了关于[构建你自己的客户端](/quickstart/client)的指南，以及[其他客户端的列表](/clients)。
</Note>

<Accordion title="为什么选择 Claude for Desktop 而不是 Claude.ai?">
  因为服务器是本地运行的，目前 MCP 仅支持桌面主机。远程主机仍在开发中。
</Accordion>

### MCP 核心概念

MCP 服务器可以提供三种主要功能：

1. **资源**：可以被客户端读取的数据（例如，API 响应或文件内容）
2. **工具**：可以被 LLM 调用的函数（需用户授权）
3. **提示**：帮助用户完成特定任务的预先编写模板

本教程将重点放在工具功能上。

<Tabs>
<Tab title="Python">

让我们开始构建我们的天气服务器吧！[你可以在此获取完整代码。](https://github.com/modelcontextprotocol/quickstart-resources/tree/main/weather-server-python)

### 前置知识

本快速入门假设你已熟悉以下内容：
- Python 编程语言
- 类似 Claude 的大型语言模型 (LLM)

### 系统要求

- 已安装 Python 3.10 或更高版本
- 必须使用 MCP Python SDK 1.2.0 或更高版本。

### 设置你的开发环境

首先，让我们安装 `uv` 并设置 Python 项目和环境：

<CodeGroup>

```bash MacOS/Linux
curl -LsSf https://astral.sh/uv/install.sh | sh
```

```powershell Windows
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

</CodeGroup>

安装完成后，请确保重新启动你的终端，以确保 `uv` 命令可以被正确识别。

接下来，创建和设置我们的项目：

<CodeGroup>
```bash MacOS/Linux
# 创建项目文件夹
uv init weather
cd weather

# 创建虚拟环境并激活
uv venv
source .venv/bin/activate

# 安装依赖
uv add "mcp[cli]" httpx

# 创建服务器文件
touch weather.py
```

```powershell Windows
# 创建项目文件夹
uv init weather
cd weather

# 创建虚拟环境并激活
uv venv
.venv\Scripts\activate

# 安装依赖
uv add mcp[cli] httpx

# 创建服务器文件
new-item weather.py
```
</CodeGroup>

现在让我们开始构建你的服务器。

## 构建你的服务器

### 导入库和设置实例

在 `weather.py` 文件顶部添加以下内容：
```python
from typing import Any
import httpx
from mcp.server.fastmcp import FastMCP

# 初始化 FastMCP 服务器
mcp = FastMCP("weather")

# 常量
NWS_API_BASE = "https://api.weather.gov"
USER_AGENT = "weather-app/1.0"
```

`FastMCP` 类使用 Python 的类型提示和文档字符串来自动生成工具定义，使创建和维护 MCP 工具变得更加简单。

### 辅助函数

接下来，让我们添加从国家天气服务 (NWS) API 查询和格式化数据的辅助函数：

下面是翻译成中文的文本：

```python
async def make_nws_request(url: str) -> dict[str, Any] | None:
    """向国家气象服务（NWS）API发出请求并进行适当的错误处理。"""
    headers = {
        "User-Agent": USER_AGENT,
        "Accept": "application/geo+json"
    }
    async with httpx.AsyncClient() as client:
        try:
            response = await client.get(url, headers=headers, timeout=30.0)
            response.raise_for_status()
            return response.json()
        except Exception:
            return None

def format_alert(feature: dict) -> str:
    """将一个天气警报的特征格式化为可读字符串。"""
    props = feature["properties"]
    return f"""
事件: {props.get('event', '未知')}
区域: {props.get('areaDesc', '未知')}
严重程度: {props.get('severity', '未知')}
描述: {props.get('description', '暂无描述')}
说明: {props.get('instruction', '未提供具体指导')}
"""
```

### 实现工具执行

工具执行处理程序负责实际执行每个工具的逻辑。让我们来添加它：

```python
@mcp.tool()
async def get_alerts(state: str) -> str:
    """获取美国某州的天气警报。

    参数:
        state: 两字母的美国州代码（例如 CA, NY）
    """
    url = f"{NWS_API_BASE}/alerts/active/area/{state}"
    data = await make_nws_request(url)

    if not data or "features" not in data:
        return "无法获取警报或未发现警报。"

    if not data["features"]:
        return "该州没有活跃的警报。"

    alerts = [format_alert(feature) for feature in data["features"]]
    return "\n---\n".join(alerts)

@mcp.tool()
async def get_forecast(latitude: float, longitude: float) -> str:
    """获取某地的天气预报。

    参数:
        latitude: 该地点的纬度
        longitude: 该地点的经度
    """
    # 首先获取预测网格端点
    points_url = f"{NWS_API_BASE}/points/{latitude},{longitude}"
    points_data = await make_nws_request(points_url)

    if not points_data:
        return "无法获取该地点的预测数据。"

    # 从点响应中获取预测 URL
    forecast_url = points_data["properties"]["forecast"]
    forecast_data = await make_nws_request(forecast_url)

    if not forecast_data:
        return "无法获取详细的天气预报。"

    # 将预测周期格式化为可读的预测
    periods = forecast_data["properties"]["periods"]
    forecasts = []
    for period in periods[:5]:  # 仅显示接下来的 5 个时间段
        forecast = f"""
{period['name']}:
温度: {period['temperature']}°{period['temperatureUnit']}
风速: {period['windSpeed']} {period['windDirection']}
预测: {period['detailedForecast']}
"""
        forecasts.append(forecast)

    return "\n---\n".join(forecasts)
```

### 运行服务器

最后，让我们初始化并运行服务器：

```python
if __name__ == "__main__":
    # 初始化并运行服务器
    mcp.run(transport='stdio')
```

你的服务器已完成！运行 `uv run weather.py`，确认一切工作正常。

### 使用 Claude for Desktop 测试你的服务器

<注意>
Claude for Desktop 目前尚不支持 Linux。Linux 用户可以继续阅读 [构建客户端](/quickstart/client) 教程以构建连接到我们刚构建的服务器的 MCP 客户端。
</注意>

首先，确保你已经安装了 Claude for Desktop。[你可以在这里下载最新版本。](https://claude.ai/download) 如果你已经安装 Claude for Desktop，请确保你更新到了最新版本。

我们需要为你想使用的 MCP 服务器配置 Claude for Desktop。为此，请在文本编辑器中打开 `~/Library/Application Support/Claude/claude_desktop_config.json`。如果文件不存在，请确保创建该文件。

例如，如果你安装了 [VS Code](https://code.visualstudio.com/)...

```html
<Tabs>
<Tab title="MacOS/Linux">
```bash
code ~/Library/Application\ Support/Claude/claude_desktop_config.json
```
</Tab>
<Tab title="Windows">
```powershell
code $env:AppData\Claude\claude_desktop_config.json
```
</Tab>
</Tabs>

然后，在 `mcpServers` 键中添加你的服务器配置。只有至少有一个服务器配置正确，Claude 桌面版中才会显示 MCP 用户界面元素。

这里，我们会像下面这样配置一个单独的天气服务器：

<Tabs>
<Tab title="MacOS/Linux">
```json Python
{
    "mcpServers": {
        "weather": {
            "command": "uv",
            "args": [
                "--directory",
                "/ABSOLUTE/PATH/TO/PARENT/FOLDER/weather",
                "run",
                "weather.py"
            ]
        }
    }
}
```
</Tab>
<Tab title="Windows">
```json Python
{
    "mcpServers": {
        "weather": {
            "command": "uv",
            "args": [
                "--directory",
                "C:\\ABSOLUTE\\PATH\\TO\\PARENT\\FOLDER\\weather",
                "run",
                "weather.py"
            ]
        }
    }
}
```
</Tab>
</Tabs>

<Warning>
你可能需要在 `command` 字段中输入 `uv` 可执行文件的完整路径。可以通过在 MacOS/Linux 下运行 `which uv` 或在 Windows 下运行 `where uv` 来获取该路径。
</Warning>

<Note>
确保传入的是服务器文件的绝对路径。
</Note>

这段配置告诉 Claude 桌面版：
1. 存在一个名为 "weather" 的 MCP 服务器；
2. 启动它时运行 `uv --directory /ABSOLUTE/PATH/TO/PARENT/FOLDER/weather run weather.py`。

保存文件后，重启 **Claude 桌面版**。

<Tab title='Node'>
让我们开始构建我们的天气服务器！[你可以在这里查看完整的代码实现。](https://github.com/modelcontextprotocol/quickstart-resources/tree/main/weather-server-typescript)

### 前置知识

此快速启动假设你已熟悉以下内容：
- TypeScript
- 类似 Claude 的大型语言模型 (LLM)

### 系统要求

对于 TypeScript，需要确保安装了最新版本的 Node.js。

### 设置开发环境

首先，如果你尚未安装 Node.js 和 npm，请从 [nodejs.org](https://nodejs.org/) 下载并安装。验证你的 Node.js 安装：
```bash
node --version
npm --version
```
本教程需要 Node.js 版本 16 或更高。

现在，创建并设置我们的项目：

<CodeGroup>
```bash MacOS/Linux
# 为项目创建一个新文件夹
mkdir weather
cd weather

# 初始化一个新的 npm 项目
npm init -y

# 安装依赖包
npm install @modelcontextprotocol/sdk zod
npm install -D @types/node typescript

# 创建项目文件
mkdir src
touch src/index.ts
```

```powershell Windows
# 为项目创建一个新文件夹
md weather
cd weather

# 初始化一个新的 npm 项目
npm init -y

# 安装依赖包
npm install @modelcontextprotocol/sdk zod
npm install -D @types/node typescript

# 创建项目文件
md src
new-item src\index.ts
```
</CodeGroup>

在 `package.json` 中添加 `type: "module"` 和一个构建脚本：

```json package.json
{
  "type": "module",
  "bin": {
    "weather": "./build/index.js"
  },
  "scripts": {
    "build": "tsc && chmod 755 build/index.js"
  },
  "files": [
    "build"
  ],
}
```

在项目根目录创建 `tsconfig.json` 文件：

```json tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "outDir": "./build",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}
```

现在我们开始构建你的服务器。

## 构建服务器

### 导入包并设置实例
```

将以下内容翻译成中文：

请将以下内容添加到你的 `src/index.ts` 文件的顶部：
```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const NWS_API_BASE = "https://api.weather.gov"; // 国家天气服务的API基础地址
const USER_AGENT = "weather-app/1.0"; // 用户代理标识

// 创建服务器实例
const server = new McpServer({
  name: "weather", // 服务名称
  version: "1.0.0", // 服务版本
});
```

### 辅助函数

接下来，我们需要添加一些辅助函数，用于从国家天气服务（National Weather Service）API查询和格式化数据：

```typescript
// 用于发送国家天气服务API请求的辅助函数
async function makeNWSRequest<T>(url: string): Promise<T | null> {
  const headers = {
    "User-Agent": USER_AGENT, // 用户代理头信息
    Accept: "application/geo+json", // 期待的响应内容类型
  };

  try {
    const response = await fetch(url, { headers });
    if (!response.ok) {
      throw new Error(`HTTP错误！状态码：${response.status}`);
    }
    return (await response.json()) as T;
  } catch (error) {
    console.error("发送国家天气服务请求时发生错误：", error);
    return null;
  }
}

// 警报数据的接口定义
interface AlertFeature {
  properties: {
    event?: string; // 事件类型
    areaDesc?: string; // 相关区域描述
    severity?: string; // 严重性
    status?: string; // 状态
    headline?: string; // 标题
  };
}

// 格式化警报数据
function formatAlert(feature: AlertFeature): string {
  const props = feature.properties;
  return [
    `事件类型：${props.event || "未知"}`,
    `相关区域：${props.areaDesc || "未知"}`,
    `严重性：${props.severity || "未知"}`,
    `状态：${props.status || "未知"}`,
    `标题：${props.headline || "无标题"}`,
    "---",
  ].join("\n");
}

// 天气预报时间段信息接口定义
interface ForecastPeriod {
  name?: string; // 时间段名称
  temperature?: number; // 温度
  temperatureUnit?: string; // 温度单位
  windSpeed?: string; // 风速
  windDirection?: string; // 风向
  shortForecast?: string; // 简短的天气预报
}

// 警报响应接口定义
interface AlertsResponse {
  features: AlertFeature[];
}

// 网格点响应接口定义
interface PointsResponse {
  properties: {
    forecast?: string; // 预测链接
  };
}

// 天气预报响应接口定义
interface ForecastResponse {
  properties: {
    periods: ForecastPeriod[]; // 预测时间段信息
  };
}
```

### 实现工具执行

工具的执行处理器负责实际运行每个工具的逻辑。让我们添加它：

```typescript
// 注册天气工具
server.tool(
  "get-alerts", // 工具名称
  "获取某个州的天气警报", // 工具描述
  {
    state: z.string().length(2).describe("州的两字母代码（例如：CA, NY）"), // 输入参数
  },
  async ({ state }) => {
    const stateCode = state.toUpperCase(); // 转换为大写字母
    const alertsUrl = `${NWS_API_BASE}/alerts?area=${stateCode}`;
    const alertsData = await makeNWSRequest<AlertsResponse>(alertsUrl);

    if (!alertsData) {
      return {
        content: [
          {
            type: "text",
            text: "无法获取警报数据",
          },
        ],
      };
    }

    const features = alertsData.features || [];
    if (features.length === 0) {
      return {
        content: [
          {
            type: "text",
            text: `当前没有 ${stateCode} 的活动警报`,
          },
        ],
      };
    }

    const formattedAlerts = features.map(formatAlert);
    const alertsText = `当前 ${stateCode} 的活动警报：\n\n${formattedAlerts.join("\n")}`;

    return {
      content: [
        {
          type: "text",
          text: alertsText,
        },
      ],
    };
  },
);

server.tool(
  "get-forecast", // 工具名称
  "获取某地的天气预报", // 工具描述
  {
    latitude: z.number().min(-90).max(90).describe("地点的纬度"), // 输入参数
    longitude: z.number().min(-180).max(180).describe("地点的经度"), // 输入参数
  },
  async ({ latitude, longitude }) => {
    // 获取网格点数据
    const pointsUrl = `${NWS_API_BASE}/points/${latitude.toFixed(4)},${longitude.toFixed(4)}`;
    const pointsData = await makeNWSRequest<PointsResponse>(pointsUrl); 
```

```typescript
if (!pointsData) {
  return {
    content: [
      {
        type: "text",
        text: `无法获取坐标的网格点数据：${latitude}, ${longitude}。此位置可能不受 NWS API 支持（仅支持美国区域）。`,
      },
    ],
  };
}

const forecastUrl = pointsData.properties?.forecast;
if (!forecastUrl) {
  return {
    content: [
      {
        type: "text",
        text: "无法从网格点数据中获取预报 URL",
      },
    ],
  };
}

// 获取预报数据
const forecastData = await makeNWSRequest<ForecastResponse>(forecastUrl);
if (!forecastData) {
  return {
    content: [
      {
        type: "text",
        text: "无法检索预报数据",
      },
    ],
  };
}

const periods = forecastData.properties?.periods || [];
if (periods.length === 0) {
  return {
    content: [
      {
        type: "text",
        text: "暂无可用的预报数据",
      },
    ],
  };
}

// 格式化预报时段
const formattedForecast = periods.map((period: ForecastPeriod) =>
  [
    `${period.name || "未知"}:`,
    `温度: ${period.temperature || "未知"}°${period.temperatureUnit || "F"}`,
    `风速: ${period.windSpeed || "未知"} ${period.windDirection || ""}`,
    `${period.shortForecast || "无可用预报"}`,
    "---",
  ].join("\n"),
);

const forecastText = `位于 ${latitude}, ${longitude} 的天气预报：\n\n${formattedForecast.join("\n")}`;

return {
  content: [
    {
      type: "text",
      text: forecastText,
    },
  ],
};
},
);

// 启动服务器

async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error("天气 MCP 服务器正在标准输入输出上运行");
}

main().catch((error) => {
  console.error("main() 中发生致命错误：", error);
  process.exit(1);
});
```

确保运行 `npm run build` 来构建您的服务器！这是连接服务器的关键步骤。

### 在 Claude for Desktop 中测试您的服务器

<注意>
Claude for Desktop 目前尚未支持 Linux。Linux 用户可继续前往 [构建客户端](/quickstart/client) 教程以构建与我们刚刚搭建的服务器连接的 MCP 客户端。
</注意>

首先，请确保您已安装 Claude for Desktop。[您可以在此处下载最新版本。](https://claude.ai/download) 如果您已经安装了 Claude for Desktop，请**确保已更新至最新版本**。

我们需要配置 Claude for Desktop，以使用所需的 MCP 服务器。为此，请在文本编辑器中打开位于 `~/Library/Application Support/Claude/claude_desktop_config.json` 的 Claude for Desktop 配置文件。如果文件不存在，请确保创建它。

例如，如果您安装了 [VS Code](https://code.visualstudio.com/)：

<Tabs>
<Tab title="MacOS/Linux">
```bash
code ~/Library/Application\ Support/Claude/claude_desktop_config.json
```
</Tab>
<Tab title="Windows">
```powershell
code $env:AppData\Claude\claude_desktop_config.json
```
</Tab>
</Tabs>

然后，您可以在 `mcpServers` 键中添加服务器。如果至少有一个服务器正确配置，Claude for Desktop 的 MCP 界面元素将会显示。

例如，我们可以像这样添加单一的天气服务器：

下面是将该文本翻译成中文后的版本：

<Tabs>
<Tab title="MacOS/Linux">
<CodeGroup>
```json Node
{
    "mcpServers": {
        "weather": {
            "command": "node",
            "args": [
                "/ABSOLUTE/PATH/TO/PARENT/FOLDER/weather/build/index.js"
            ]
        }
    }
}
```
</CodeGroup>
</Tab>
<Tab title="Windows">
<CodeGroup>
```json Node
{
    "mcpServers": {
        "weather": {
            "command": "node",
            "args": [
                "C:\\PATH\\TO\\PARENT\\FOLDER\\weather\\build\\index.js"
            ]
        }
    }
}
```
</CodeGroup>
</Tab>
</Tabs>

这段配置告诉 Claude for Desktop：
1. 存在一个名为 "weather" 的 MCP 服务器。
2. 通过运行 `node /ABSOLUTE/PATH/TO/PARENT/FOLDER/weather/build/index.js` 启动它。

保存文件后，重启 **Claude for Desktop**。

</Tab>
<Tab title='Java'>

<Note>
这是基于 Spring AI MCP 自动配置和启动器的快速入门示例。  
如果需要手动创建同步和异步 MCP 服务器，请参考 [Java SDK Server](/sdk/java/mcp-server) 文档。
</Note>

让我们开始构建我们的天气服务服务器吧！  
[完整代码可以在这里找到。](https://github.com/spring-projects/spring-ai-examples/tree/main/model-context-protocol/weather/starter-stdio-server)

更多信息，请参考 [MCP 服务器启动器文档](https://docs.spring.io/spring-ai/reference/api/mcp/mcp-server-boot-starter-docs.html)。  
如果需要手动实现 MCP 服务器，请查看 [MCP 服务器 Java SDK 文档](/sdk/java/mcp-server)。

### 系统要求

- 安装 Java 17 或更高版本。
- [Spring Boot 3.3.x](https://docs.spring.io/spring-boot/installing.html) 或更高版本。

### 设置环境

使用 [Spring 初始化器](https://start.spring.io/) 快速启动项目。

需要添加以下依赖：

<Tabs>
  <Tab title="Maven">
  ```xml
  <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-mcp-server-spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-web</artifactId>
        </dependency>
  </dependencies>
  ```
  </Tab>
  <Tab title="Gradle">
  ```groovy
  dependencies {
    implementation platform("org.springframework.ai:spring-ai-mcp-server-spring-boot-starter")
    implementation platform("org.springframework:spring-web")   
  }
  ```
  </Tab>
</Tabs>

然后通过设置应用程序属性来配置您的应用程序：

<CodeGroup>

```bash application.properties
spring.main.bannerMode=off
logging.pattern.console=
```

```yaml application.yml
logging:
  pattern:
    console:
spring:
  main:
    banner-mode: off
```
</CodeGroup>

[服务器配置属性](https://docs.spring.io/spring-ai/reference/api/mcp/mcp-server-boot-starter-docs.html#_configuration_properties) 文档记录了可用的所有属性。

现在，让我们开始构建服务器。

## 构建您的服务器

### 天气服务

让我们实现一个 [WeatherService.java](https://github.com/spring-projects/spring-ai-examples/blob/main/model-context-protocol/weather/starter-stdio-server/src/main/java/org/springframework/ai/mcp/sample/server/WeatherService.java)，该类使用 REST 客户端从国家天气服务 API 查询数据：

```java
@Service
public class WeatherService {

	private final RestClient restClient;

	public WeatherService() {
		this.restClient = RestClient.builder()
			.baseUrl("https://api.weather.gov")
			.defaultHeader("Accept", "application/geo+json")
			.defaultHeader("User-Agent", "WeatherApiClient/1.0 (your@email.com)")
			.build();
	}
```

将以下文本翻译成中文：

---

### 工具描述

```java
@Tool(description = "获取特定纬度/经度的天气预报")
public String getWeatherForecastByLocation(
    double latitude,   // 纬度
    double longitude   // 经度
) {
    // 返回详细的天气预报，包括：
    // - 温度及其单位
    // - 风速及风向
    // - 详细的预报描述
}

@Tool(description = "获取指定美国州的天气警报")
public String getAlerts(
    @ToolParam(description = "美国州的两位字母代码（例如 CA, NY）") String state)
) {
    // 返回当前生效的警报信息，包括：
    // - 事件类型
    // - 受影响区域
    // - 严重程度
    // - 描述信息
    // - 安全指引
}

// ......
}
```

`@Service` 注解会自动将服务注册到应用程序上下文中。
Spring AI 的 `@Tool` 注解方便创建并维护 MCP 工具。

自动配置功能会将这些工具自动注册到 MCP 服务器中。

---

### 创建启动应用程序

```java
@SpringBootApplication
public class McpServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(McpServerApplication.class, args);
    }

    @Bean
    public ToolCallbackProvider weatherTools(WeatherService weatherService) {
        return MethodToolCallbackProvider.builder().toolObjects(weatherService).build();
    }
}
```

利用 `MethodToolCallbackProvider` 工具，将 `@Tool` 注解生成的工具转换为 MCP 服务器可操作的回调。

---

### 启动服务器

最后，构建服务器：

```bash
./mvnw clean install
```

此命令会在 `target` 文件夹中生成一个名为 `mcp-weather-stdio-server-0.0.1-SNAPSHOT.jar` 的文件。

现在，从现有的 MCP 主机（例如 Claude for Desktop）测试你的服务器。

---

## 使用 Claude for Desktop 测试您的服务器

<注意>
Claude for Desktop 目前尚未支持 Linux。
</注意>

首先，确认你已安装 Claude for Desktop。  
[你可以在这里下载安装最新版本。](https://claude.ai/download) 如果你已安装，请确保更新到最新版本。

需要为 Claude for Desktop 配置希望使用的 MCP 服务器。  
在文本编辑器中打开 Claude for Desktop 的配置文件，路径为：  
`~/Library/Application Support/Claude/claude_desktop_config.json`。  
如果文件不存在，请新建一个。

例如，如果你安装了 [VS Code](https://code.visualstudio.com/):

<Tabs>
  <Tab title="MacOS/Linux">
  ```bash
  code ~/Library/Application\ Support/Claude/claude_desktop_config.json
  ```
  </Tab>
  <Tab title="Windows">
  ```powershell
  code $env:AppData\Claude\claude_desktop_config.json
  ```
  </Tab>
</Tabs>

然后，将服务器添加到 `mcpServers` 键下。  
只有至少配置了一个服务器，Claude for Desktop 的 MCP 界面元素才会显示。

以下是添加单个天气服务器的示例配置：

<Tabs>
  <Tab title="MacOS/Linux">
  ```json
  {
    "mcpServers": {
      "spring-ai-mcp-weather": {
        "command": "java",
        "args": [
          "-Dspring.ai.mcp.server.stdio=true",
          "-jar",
          "/ABSOLUTE/PATH/TO/PARENT/FOLDER/mcp-weather-stdio-server-0.0.1-SNAPSHOT.jar"
        ]
      }
    }
  }
  ```
  </Tab>

  <Tab title="Windows">
  ```json
  {
    "mcpServers": {
      "spring-ai-mcp-weather": {
        "command": "java",
        "args": [
          "-Dspring.ai.mcp.server.transport=STDIO",
          "-jar",
          "C:\\ABSOLUTE\\PATH\\TO\\PARENT\\FOLDER\\weather\\mcp-weather-stdio-server-0.0.1-SNAPSHOT.jar"
        ]
      }
    }
  }
  ```
  </Tab>
</Tabs>

<注意>
请务必传入服务器的绝对路径。
</注意>

上述配置告诉 Claude for Desktop：
1. 存在名为 `spring-ai-mcp-weather` 的 MCP 服务器。
2. 通过运行命令 `java -jar /ABSOLUTE/PATH/TO/PARENT/FOLDER/mcp-weather-stdio-server-0.0.1-SNAPSHOT.jar` 启动该服务器。

保存文件并重新启动 **Claude for Desktop**。

## 使用 Java 客户端测试你的服务器

### 手动创建一个 MCP 客户端

使用 `McpClient` 连接到服务器：

```java
var stdioParams = ServerParameters.builder("java")
  .args("-jar", "/ABSOLUTE/PATH/TO/PARENT/FOLDER/mcp-weather-stdio-server-0.0.1-SNAPSHOT.jar")
  .build();

var stdioTransport = new StdioClientTransport(stdioParams);

var mcpClient = McpClient.sync(stdioTransport).build();

mcpClient.initialize();

ListToolsResult toolsList = mcpClient.listTools();

CallToolResult weather = mcpClient.callTool(
  new CallToolRequest("getWeatherForecastByLocation",
      Map.of("latitude", "47.6062", "longitude", "-122.3321")));

CallToolResult alert = mcpClient.callTool(
  new CallToolRequest("getAlerts", Map.of("state", "NY")));

mcpClient.closeGracefully();
```

### 使用 MCP Client Boot Starter

通过使用 `spring-ai-mcp-client-spring-boot-starter` 依赖创建一个新的启动器应用程序：

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-mcp-client-spring-boot-starter</artifactId>
</dependency>
```

并设置 `spring.ai.mcp.client.stdio.servers-configuration` 属性指向你的 `claude_desktop_config.json` 文件。
你可以重复使用现有的 Anthropic Desktop 配置文件：

```properties
spring.ai.mcp.client.stdio.servers-configuration=file:PATH/TO/claude_desktop_config.json
```

当你启动客户端应用程序时，自动配置功能会使用 `claude_desktop_config.json` 文件自动创建 MCP 客户端。

有关详细信息，请参阅 [MCP Client Boot Starters](https://docs.spring.io/spring-ai/reference/api/mcp/mcp-server-boot-client-docs.html) 的参考文档。

## 更多 Java MCP Server 示例

[starter-webflux-server](https://github.com/spring-projects/spring-ai-examples/tree/main/model-context-protocol/weather/starter-webflux-server) 展示了如何使用 SSE 传输创建一个 MCP 服务器。
它展示了如何使用 Spring Boot 的自动配置功能定义和注册 MCP 工具、资源及提示。

</Tab>
</Tabs>

### 使用命令进行测试

确保 Claude for Desktop 能识别出我们在 `weather` 服务器中公开的两个工具。可以通过寻找锤子图标 <img src="/images/claude-desktop-mcp-hammer-icon.svg" style={{display: 'inline', margin: 0, height: '1.3em'}} /> 来完成此操作：

<Frame>
  <img src="/images/visual-indicator-mcp-tools.png" />
</Frame>

点击锤子图标后，你应能看到列出的两个工具：

<Frame>
  <img src="/images/available-mcp-tools.png" />
</Frame>

如果你的服务器未被 Claude for Desktop 识别，请转到 [故障排除](#troubleshooting) 部分以获取调试技巧。

如果锤子图标出现，你现在可以在 Claude for Desktop 中运行以下命令来测试服务器：

- Sacramento 的天气如何？
- Texas 有哪些活跃的天气预警？

<Frame>
  <img src="/images/current-weather.png" />
</Frame>
<Frame>
  <img src="/images/weather-alerts.png" />
</Frame>

<Note>
由于使用的是美国国家气象局，这些查询仅适用于美国的位置。
</Note>

## 底层发生了什么？

当你提问时：

1. 客户端将你的问题发送给 Claude。
2. Claude 分析可用的工具，并决定使用哪些工具。
3. 客户端通过 MCP 服务器执行选定的工具。
4. 结果被返回给 Claude。
5. Claude 构造一个自然语言响应。
6. 响应显示给你！

## 故障排除

<AccordionGroup>
<Accordion title="Claude for Desktop 集成问题">
**从 Claude for Desktop 获取日志**

与 MCP 相关的 Claude.app 日志会写入 `~/Library/Logs/Claude` 下的日志文件：

- `mcp.log` 包含有关 MCP 连接和连接失败的一般日志。
- 名为 `mcp-server-SERVERNAME.log` 的文件包含来自指定服务器的错误（stderr）日志。

您可以运行以下命令来列出最近的日志并实时查看新增日志：
```bash
# 检查 Claude 的日志以排查错误
tail -n 20 -f ~/Library/Logs/Claude/mcp*.log
```

**Claude 中未显示服务器**

1. 检查您的 `claude_desktop_config.json` 文件语法是否正确  
2. 确保项目路径是绝对路径而不是相对路径  
3. 完全重新启动 Claude for Desktop  

**工具调用悄然失败**

如果 Claude 尝试使用工具却失败了：  

1. 检查 Claude 的日志以查看错误  
2. 确认您的服务器构建并运行时没有错误  
3. 尝试重启 Claude for Desktop  

**这些都不奏效时，该怎么办？**

请参考我们的[调试指南](/docs/tools/debugging)，以获取更好的调试工具和更详细的指导。

</Accordion>
<Accordion title="天气 API 问题">
**错误：无法获取网格点数据**

通常意味着以下原因之一：
1. 坐标在美国境外  
2. NWS API 出现问题  
3. 您的请求频率受限  

解决方法：

- 确保您使用的是美国坐标  
- 在请求之间添加一个短暂的延迟  
- 检查 NWS API 的状态页面  

**错误：暂无 [州] 的活动预警**

这并不是一个错误——仅仅表示该州当前没有天气预警。尝试其他州，或在强天气时期检查预警。  
</Accordion>

</AccordionGroup>

<Note>
若需更高级的故障排除，请查看我们的[调试 MCP 的指南](/docs/tools/debugging)。
</Note>

## 后续步骤

<CardGroup cols={2}>
  <Card
    title="构建客户端"
    icon="outlet"
    href="/quickstart/client"
  >
    了解如何构建您自己的 MCP 客户端以连接到您的服务器
  </Card>
  <Card
    title="示例服务器"
    icon="grid"
    href="/examples"
  >
    浏览官方 MCP 服务器和实现的展示库
  </Card>
  <Card
    title="调试指南"
    icon="bug"
    href="/docs/tools/debugging"
  >
    学习如何有效调试 MCP 服务器和集成
  </Card>
  <Card
    title="使用 LLM 构建 MCP"
    icon="comments"
    href="/tutorials/building-mcp-with-llms"
  >
    学习如何使用诸如 Claude 这样的 LLM 来加速您的 MCP 开发
  </Card>
</CardGroup>