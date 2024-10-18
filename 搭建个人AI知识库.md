---
title: 搭建个人AI知识库：RAG与本地模型实践指南
date: 2024-10-18
tags: [AI, 大语言模型, 知识库]
categories: [AI，软件，工具]
---

# 引言

你是否想过拥有一个私人订制的AI助手，能够随时为你提供最个性化的信息？本文将带你一步步搭建一个基于本地模型和RAG技术的个人知识库。

# 搭建本地模型
## 环境
- os: archlinux
- 内存: 32g
- cpu: 6核12线程
- python: 3.12.7
- docker27.3.1 + docker-compose
- 向量库: milvus2.4.13 + attu2.4(客户端)
## ollama
``` shell
pacman -S ollama

systemctl start ollama.service

# 通过下述url判断ollama是否安装成功
http://127.0.0.1:11434/

```
## llama3.2:3b
``` shell
ollama run llama3.2:3b
```
## OpenWebUI(非必须)
``` shell
# 启动openwebui, 按照自己需要调整端口
docker run -d -p 3000:8080 --add-host=host.docker.internal:host-gateway -v open-webui:/app/backend/data --name open-webui ghcr.io/open-webui/open-webui:main

# 浏览器访问, 可以看到之前启动的模型
http://localhost:3000/
```
## 程序访问测试
``` python
from langchain_ollama import ChatOllama
from langchain_core.messages import HumanMessage

# 创建ChatOllama实例，指定模型名称
model = ChatOllama(model="llama3.2:3b")

# 定义你的问题
question = HumanMessage("你是如何工作的？")

# 使用模型处理问题
response = model.invoke([question])

# 打印返回的结果
print(response.content)
```
# 构建知识库
## RAG是什么
大模型的训练数据是有截止日期的，那当我们需要依靠不包含在大模型训练集中的数据时，我们该怎么做呢？一种就是对模型进行微调，另外就是通过检索增强生成RAG（Retrieval Augmented Generation）。在这个过程中，首先检索外部数据，然后在生成步骤中将这些数据传递给LLM。

利用大模型的能力搭建知识库就是一个RAG技术的应用。

### RAG的应用抽象为5个过程：
- 文档加载（Document Loading） ：从多种不同来源加载文档。LangChain提供了100多种不同的文档加载器，包括PDF在内的非结构化的数据、SQL在内的结构化的数据，以及Python、Java之类的代码等
- 文本分割（Splitting） ：文本分割器把Documents 切分为指定大小的块，我把它们称为“文档块”或者“文档片”
- 存储（Storage）： 存储涉及到两个环节，分别是：
    1. 将切分好的文档块进行嵌入（Embedding）转换成向量的形式
    2. 将Embedding后的向量数据存储到向量数据库
- 检索（Retrieval） ：一旦数据进入向量数据库，我们仍然需要将数据检索出来，我们会通过某种检索算法找到与输入问题相似的嵌入片
- 输出（Output） ：把问题以及检索出来的嵌入片一起提交给LLM，LLM会通过问题和检索出来的提示一起来生成更加合理的答案
## 个人笔记
首先起码得有自己的知识库，我这里就是个人多年整理的笔记。或者你有项目相关的文档，也可以作为知识库的基础。
## 将个人笔记写入到Milvus
``` python
from langchain_community.document_loaders import DirectoryLoader
from langchain_community.vectorstores import Milvus
from langchain_ollama import OllamaEmbeddings
from langchain_text_splitters import RecursiveCharacterTextSplitter
from pymilvus import MilvusClient
import os

# 检查collection是否存在, 如果不指定，默认为LangChainCollection
collection_name = "note"

# 设置 Milvus 客户端
client = MilvusClient(uri="http://localhost:19530")

# 检查collection是否存在
if client.has_collection(collection_name):
    # collection存在，执行后续操作
    print(f"Collection '{collection_name}' exists.")
else:
    # collection不存在，创建collection并进行向量化
    print(f"Collection '{collection_name}' does not exist. Creating now...")
    # 从url导入知识作为聊天背景上下文, glob代表只查找org文件，可根据实际情况调整为txt等，recursive=True表示会递归查找
    loader = DirectoryLoader(os.path.join(os.environ["HOME"], "Documents/notes"), glob="*.org", recursive=True)
    # 加载一堆文件
    docs = loader.load()

    # 文本分词器
    # chunk_size=1000
    # 表示拆分的文档的大小，也就是上面所说的要设置为多少合适取决于所使用LLM 的窗口大小
    # chunk_overlap=100
    # 这个参数表示每个拆分好的文档重复多少个字符串。
    # 不过这种递归的方式更只能点，不设参数试试默认
    text_splitter = RecursiveCharacterTextSplitter()
    documents = text_splitter.split_documents(docs)

    # ollama嵌入层
    embeddings = OllamaEmbeddings(
        model="llama3.2:3b"
    )

    # 文档向量化，会持久化
    vector_store = Milvus.from_documents(documents=documents, embedding=embeddings, collection_name=collection_name, drop_old=True)
    print(f"collection'{collection_name}'创建成功!")

```
注: 上述加载文件的目录需要根据自己实际情况调整，其它的最好用默认，减少出错概率
## 将llm与Milvus结合
```python
from langchain.chains import create_retrieval_chain
from langchain.chains.combine_documents import create_stuff_documents_chain
from langchain_community.vectorstores import Milvus
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_ollama import OllamaEmbeddings
from langchain_ollama import OllamaLLM
from langchain_text_splitters import RecursiveCharacterTextSplitter
from pymilvus import MilvusClient


def exec(question):
    # 检查collection是否存在
    collection_name = "note"
    # 设置 Milvus 客户端
    client = MilvusClient(uri="http://localhost:19530")
    vector_store = None
    # 检查collection是否存在
    if client.has_collection(collection_name):
        # collection存在，执行后续操作
        print(f"Collection '{collection_name}' exists.")
        # 文本分词器
        text_splitter = RecursiveCharacterTextSplitter()
        documents = text_splitter.split_documents([])

        # ollama嵌入层
        embeddings = OllamaEmbeddings(
            model="llama3.2:3b"
        )
        # 文档向量化
        vector_store = Milvus.from_documents(documents=documents, embedding=embeddings, collection_name=collection_name)
    else:
        # collection不存在，创建collection并进行向量化
        print(f"Collection '{collection_name}' does not exist. Please exec LoadFile2Vector.py first")
    if vector_store is not None:
        # 创建ollama 模型 llama2
        llm = OllamaLLM(model="llama3.2:3b")
        output_parser = StrOutputParser()

        # 创建提示词模版
        prompt = ChatPromptTemplate.from_template(
            """Answer the following question based only on the provided context:
            <context>
            {context}
            </context>
            Question: {input}"""
        )

        # 生成chain ：   prompt | llm
        document_chain = create_stuff_documents_chain(llm, prompt)

        # 向量数据库检索器
        retriever = vector_store.as_retriever()

        # 向量数据库检索chain :  vector | prompt | llm
        retrieval_chain = create_retrieval_chain(retriever, document_chain)

        # 调用上面的 (向量数据库检索chain)
        response = retrieval_chain.invoke({"input": question})
        # 打印结果
        print(response["answer"])


if __name__ == '__main__':
    exec("我有什么梦想? 如何实现")
```
大致的流程是：用户的query先转成embedding，去向量数据库查询最接近的top K回答；然后这query + top K的回答 + 其他context一起进入LLM，让LLM整合上述所有的信息后给出最终的回复。
## 提供接口(非必须)
可通过fastapi等提供restful接口供外部调用，比如一些个人项目公司内部项目之类的，瞬间高大上起来了。

# 项目源码
https://github.com/zhaozhiwei1992/NoteAI.git

# 参考
https://blog.csdn.net/AAI666666/article/details/137509781

https://ollama.com/library

文本向量转换: https://github.com/shibing624/text2vec

文本存储: https://juejin.cn/post/7360564568660410377
