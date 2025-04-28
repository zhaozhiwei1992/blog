---
title: "RooCode结合本地MCP实践"
date: "2025-04-28"
updated: "2025-04-28"
tags: [AI,MCP]
categories: AI
---

# 引言

目前MCP也炒了一段时间，是骡子是马得拉出来溜溜。这篇文章偏向点技术，如果没有一点开发经验听个响就好。

# 实现两个简单功能

## 计算求和

## 显示本地目录列表

# 操作步骤

## 通过python实现一个server

### 文件创建

``` sh
cd ~/workspace/python/demo
mkdir mcp && cd mcp
touch SumMCP.py
```

### 代码

``` python
"""
一个MCP Server, SumMCP.py
"""
from fastmcp import FastMCP
import os

mcp = FastMCP("Demo", log_level="ERROR")

@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two numbers"""
    print("add", a, b)
    return a + b

@mcp.tool()
def listdir(path: str) -> list[str]:
    """show user dir list 显示用户文件列表"""
    return os.listdir(path)

if __name__ == "__main__":
    # mcp.run()
    # mcp.run(transport="sse", port=9123)
    # 基于标准输入输出实现，本地小工具推荐这种方式
    mcp.run(transport='stdio')
```

### 依赖安装

我这里使用的是anaconda，其它依赖管理都行

``` sh
conda create -n py_3.12 python=3.12
source /opt/anaconda/bin/activate py_3.12
pip install fastmcp
```

### 参考

<https://github.com/jlowin/fastmcp>

## vscode安装RooCode插件

## 在RooCode插件中配置MCP

### 配置

在mcp_setting.json文件中配置自己的server

``` example
{
  "mcpServers": {
    "demo": {
      "command": "~/.conda/envs/py_3.12/bin/fastmcp",
      "args": ["run", "~/workspace/python/demo/mcp/SumMCP.py"],
      "disabled": false,
      "alwaysAllow": []
    }
  }
}
```

### 检查是否成功

配置完成后，mcpServer会显示绿色图标,如下

![mcp配置](https://43.143.194.245/minio/images/roocode_mcp.png)

如果在之前SumMCP.py中增加了工具，一定要在这里刷新下，显示出新的工具，然后在插件中使用才会读取到。

## 测试

### 求和

使用demo工具, 计算99和98的和, 效果如下

![显示求和](https://43.143.194.245/minio/images/roocode_mcp_sum.png)

### 显示文件列表

使用listdir工具显示 /home/xx/Documents下的文件列表，效果如下

![显示文件列表](https://43.143.194.245/minio/images/roocode_mcp_listdir.png)

# 总结

使用mcp的方式可以按照自己要求实现一些插件本身做不了或者不好实现的功能。目的是为了让外部交互的返回更好的跟模型上下文结合，从而产生更好的效果。如果对话过程中，模型能够自动识别是需要使用哪个mcp那么效果应该会更好。目前是需要明确告诉插件使用mcp，有时候甚至得告诉它具体是哪个tool。

这里只是一个最简单的使用，企业项目中能否使用该方式扩展业务，如何扩展，且听下回分解。
