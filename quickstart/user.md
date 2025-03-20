---
标题: "适用于 Claude 桌面用户"
描述: "开始使用 Claude for Desktop 的预构建服务器。"
---

在本教程中，您将扩展 [Claude for Desktop](https://claude.ai/download)，使它能够读取您计算机的文件系统、创建新文件、移动文件，甚至搜索文件。

<Frame>
  <img src="/images/quickstart-filesystem.png" />
</Frame>

不用担心 —— 在执行这些操作之前，它会征求您的许可！

## 1. 下载 Claude for Desktop

首先下载 [Claude for Desktop](https://claude.ai/download)，选择 macOS 或 Windows。（目前 Claude for Desktop 尚不支持 Linux。）

按照安装说明操作。

如果您已经安装了 Claude for Desktop，请确保它是最新版本。点击计算机上的 Claude 菜单，选择 "Check for Updates..."。

<Accordion title="为什么使用 Claude for Desktop 而不是 Claude.ai？">
  由于服务器在本地运行，目前 MCP 仅支持桌面主机。远程主机正在积极开发中。
</Accordion>

## 2. 添加文件系统 MCP 服务器

为了添加文件系统功能，我们将为 Claude for Desktop 安装一个预构建的 [Filesystem MCP Server](https://github.com/modelcontextprotocol/servers/tree/main/src/filesystem)。这是由 Anthropic 和社区创建的数十个 [服务器](https://github.com/modelcontextprotocol/servers/tree/main) 之一。

首先，在您的计算机上打开 Claude 菜单并选择 "Settings..."。请注意，这些设置与应用程序窗口中的 Claude 帐户设置不同。

这在 Mac 上的界面如下：
<Frame style={{ textAlign: 'center' }}>
  <img src="/images/quickstart-menu.png" width="400" />
</Frame>

点击设置面板左侧栏中的 "Developer"，然后点击 "Edit Config":
<Frame>
  <img src="/images/quickstart-developer.png" />
</Frame>

如果尚未存在，此时会在以下路径创建一个配置文件:
- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`

文件也会立即在您的文件系统中显示。

使用任意文本编辑器打开配置文件。将文件内容替换为以下内容：
<Tabs>
<Tab title="MacOS/Linux">
```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/Users/username/Desktop",
        "/Users/username/Downloads"
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
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "C:\\Users\\username\\Desktop",
        "C:\\Users\\username\\Downloads"
      ]
    }
  }
}
```
</Tab>
</Tabs>

将 `username` 替换为您计算机的用户名。路径应指向您希望 Claude 能访问和修改的有效目录。默认配置支持 Desktop 和 Downloads，您也可以添加更多路径。

另外，您还需要在计算机上安装 [Node.js](https://nodejs.org) 才能正常运行。要验证是否安装了 Node，请打开计算机的命令行工具。
- 在 macOS 上，从应用程序文件夹打开 Terminal
- 在 Windows 上，按下 "Windows + R"，输入 "cmd"，然后按 Enter

进入命令行后，输入以下命令以验证是否已安装 Node：
```bash
node --version
```
如果出现 "command not found" 或 "node is not recognized" 这样的错误，则需要从[nodejs.org](https://nodejs.org/) 下载并安装 Node。

<Tip>
**配置文件是如何工作的？**

以下内容已翻译为中文：

---

此配置文件告诉 Claude for Desktop 每次启动应用程序时，要启动哪些 MCP 服务器。在本例中，我们添加了一个名为“filesystem”的服务器，该服务器将使用 Node 的 `npx` 命令来安装并运行 `@modelcontextprotocol/server-filesystem`。有关此服务器的详细描述，请参阅[这里](https://github.com/modelcontextprotocol/servers/tree/main/src/filesystem)。该服务器允许您在 Claude for Desktop 中访问文件系统。
</Tip>

<Warning>
**命令权限**

Claude for Desktop 将以您用户帐户的权限运行配置文件中的命令，并访问您的本地文件。仅在您理解并信任命令来源时，才将其添加到配置文件中。
</Warning>

## 3. 重启 Claude

更新您的配置文件后，您需要重新启动 Claude for Desktop。

重新启动后，您应该会在输入框右下角看到一个锤子图标：  
<img src="/images/claude-desktop-mcp-hammer-icon.svg" style={{display: 'inline', margin: 0, height: '1.3em'}} />

<Frame>
  <img src="/images/quickstart-hammer.png" />
</Frame>

单击锤子图标后，您应该会看到 Filesystem MCP Server 提供的工具：  

<Frame style={{ textAlign: 'center' }}>
  <img src="/images/quickstart-tools.png" width="400" />
</Frame>

如果您的服务器未被 Claude for Desktop 检测到，请转到 [故障排除](#troubleshooting) 部分以获取调试建议。

## 4. 尝试使用！

现在，您可以与 Claude 对话，并询问有关您的文件系统的内容。它会知道在何时调用相关工具。

以下是一些您可以尝试对 Claude 提出的请求：  
- 你能写一首诗并将它保存到我的桌面上吗？  
- 下载文件夹里有哪些和工作相关的文件？  
- 你能把桌面的所有图片移动到一个名为“Images”的新文件夹吗？  

根据需求，Claude 会调用相关工具并在执行操作之前寻求您的批准：  

<Frame style={{ textAlign: 'center' }}>
  <img src="/images/quickstart-approve.png" width="500" />
</Frame>  

## 故障排除

<AccordionGroup>
<Accordion title="服务器未出现在 Claude / 没有锤子图标">
  1. 完全重启 Claude for Desktop  
  2. 检查 `claude_desktop_config.json` 文件的语法是否正确  
  3. 确保 `claude_desktop_config.json` 文件中包含的文件路径有效，且是绝对路径而非相对路径  
  4. 查看 [日志](#getting-logs-from-claude-for-desktop) 以了解服务器为何未连接  
  5. 在命令行中手动运行服务器（将 `username` 替换为您在 `claude_desktop_config.json` 中的设置）以查看是否有任何错误：  

<Tabs>
<Tab title="MacOS/Linux">
```bash
npx -y @modelcontextprotocol/server-filesystem /Users/username/Desktop /Users/username/Downloads
```
</Tab>
<Tab title="Windows">
```bash
npx -y @modelcontextprotocol/server-filesystem C:\Users\username\Desktop C:\Users\username\Downloads
```
</Tab>
</Tabs>
</Accordion>

<Accordion title="从 Claude for Desktop 获取日志">
  与 MCP 相关的 Claude.app 日志写入到以下位置的日志文件：  
  - macOS: `~/Library/Logs/Claude`  
  - Windows: `%APPDATA%\Claude\logs`  

  - `mcp.log` 将包含有关 MCP 连接和连接失败的常规日志。  
  - 名为 `mcp-server-SERVERNAME.log` 的文件将包含来自指定服务器的错误（stderr）日志。  

  您可以运行以下命令列出最近的日志并监控新日志（在 Windows 上，它仅显示最近的日志）：  

<Tabs>
<Tab title="MacOS/Linux">
```bash
# 检查 Claude 的日志中是否存在错误
tail -n 20 -f ~/Library/Logs/Claude/mcp*.log
```
</Tab>
<Tab title="Windows">
```bash
type "%APPDATA%\Claude\logs\mcp*.log"
```
</Tab>
</Tabs>
</Accordion>

<Accordion title="工具调用静默失败">
  如果 Claude 尝试使用工具但失败：  


1. 检查 Claude 的日志是否存在错误  
2. 确保您的服务器构建并运行时没有错误  
3. 尝试重启 Claude for Desktop  
</Accordion>  
<Accordion title="这些方法都不起作用时，我该怎么办？">  
请参阅我们的[调试指南](/docs/tools/debugging)，以获取更好的调试工具和更详细的指导。  
</Accordion>  
<Accordion title="Windows 上的 ENOENT 错误和路径中出现 `${APPDATA}` 的情况">  
如果配置的服务器无法加载，并且您在日志中看到的错误提及路径中包含 `${APPDATA}`，您可能需要将 `%APPDATA%` 的展开值添加到 `claude_desktop_config.json` 文件中的 `env` 键下：  

```json  
{
  "brave-search": {
    "command": "npx",
    "args": ["-y", "@modelcontextprotocol/server-brave-search"],
    "env": {
      "APPDATA": "C:\\Users\\user\\AppData\\Roaming\\",
      "BRAVE_API_KEY": "..."
    }
  }
}
```

完成此更改后，请再次启动 Claude Desktop。  

<Warning>  
**需要全局安装 NPM**  

如果您没有全局安装 NPM，`npx` 命令可能仍会失败。如果已全局安装 NPM，您将会在系统中找到 `%APPDATA%\npm` 路径。如果尚未安装，您可以通过以下命令全局安装 NPM：  

```bash  
npm install -g npm  
```  
</Warning>  

</Accordion>  
</AccordionGroup>  

## 下一步  
<CardGroup cols={2}>  
  <Card  
    title="探索其他服务器"  
    icon="grid"  
    href="/examples"  
  >  
    查看我们官方 MCP 服务器和实现的示例库  
  </Card>  
  <Card  
    title="构建您自己的服务器"  
    icon="code"  
    href="/quickstart/server"  
  >  
    现在开始构建您自定义的服务器，以用于 Claude for Desktop 和其他客户端  
  </Card>  
</CardGroup>  