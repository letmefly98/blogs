# AI Spec 工作流初体验

## AI 工作流的演进历程

起初 `GPT-3` 时代，输入的文本（`Prompt`）是个对话框，还会限制输入字符的长度，用 `token` 来计费。当对话次数多了后，大概消耗 2k-4k `tokens` 后模型就表现出遗忘前文的表现。  
虽然有“滑动窗口”这种妥协方案，把初始提示或最近的对话再传一遍，但长流程的连贯性依旧不尽人意。  

随着模型架构（如 `Transformer` 注意力机制的改进）和硬件能力的提升，上下文窗口已经能支持甚至 1000k 量级，可大概理解为输入 50w+ 的汉字都没问题，极大地减少了截断问题。  

而这又产生了新的问题，要理解的内容变多了，在回答错几次后可能就开始离答案越来越远，即​​“中间丢失”现象。  
像人类思考一样，总结之前的对话形成结论再带着结论进行下一个对话，算是能缓解部分这类纠偏而产生的误差。  

再后来有了“检索增强生成”技术，即让模型从被动接受输入的信息，转为主动判断需要的知识。比如你问模型组件的报错怎么改，模型会先去读组件的文档，再结合问题进行输出。  
其实对应的既是能读写的智能体（`Agent`）。而读哪些组件文档起初是靠训练，现在则是搭建以 `MCP` 为协议的服务。  

渐渐的，从最初 输入->模型生成->结束 的模式，模型开始有各种诸如总结、检索、等待回复等等中间过程，也就有了 AI 工作流。  

而推测工作流（`Speculative Workflow`，简写为 `Spec`）是一种将获得结果拆成多个步骤的运行方式。即先出计划书，确认完计划无误后，模型再按计划书分批按序执行。

## Spec 工作流的优缺点

### 优点

* 相比普通的 `Vibe Coding` 写到哪算哪的即兴发挥，工作流有预先规划相对可控。
* 对模糊或复杂的需求，能拆小拆细，一步步地去确认，而不是一口气做完。
* 工作流中生成的计划书可以比较方便的留档和查阅，而不是把一堆还带问题修复过程的对话导出才能留档。

### 缺点

* 不适合小型需求。比如改个弹窗、优化下函数还得计划一下有点不必了，直接交给 `IDE` 的 `agent` 然后检查下就行。
* 不适合验收结果不确定的需求。比如生成个网站，要列的计划会非常的多，确认时也不知道结果，不如先用别的工具生成完再对某些点按计划改。

## 准备工作

以下操作以 [CodeBuddy IDE](https://copilot.tencent.com/ide/) 为例。  
当然你也可以用 [spec-kit](https://github.com/github/spec-kit) 之类的工具，或者 `IDE` 后续会发布的自定义斜杠命令，内核是一致的。

要让模型采用我们的 `Spec` 工作流，你需要先为其准备规则（`Rules`）。  
我一般采用 `重述需求->技术方案->具体实现清单` 即 `require -> design -> tasks` 的三段式。 

以 “将项目中所有 `xxModel.ts` 文件名称改为 `xx.model.ts` 并修正文件引用” 为例，  
* 重述需求，会继续明确需求，比如已经是 `xx.model.ts` 的文件怎么处理，是否所有对该文件的引用都修改，等等。  
* 技术方案，会列出比如用 `shell` 命令还是别的，以及按先查找、再修改、再打包验证的步骤，等等。  
* 具体实现，我基本就不看了。随着 ide 的执行，可能才发现用打包来验证文件引用会太慢和往复，就该中断执行继续对话调整下技术方案，再确认执行。

### CodeBuddy 的自定义斜杠命令

为了方便验证，我们先学会使用 `IDE` 的斜杠命令。  

比如，我有一个 `.codebuddy/.rules/hello.mdc` 文件，内容如下：  

```markdown
---
description: Command /hello - 简单的问候命令，输出 "hello" 加上用户提供的参数
globs:
  alwaysApply: false
---

# 命令：/hello（问候命令）

## 目标
- 接收用户输入的参数，输出格式化的问候语
- 演示自定义命令的基本用法

## 输入
- 用户调用格式：/hello [参数]
- 参数可以是任意文本内容

## 执行流程与能力
1) 参数解析
- 提取 /hello 命令后面的所有文本作为 $1 参数
- 如果没有提供参数，则使用默认值 "World"

2) 输出格式化
- 输出格式：hello [参数内容]
- 保持简洁明了的输出风格

## 产出与落盘
- 直接输出问候语，无需创建文件

## 使用示例
- `/hello World` → 输出：hello World
- `/hello 张三` → 输出：hello 张三
- `/hello CodeBuddy` → 输出：hello CodeBuddy
- `/hello` → 输出：hello World（默认值）
```

基本满足以上文档结构即可：
* 命令：/{命令名称}（命令简述）
* 目标
* 前置
* 输入
* 执行流程与能力
* 产出与落盘
* 使用示例

当然你也可以直接在对话框中输入“创建 /hello 自定义命令，调用该命令时模型输出 “hello $1”，其中 $1 为调用命令时跟在后面的文本” 回车。  
让你的 IDE 按它的规范自己创一个，可能结构与我的会有不同，以你的为准。

保存 `hello.mdc` 后，在对话框中去验证，以此你学会了快速调用提示词。

<img src="/assets/img/ai-spec-hello-case-preview.png" style="max-width:400px" alt="IDE 自定义斜杠命令基础演示" />

而实现工作流，可以认为就是多个自定义命令按需调用来的。  
那么新建 `require.mdc`、`design.mdc`、`tasks.mdc` 来实现我们的 `Spec` 工作流吧。

> talk is cheap, show me the prompt.

### require.mdc

在 **重述需求** 流程中，就和平常需求会一样，着重需要调整的是：

* 识别歧义、明确边界、补充场景、等待澄清问题或确认需求、规划优先级
* 按更规范化的 EARS 语法撰写需求，落盘为 PRD 需求文档

可查看 <a :href="`${baseUrl}/assets/md/require.mdc`" title="笔者在用的design.mdc">require.mdc</a>，是我个人在用的版本。

以 “找出 README.md 中没有列出的文章” 这个需求为例，在 PRD 需求文档 `.specs/missing-articles-analysis/requirements.md` 中将生成如下内容：

<img src="/assets/img/ai-spec-require-case-preview.png" style="max-width:400px" alt="require.mdc 生成的需求文档节选" />

对歧义与待确认清单的内容，用对话框继续对话解答，让其变为已确认，且文档其他内容也无异议后。即可进入下个流程。

### design.mdc

在 **技术方案设计** 流程中，就和技术沟通会一样，着重需要调整的是：

* 实现方案选型
* 模块拆分
* 运行流程顺序
* 技术难点与对策
* 安全合规、风险识别、应急预案
* 测试方案、测试数据准备、验收方案
* 统计报告
* 落盘为方案设计文档

可查看 <a :href="`${baseUrl}/assets/md/design.mdc`" title="笔者在用的design.mdc">design.mdc</a>，是我个人在用的版本。生成内容大致如下：

<img src="/assets/img/ai-spec-design-case-preview.png" style="max-width:400px" alt="design.mdc 生成的技术方案节选" />

同理，对不如意的地方用对话框继续对话。方案觉得可行后，即可进入下个流程。

### tasks.md

在 **具体实现** 流程中，就和真正开发时一样，着重需要调整的是：

* 将工作拆解为 10-15 分钟的最小可执行任务单元
* 每完成一个单元，更新下 todo-list 状态
* 执行过程中遇上设计偏离、需求确实、性能卡点，按情况中断或登记在册
* 开发遵从已有代码、开发规范
* 严格遵循质量门禁

可查看 <a :href="`${baseUrl}/assets/md/tasks.mdc`" title="笔者在用的tasks.mdc">tasks.mdc</a>，是我个人在用的版本。过程中的 todo-list 内容大致如下：

<img src="/assets/img/ai-spec-tasks-case-preview.png" style="max-width:400px" alt="design.mdc 生成的技术方案节选" />

### spec.mdc

以上三个自定义命令完成后，可再新建一个 `spec.mdc` 去组合三个命令，简化一点交互。可查看 <a :href="`${baseUrl}/assets/md/spec.mdc`" title="笔者在用的spec.mdc">spec.mdc</a>。

至此，你只需

* 在对话框中输入 “/spec {需求}” 即可。
* 等待需求文档生成后检查，回答其中的待确认清单，然后输入确认。
* 等待技术方案设计文档生成后检查，沟通确认技术方案，然后输入确认。
* 等待 IDE 自动运行，若技术方案无误多半能得到满意结果，但若耗时超长应中断对话去沟通调整技术方案。

## 总结

* 介绍了 `AI Spec` 工作流的历程和优缺点
* 学会使用 IDE 的自定义斜杠命令
* 撰写或粘贴 `require.mdc`、`design.mdc`、`tasks.mdc` 且运行结果满足预期
* 撰写或粘贴 `spec.mdc` 完成闭环，可在项目中实战了

## 其他

* 腾讯云所用 Rules：https://github.com/TencentCloudBase/CloudBase-AI-ToolKit/blob/main/config/.cursor/rules/cloudbase-rules.mdc
* Kiro 所用 Rules：https://gist.github.com/CypherpunkSamurai/ad7be9c3ea07cf4fe55053323012ab4d
