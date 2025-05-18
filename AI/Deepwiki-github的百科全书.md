<<<<<<< HEAD
# DeepWiki:  Github的百科全书
=======
# Deepwiki-github的百科全书
>>>>>>> 9b3512f2a905a8e159992324edb4cfebd3829e6e

官网：[DeepWiki](https://link.zhihu.com/?target=https%3A//deepwiki.com/)

![image.png](https://s2.loli.net/2025/05/11/JUH6Su98pbIYQrT.png)

## **一、介绍**

DeepWiki是由[Cognition AI](https://zhida.zhihu.com/search?content_id=257084220&content_type=Article&match_order=1&q=Cognition+AI&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NDcxMTQ3NjEsInEiOiJDb2duaXRpb24gQUkiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNTcwODQyMjAsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.BMoyyrCB2aSyJaSnx68RyJ3COlYtMEhPnGou4p8eC7I&zhida_source=entity)（Cognition Labs）基于其明星产品[Devin](https://zhida.zhihu.com/search?content_id=257084220&content_type=Article&match_order=1&q=Devin&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NDcxMTQ3NjEsInEiOiJEZXZpbiIsInpoaWRhX3NvdXJjZSI6ImVudGl0eSIsImNvbnRlbnRfaWQiOjI1NzA4NDIyMCwiY29udGVudF90eXBlIjoiQXJ0aWNsZSIsIm1hdGNoX29yZGVyIjoxLCJ6ZF90b2tlbiI6bnVsbH0.UOavXuesF8dYktEsAoo53t5GV0XdxMLtHkZF0l42xGU&zhida_source=entity)（全球首个AI软件工程师）开发的一款开源工具，它结合了最前沿的人工智能技术，旨在帮助开发者更高效地阅读、理解和分析 GitHub 上的源码，从而加速开发进程，提升代码质量（无需注册即可使用）。自2025年4月27日发布以来，DeepWiki迅速成为开发者社区的热门工具，被誉为“GitHub的维基百科”。

### 亮点功能

1. **自动化文档生成**
   DeepWiki通过分析代码、README文件及配置文件，自动生成结构化技术文档，涵盖项目目标、核心模块、依赖关系图等。其生成的文档比传统README更详细，甚至能为缺乏文档的项目提供清晰说明。
2. **对话式AI助手**
3. **支持公共与私有仓库**，用户还可手动请求索引未收录的公共仓库。
4. **提供直观的代码可视化功能，如类图、函数调用关系图等，帮助开发者以图形化的方式理解代码结构。**
5. **内置多种代码分析工具，如代码质量评估、潜在缺陷预测等，帮助开发者发现代码中的潜在问题。**

## **二、技术背景与实现**

此部分的分析图例，基于使用deepwiki对deepwiki自己代码分析的结果：[https://deepwiki.com/deepskies/DeepWiki](https://link.zhihu.com/?target=https%3A//deepwiki.com/deepskies/DeepWiki)

**底层技术**：整合了Devin的AI能力，结合[大规模语言模型](https://zhida.zhihu.com/search?content_id=257084220&content_type=Article&match_order=1&q=大规模语言模型&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NDcxMTQ3NjEsInEiOiLlpKfop4TmqKHor63oqIDmqKHlnosiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNTcwODQyMjAsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.f434Jq7TlaoXwgHXMEgYwKfXFXhFfDFK3h2e9l1sQ5Y&zhida_source=entity)（LLM）、[多模态代码理解模型](https://zhida.zhihu.com/search?content_id=257084220&content_type=Article&match_order=1&q=多模态代码理解模型&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NDcxMTQ3NjEsInEiOiLlpJrmqKHmgIHku6PnoIHnkIbop6PmqKHlnosiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNTcwODQyMjAsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.DkzgaXgwtqtatWR_MPqUuA9g4ejgbqbQcuymgrFwKIo&zhida_source=entity)及云计算基础设施，实现对代码库的全局结构分析。模型将代码库分解为层级系统，逐层生成文档，解决了传统工具难以理解全局逻辑的难题。

**数据规模**：已索引3万个仓库，处理超40亿行代码及1000亿个Token，耗资30万美元计算资源。单个仓库索引平均成本约12美元。

## **三、使用方法**

### 3.1 **URL替换法**

将GitHub链接中的`github.com`替换为`deepwiki.com`，例如：
`https://github.com/xxx/xxx` → `https://deepwiki.com/xxx/xxx`。

### 3.2 **官网搜索**

访问[DeepWiki官网](https://link.zhihu.com/?target=https%3A//deepwiki.com/)，通过搜索框输入仓库名称或浏览热门项目列表。

### 3.3 **第三方工具**

开发者社区提供了[Tampermonkey](https://zhida.zhihu.com/search?content_id=257084220&content_type=Article&match_order=1&q=Tampermonkey&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NDcxMTQ3NjEsInEiOiJUYW1wZXJtb25rZXkiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNTcwODQyMjAsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.2aRRBJY8wbdP1ALUC-Znukbv4muuOn0YJSq7upeTDSA&zhida_source=entity)脚本，可在GitHub页面添加“Go DeepWiki”按钮，一键跳转至对应文档页。

## **四、应用场景**

**开发者学习与协作**：帮助新人快速理解复杂项目结构，减少学习成本；支持团队通过共享文档提升协作效率。

**开源社区促进**：弥补文档缺失的痛点，降低参与开源项目的门槛。用户反馈显示，部分项目生成的文档甚至优于官方版本。

**教育与技术面试**：学生可通过DeepWiki学习优秀项目，求职者可快速掌握目标公司的技术栈。

## **五、示例**

以github上面的DeepWiki-open为例https://deepwiki.com/AsyncFuncAI/deepwiki-open，可以看到Deepwiki目前的大概展示效果

![image.png](https://s2.loli.net/2025/05/11/IVABpuUtvHs7gbX.png)

## 六、总结

DeepWiki通过AI技术重新定义了代码文档的生成与交互方式，其开源免费模式为开发者社区带来革新。尽管存在准确性和成本挑战，但其在提升代码可读性、促进开源协作方面的潜力已得到广泛认可。

### 