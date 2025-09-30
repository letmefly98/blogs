# 如何使用 Chrome Devtool MCP 

## MCP 简介

开始用 `AI` 帮写代码玩 `Vibe Coding` 的朋友，必然是要知道 `MCP` 服务的。 

其中 `MCP`（即 `Model Context Protocol`）实则是一种协议，用于给 `AI Agent` 提供规范化的通信，让其能了解某些特定的知识、或进行特定服务的操作。比如：  

* 连通 `nuxt-mcp` 服务可给 `AI Agent` 提供最新的文档知识
* 连通 `tapd MCP` 服务可让 `AI Agent` 去读取某个需求的内容、或直接流转状态等
* 腾讯收录的 `MCP` 服务列表：https://cloud.tencent.com/developer/mcp
* `GitHub` 收录的 `MCP` 服务列表：https://github.com/mcp
* 百度广告位推荐的 `MCP` 服务列表：https://www.mcpworld.com/mcp

各式服务都有，比如有的 `AI` 不支持读 `PDF` 文件那调一个会读 `PDF` 文件的 `MCP` 服务即可，等等。  
这样大大地扩展了 `AI` 的功能，也极大地方便了 `AI` 工作流的搭建。

## 开发自定义 MCP 服务

`MCP` 的开发不是本文重点，仅罗列相关资料：

* `MCP` 服务开发导览：https://mcp-docs.cn/introduction
* `ts` 版 `MCP` 服务的依赖库：https://github.com/modelcontextprotocol/typescript-sdk

## 安装 MCP 服务

各家 `IDE` 交互略有不同，但基本都可以手动配置、`CLI` 命令配置、对话框聊天配置。

这里以 [CodeBuddy IDE](https://copilot.tencent.com/ide/) 的手动配置为例。

<img src="/assets/img/chrome-devtool-mcp-setting-in-codebuddy.png" style="max-width:400px" alt="在 CodeBuddy IDE 中配置 Chrome Devtool MCP" />

大多手动配置都是如上类似的方式，在相关 `json` 文件的 `mcpServers` 字段中添加。  
其中 `command` 字段值可能为 `npx` 或 `uvx` 等，需要你拥有相应的 `nodejs` 或 `python` 环境。  
当然也有 `type` 为 `streamableHttp` 不用环境直接连 `url` 的。  
具体以相应 `MCP` 服务的文档为准。

## Chrome Devtool MCP 简介

对于调试网页，`Chrome Devtool` 是不可或缺的工具，它的 `MCP` 服务的重要性也不言而喻了。  

按其文档 https://github.com/ChromeDevTools/chrome-devtools-mcp 所写，会支持的内容有：

* 操作交互，比如点击、填写内容等
* 网页操作，比如新建/关闭切换标签页、跳页等
* 模拟环境，比如模拟 `CPU` 个数、模拟网络、模拟浏览器尺寸等
* 性能分析，比如开启性能录制、导出性能分析等
* 网络，比如列出所有请求等
* 调试，比如插入代码、查阅打印、截屏、记录操作等

能想到的应用场景也不少，粗浅罗列下也有：

* 直接让 `AI` 去读取 `Console` 面板上的报错去分析修复
* 用自然语言去进行交互或爬虫
* 让 `AI` 自行理解性能卡点，并给出优化建议
* ...

## 安装 Chrome Devtool MCP

在 `IDE` 中先加上 `chrome-devtools` 的配置。

```json
{
  "mcpServers": {
    "chrome-devtools": {
      "timeout": 60,
      "type": "stdio",
      "command": "npx",
      "args": [
        "chrome-devtools-mcp@latest",
        "-u=http://127.0.0.1:9222"
      ]
    }
  }
}
```

安装是能成功的，但应该会连不上 `Chrome` 的 `LocalServer`，还需要满足以下条件：

* `NodeJS` 版本大于 20
* `Chrome` 浏览器升级到最新
* 开放 `Chrome` 浏览器调试端口
* 以 `debug` 模式启动 `Chrome` 浏览器

#### 开放 Chrome 浏览器调试端口

参考文档：https://developer.chrome.com/docs/devtools/remote-debugging/local-server?hl=zh-cn

在 `Chrome` 浏览器中访问 `chrome://inspect/#devices`，其中 `Discover network targets` 的 `Configure...` 需配有 `9222` 端口。

<img src="/assets/img/chrome-device-port-setting.png" style="max-width:400px" alt="在 CodeBuddy IDE 中配置 Chrome Devtool MCP" />

一般默认会配有 `9222` 和 `9229`，检查一下总没错。

#### 以 debug 模式启动 Chrome 浏览器

在命令行调用以下脚本，将打开浏览器，访问  `http://localhost:9222/json` 有数据则进入了 `debug` 模式。

```shell
# win
"C:\Program Files\Google\Chrome\Application\chrome.exe" --remote-debugging-port=9222  --user-data-dir=/tmp/chrome-debug

# mac
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome --remote-debugging-port=9222 --user-data-dir=/tmp/chrome-debug
```

如果你觉得每次输入这样的命令麻烦，创建快捷方式的方案还挺多的：

* 置于系统变量中，比如 mac 在 `~/.zshrc` 中加 `alias xxx=[command]`，命令行中用 `xxx` 即可
* 创建 `.sh` 或 `.bat` 脚本置于易用的地方
* 用 `win` 的 [BatToExtConverter](https://www.majorgeeks.com/files/details/bat_to_exe_converter.html) 或 `mac` 系统自带的 `自动操作` 也能生成启动器

## 使用 Chrome Devtool MCP

启动 `debug` 模式的 `Chrome` 浏览器后，在对话框中输入，  
诸如 “获取当前网页的网址” 或 “在输入框中输入xxx并回车” 多半能如愿用 `AI` 操作浏览器.了
