---
title: OpenManus+DeepSeek体验
date: 2025-03-10
updated: 2025-03-10
tags: [OpenManus, AI, DeepSeek]
---

# 前言
这篇文章偏技术，如果跟着文章操作有难度，建议找网上找相关视频操作。
# 什么是OpenManus，为什么要使用
说到OpenManus就不得不先提下Manus，Manus是全球首款真正意义上的通用型AI Agent，由中国的创业公司Monica开发，看官方的演示感觉很6，但是10w的邀请吗告诉我，我不配。

幸好有大佬连夜整出来一个OpenManus，引用官方一句描述: Manus 非常棒，但 OpenManus 无需邀请码即可实现任何创意 ！

## 那么OpenManus能做什么呢

复杂任务规划与执行：它可以把复杂的任务拆解成多个小步骤，自动规划并执行，最后给出完整的成果。

工具调用与自动化：很多工作离不开外部工具。 OpenManus 能灵活调动 各种工具，比如浏览器、数据分析软件等，完全不用你担心技术细节。

智能信息收集与处理：OpenManus 不仅能浏览网页、提取信息，还能进行精准的内容整理。

# 安装使用
## 环境
### os: archlinux
## 安装OpenManus
```
1. 源码下载
git clone https://github.com/mannaandpoem/OpenManus.git

2. 进入项目根目录
cd OpenManus

3. 创建新的 conda 环境
conda create -n open_manus python=3.12

4. 等待完成后，设置当前python环境
source /opt/anaconda/bin/activate open_manus

5. 安装依赖
pip install -r requirements.txt
```
## 购买DeepSeekApi
```
1. 访问DeepSeek官网 https://www.deepseek.com./ 。

2. 点击右上角API开放平台，左边充值即可，测试充个10元就勾了。

3. 充值完毕后，点击API Keys，创建自己的key，我这里起名字就是OpenManus(可以看看这玩意儿消耗tokens的量)
```

## 配置OpenManus
```
OpenManus 需要配置使用的 LLM API，请按以下步骤设置：

1. 保证当前还在项目跟目录下

2. 创建config.toml文件：
cp config/config.example.toml config/config.toml

3. 编辑 config/config.toml 添加 API 密钥和自定义设置：(注: model和base_url在deepseek充值界面接口文档就可以看到)

# Global LLM configuration
[llm]
model = "deepseek-chat"
base_url = "https://api.deepseek.com"
api_key = "sk-8.....9ff89"
max_tokens = 4096
temperature = 0.0

# [llm] #AZURE OPENAI:
# api_type= 'azure'
# model = "YOUR_MODEL_NAME" #"gpt-4o-mini"
# base_url = "{YOUR_AZURE_ENDPOINT.rstrip('/')}/openai/deployments/{AZURE_DEPOLYMENT_ID}"
# api_key = "AZURE API KEY"
# max_tokens = 8096
# temperature = 0.0
# api_version="AZURE API VERSION" #"2024-08-01-preview"

# Optional configuration for specific LLM models
[llm.vision]
model = "claude-3-5-sonnet"
base_url = "https://api.openai.com/v1"
api_key = "sk-..."

```

## 启动
``` python
python main.py
```

输出如下: 
```
INFO     [browser_use] BrowserUse logging setup complete with level info
INFO     [root] Anonymized telemetry enabled. See https://docs.browser-use.com/development/telemetry for more information.
Enter your prompt (or 'exit'/'quit' to quit): (这里就是发挥创意的地方了)

```

# 一些小问题处理
## 目前内部默认采用的是google搜索，国内不好用，调整为bing搜索

1. 在app/tool/目录，加bing_search.py
``` python
import asyncio
from typing import List
from urllib.parse import quote
import requests
from bs4 import BeautifulSoup
from app.tool.base import BaseTool

class BingSearch(BaseTool):
    name: str = "bing_search"
    description: str = """执行必应搜索并返回相关链接列表。
当需要获取国际信息或英文内容时建议使用此工具。
工具返回与搜索查询匹配的URL列表。"""
    parameters: dict = {
        "type": "object",
        "properties": {
            "query": {
                "type": "string",
                "description": "(必填) 提交给必应的搜索关键词"
            },
            "num_results": {
                "type": "integer",
                "description": "(可选) 返回的搜索结果数量，默认10",
                "default": 10
            }
        },
        "required": ["query"]
    }

    async def execute(self, query: str, num_results: int = 10) -> List[str]:
        """
        执行必应搜索并返回URL列表

        Args:
            query: 搜索关键词
            num_results: 返回结果数量

        Returns:
            匹配搜索结果的URL列表
        """

        def sync_search():
            headers = {
                'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
                'Accept-Language': 'en-US,en;q=0.9'
            }
            url = f'https://www.bing.com/search?q={quote(query)}'
            links = []

            for page in range(0, num_results // 10 + 1):
                resp = requests.get(
                    f'{url}&first={page * 10}',
                    headers=headers,
                    timeout=10
                )
                soup = BeautifulSoup(resp.text, 'html.parser')

                for result in soup.select('.b_algo'):
                    link = result.find('a', href=True)
                    if link and 'href' in link.attrs:
                        links.append(link['href'])
                        if len(links) >= num_results:
                            return links
            rst = links[:num_results]
            return rst

        loop = asyncio.get_event_loop()
        return await loop.run_in_executor(None, sync_search)

```
2. agent/manus.py 中添加 BingSearch
``` python
from pydantic import Field

from app.agent.toolcall import ToolCallAgent
from app.prompt.manus import NEXT_STEP_PROMPT, SYSTEM_PROMPT
from app.tool import Terminate, ToolCollection
from app.tool.browser_use_tool import BrowserUseTool
from app.tool.file_saver import FileSaver
from app.tool.google_search import GoogleSearch
# 一定要先引入
from app.tool.bing_search import BingSearch
from app.tool.python_execute import PythonExecute


class Manus(ToolCallAgent):
    """
    A versatile general-purpose agent that uses planning to solve various tasks.

    This agent extends PlanningAgent with a comprehensive set of tools and capabilities,
    including Python execution, web browsing, file operations, and information retrieval
    to handle a wide range of user requests.
    """

    name: str = "Manus"
    description: str = (
        "A versatile agent that can solve various tasks using multiple tools"
    )

    system_prompt: str = SYSTEM_PROMPT
    next_step_prompt: str = NEXT_STEP_PROMPT

    # Add general-purpose tools to the tool collection
    # 注意这里添加BingSearch
    available_tools: ToolCollection = Field(
        default_factory=lambda: ToolCollection(
            PythonExecute(), GoogleSearch(), BingSearch(), BrowserUseTool(), FileSaver(), Terminate()
        )
    )
```
3. prompt/manus.py 修改GoogleSearch为BingSearch
``` sh
sed -i 's/GoogleSearch/BingSearch/g'
```

参考: https://github.com/mannaandpoem/OpenManus/issues/277
# 测试
1. 查找中国 前十高校，通过csv格式返回到跟目录，只需要有学校名称 、排名、官网即可

实际测试，这玩意儿还是bug不少，跑了十几分钟结果没出结果，白瞎了我的token :D。
# 总结
速度真的是挺慢，但是能够自己主动去搜索解决问题也算是挺牛了，对于技术人员来说又多了一个玩具。

这种主动去思考解决问题的思想比较有意思，在其它平台定义工作流时候可以借鉴这种方案，创建自己的manus，加油!!
