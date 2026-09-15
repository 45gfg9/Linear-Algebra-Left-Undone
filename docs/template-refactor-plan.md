# LALU 模板重构方案

> 状态：设计稿。本文只描述后续实现方案，不改变当前任何 TeX、索引或构建行为。

## 目标与不变量

本次重构的目标不是改变讲义的编排，而是把目前依赖跨章节分组、动态控制序列标记和隐式文件副作用的实现，替换成可检查、可测试的显式状态模型。

必须保持以下用户可见约定：

- 普通正文继续使用 `\chapter`，显示为“第 N 讲”。
- 未竟专题继续使用 `\LUchapter{标题}`；标题编号显示为“未竟专题 X”，其中 `X` 是未竟专题的全局中文序号。
- 未竟专题的数字引用继续由“上一普通正文编号 + 希腊字母”组成。例如第一讲后的前两个未竟专题分别引用为 `1ε`、`1δ`。
- 未竟专题内的节、公式、图表和定理继续沿用该数字引用，例如 `1ε.2`。
- 正文和答案册中的既有标签键、目录层级及主要版式应保持兼容；确需改变的辅助文件格式必须通过干净重编译迁移。
- 所有 Hyperref destination 必须唯一、稳定、只含适合机器处理的 ASCII 内容；显示编号不得直接充当锚点编号。

以下三个概念应始终分离：

| 概念 | 示例 | 用途 |
| --- | --- | --- |
| 标题显示编号 | `未竟专题十四` | 章标题、答案册单元标题、目录章条目 |
| 数字引用编号 | `23ε` | `\ref`、节/公式/定理的父编号 |
| 机器锚点编号 | `chapter.LU.14` | Hyperref destination、外部 PDF 链接 |

## 一、正文、未竟专题与答案册单元

### 1.1 统一的章节记录

正文和答案抽取器共享一个只读的“当前章节记录”。建议由一个内部模块维护下列字段，而不是通过 `\@exist@...` 控制序列是否存在来反推状态：

| 字段 | 含义 | 示例 |
| --- | --- | --- |
| `phase` | `frontmatter`、`mainmatter`、`appendix` 或 `backmatter` | `mainmatter` |
| `kind` | `normal`、`unfinished`、`appendix` 或 `unnumbered` | `unfinished` |
| `structural-chapter` | 标准 `c@chapter` 的当前结构流水号 | `37` |
| `normal-display` | 普通正文的显示编号 | `23` |
| `normal-base` | 最近一个普通正文的阿拉伯数字编号 | `23` |
| `unfinished-serial` | 未竟专题的全局流水号 | `14` |
| `unfinished-slot` | 同一普通正文之后的未竟专题次序 | `1` |
| `unnumbered-serial` | 实际无编号顶层标题的内部流水号，仅用于必要的唯一锚点 | `3` |
| `title` | 完整标题，保持未展开 token | `范畴论视角下的线性代数` |
| `short-title` | 页眉、目录所用短标题；缺省时等于完整标题 | 同上 |
| `unit-id` | 稳定的逻辑单元标识 | `lu:14` |

实现上可使用 expl3 的全局 token list、integer 和 boolean；对外只暴露查询命令，不允许正文直接改内部变量。至少提供以下语义接口：

- 进入普通正文单元；
- 进入未竟专题单元；
- 进入无编号章节边界；
- 取得当前记录的不可变快照，供答案抽取器使用；
- 判断当前是否存在可承载习题的编号单元。

状态转换必须幂等。“离开未竟专题”调用两次不应产生额外 `\endgroup`、计数器变化或格式变化。未竟专题不得通过一个跨越整章的未闭合 TeX 分组维持状态。

### 1.2 正文计数器与转换规则

首选实现应把“结构上的顶层单元次序”和“普通正文显示编号”彻底分离，不再把标准 `chapter` 先减一再让 `\chapter` 加回。各计数器的职责为：

| 计数器 | 职责 | 是否直接显示 |
| --- | --- | --- |
| 标准 `chapter`（即 `c@chapter`） | 所有**有编号顶层单元**的结构流水号；驱动标准 reset 链和 package hook | 否 |
| 新增 `LALUnormalchapter` | 只在普通正文开始时递增，表示“第 N 讲”中的 `N` | 是 |
| `LUchapter` | 只在未竟专题开始时递增，表示“未竟专题 X”中的全局序号 | 是 |
| `LUgreekchap` | 同一普通正文之后的未竟专题 slot | 通过希腊字母显示 |
| 新增 `LALUappendixchapter` | 附录显示编号 `A`、`B`…… | 仅 appendix 模式 |
| 内部 `LALUunnumberedunit` | 每个实际无编号顶层标题的锚点流水号；阶段命令本身不递增 | 否 |

因此，普通编号章、未竟专题和编号附录都必须调用最终的标准 `\chapter`。每次这样的编号调用都会使标准 `c@chapter` 自然递增一次，LaTeX 注册在 chapter 下的 section、equation、figure、table、定理等 reset 以及 Hyperref/CTeX/其他 package hook 也自然执行一次。结构 counter 不再为了显示编号而回退、复用或伪造数值。

普通 `\chapter` 适配器必须先按 phase 和标准编号判定分派：mainmatter 的编号章走 normal 路径，appendix 的编号章走 appendix 路径，frontmatter/backmatter 中的章以及被 `secnumdepth` 抑制编号的章走 unnumbered 路径。星号形式始终走 unnumbered 路径；`\LUchapter` 则是只允许 mainmatter 编号路径的独立入口。这样同一个公共 `\chapter` 不会被“普通章流程只接受 mainmatter”的前置条件误伤。

普通章的编号路径为：

1. 确认当前处于 `mainmatter`，并采用与标准 chapter 相同的“本次是否编号”判定。若受 `secnumdepth` 等设置影响而不编号，则不得改动任何显示计数器，而应按无编号边界更新状态后调用标准章命令。
2. 仅在确定编号后设置 `kind=normal`，递增 `LALUnormalchapter`，把该值写入当前记录的 `normal-display` 和 `normal-base`，并把 LU slot 重置为 0。
3. 调用最终标准 `\chapter`；它递增结构 `c@chapter` 并执行全部标准 reset/hook。
4. `\thechapter` 在本次调用期间从 `LALUnormalchapter` 取显示值，而不是从 `\arabic{chapter}` 取值。

未竟专题转换流程为：

1. 要求当前处于 `mainmatter`、已经存在普通正文，并且标准 chapter 判定本次会编号；任一条件不满足都必须在修改任何计数器或样式前给出明确的 package error，而不是生成 `0ε` 或一个半初始化单元。
2. 将当前 `LALUnormalchapter` 快照为 `normal-base`，设置 `kind=unfinished`，并分别递增全局 `LUchapter` 与当前普通章的 `LUgreekchap`。普通章入口已经把 slot 归零，因此这里不再覆盖 `normal-base` 后做含义不清的“是否同章”比较。
3. 调用同一个最终标准 `\chapter`；结构 `c@chapter` 再递增一次并执行全部标准 reset/hook。
4. `\thechapter` 在本次调用及其正文期间返回保存的 `normal-base + Greek(slot)`；章标题的中文序号则来自 `LUchapter`。

`\chapter*` 结束当前编号单元但不清除最近普通章及其 LU slot。因而随后在 `mainmatter` 中再次调用 `\LUchapter` 是允许的，并继续使用下一个 slot；例如 `chapter A -> LU B -> chapter* -> LU C` 的两个 LU 引用仍依次为 `1ε`、`1δ`。如果希望星号章成为禁止后续 LU 的更强边界，应另设显式命令，不能通过重置 slot 得到可能重复的编号。

appendix 中的编号章由同一个 `\chapter` 适配器按 `phase` 分派：

1. 确认当前处于 `appendix` 且标准 chapter 判定本次会编号。
2. 设置 `kind=appendix`，递增 `LALUappendixchapter`，并记录本次附录显示字母。
3. 调用同一份最终标准 `\chapter`，让结构 `c@chapter` 递增并执行标准 reset/hook。
4. 本次调用及其正文中的 `\thechapter` 返回 `\Alph{LALUappendixchapter}`，锚点进入 `APP` 命名空间。该路径不得递增或重置普通正文、LU、slot 计数器。

unnumbered 路径在调用最终标准章命令前设置 `kind=unnumbered`、清空当前 `unit-id`，并为每个实际排出的无编号顶层标题递增一次内部 `LALUunnumberedunit`。它保留最近普通章、LU slot 和所有显示计数器，不递增结构 `chapter`，也不伪造编号章的 reset；仅 phase 命令本身不算一个无编号标题，不能递增该内部 serial。

以上路径都必须“先验证、后变更”；验证失败不得遗留已经递增的逻辑计数器、切换一半的 CTeX 样式或不完整的当前记录。

例如 `chapter A -> LU B -> LU C -> chapter D` 完成后，结构 `c@chapter` 依次为 1、2、3、4，普通显示计数器依次为 1、1、1、2；可见引用依次为 `1`、`1ε`、`1δ`、`2`。

现有希腊字母序列的前六项必须保持为：

1. `ε`
2. `δ`
3. `λ`
4. `μ`
5. `φ`（正文仍使用当前 `\varphi` 字形）
6. `θ`

该序列应由一张集中配置表生成正文形式和 PDF 字符串形式。超过已配置容量时应给出带有当前普通正文编号和所需 slot 的明确错误；如需扩展，只追加映射，不得改变既有六项顺序。

普通 `\chapter` 和 `\chapter*` 需要经过类在导言区末尾安装的完整签名适配器，或等价且经过验证的命令 hook。适配器必须：

- 在 Hyperref、CTeX、fncychap 等完成补丁后安装，不能像当前实现一样提前保存一个可能过期的 `\chapter` 副本；
- 保留星号形式、短标题和长标题；
- 在普通章调用前恢复普通章状态和样式；
- 让 `\chapter*` 安全结束未竟专题显示状态，但不改变最近的普通正文编号；重复执行“离开 LU”的清理必须幂等，而每个实际排出的星号章只递增其内部 unnumbered anchor serial；
- 以未展开形式保存标题，不通过 `x` 型展开写入答案文件。

这里“最终标准 `\chapter`”指 Hyperref、CTeX、fncychap 等完成补丁后的实现。公共 chapter 适配器的 normal、appendix、unnumbered 分支和 `\LUchapter` 都只能调用这一份实现；不得各自复制 `\@chapter`、分页、写目录或 reset 代码。

#### 文档阶段和无编号边界

结构 counter 与状态在特殊文档阶段遵循以下规则：

| 操作/阶段 | 结构 `c@chapter` | 显示计数器与状态 |
| --- | --- | --- |
| `\frontmatter` | 不重置为可见章号；无编号章不递增 | `phase=frontmatter`、`kind=unnumbered`；首次进入时普通/LU 显示计数器仍为 0；默认禁止 `\LUchapter` |
| 首次 `\mainmatter` | 初始化本书的结构与显示计数状态 | `phase=mainmatter`、`kind=unnumbered`、当前单元为空；首个普通章将 `LALUnormalchapter` 增至 1 |
| `\chapter*` | 不递增结构/显示 counter，也不额外触发编号章 reset | `kind=unnumbered`；结束 LU 的显示状态，但保留“最近普通正文”快照；清理可重复执行，每个实际标题取得新的内部 unnumbered serial |
| `\backmatter` | 后记、索引等无编号顶层单元不递增 | `phase=backmatter`、`kind=unnumbered`；清空当前单元，冻结普通/LU/slot 计数，默认禁止 `\LUchapter` |
| 首次 `\appendix` | **不把结构 `c@chapter` 清零** | `phase=appendix`、`kind=unnumbered`；清空当前单元，冻结普通正文和 LU 计数，将 appendix 显示计数器初始化为 0 |
| appendix 中的编号 `\chapter` | 每个附录递增一次 | `kind=appendix`，显示 `A`、`B`……；默认禁止 `\LUchapter`，因为“上一正文章节 + 希腊字母”不适用于字母附录 |

支持的阶段顺序为 `frontmatter -> mainmatter -> appendix -> backmatter`，其中 frontmatter 和 appendix 均可省略。同一阶段命令的重复调用必须幂等：重复 `\mainmatter` 不得重置结构、普通正文、LU 或 slot 计数器，重复 `\appendix` 不得把附录显示计数器清零并重新产生 `A`；重复 front/mainmatter 也不得把页码重新切到 1 而制造重复 page destination。首次合法转换可以保留底层标准类原有的清页和页码模式切换，重复调用则应成为无状态变化的空操作或只发一次诊断。任何倒退到较早阶段的调用都应明确报错，除非未来另行定义并测试一套受支持的回退协议。

标准类的 `\appendix` 通常会把 `c@chapter` 清零；新适配层不能沿用这一计数器副作用。它只应用附录名称/显示格式，并在首次进入 appendix 时初始化 `LALUappendixchapter`，结构 `c@chapter` 继续单调递增。附录锚点使用独立 ASCII 命名空间，例如 `chapter.APP.A`。如果未来确需“附录后的未竟专题”，必须先单独定义其显示和引用契约，不能隐式复用正文规则。

在 `frontmatter`、首次 `\mainmatter` 与首个普通章之间、`backmatter`、`chapter*` 之后以及 `\appendix` 与首个附录章之间调用 `exercise` 时，章节状态 API 应报告“没有可映射的编号单元”，不得沿用上一个编号章的答案映射。第一阶段的答案 schema 只支持普通正文和 LU 单元，因此附录章中的 `exercise` 也应给出明确错误；若未来要支持附录习题，必须同时补齐 appendix 的 `unit-id`、答案标题、显示编号、锚点和回归测试，而不是把它伪装成普通章。

`\LUchapter` 的兼容签名仍接受现有的单个必选参数，同时可新增可选短标题：

```tex
\LUchapter{完整标题}
\LUchapter[目录与页眉短标题]{完整标题}
```

### 1.3 显示编号、引用编号与锚点

`\thechapter` 应成为一个由当前状态分派的稳定命令：

- 普通正文返回 `\arabic{LALUnormalchapter}`。
- 未竟专题返回保存的 `normal-base` 加当前希腊字母的正文形式。
- 附录返回 `\Alph{LALUappendixchapter}`。
- 无编号状态返回空的、可展开 token list，并且语义查询 API 同时报告“没有当前编号单元”；绝不能悄悄返回上一个普通章或 LU 的显示编号。若 Hyperref 或其他包需要为实际的无编号顶层标题建立内部 destination，则另用单调递增、不可见的 unnumbered serial，使 `\theHchapter` 返回 `UNNUMBERED.<serial>`，而不占用结构或显示 counter。

CTeX 的章标题编号应使用另一条格式化路径：

- 普通正文：前缀“第”，编号为普通正文编号，后缀“讲”。
- 未竟专题：前缀“未竟专题”，编号为 `\zhnumber{unfinished-serial}`，无“讲”后缀。
- 附录：沿用当前类/CTeX 的附录标题格式，但编号必须来自 `LALUappendixchapter`，不能来自结构 `chapter`。
- 无编号状态：沿用标准星号章或 front/backmatter 的无编号标题路径，不读取任何旧单元编号。

优先把各套 CTeX 设置封装为命名样式，并在状态转换时显式、幂等地应用；如 CTeX 支持可靠的动态格式命令，也可由一个分派器生成。不得再以跨章分组恢复 CTeX 设置。

章标题/目录编号与引用编号必须有独立 provider。建议分别提供 `\LALUChapterTitleNumber`、`\LALUChapterTocNumber` 和 `\LALUCurrentReferenceNumber`：普通章三者都基于 `N`，未竟专题前两者生成“未竟专题十四”而引用 provider 生成 `23ε`，附录前两者生成类所约定的附录形式而引用 provider 生成 `A`。`\chapter` 适配器应通过 CTeX/标准章命令支持的格式接口，把 title/TOC provider 交给同一次标准章调用；如果所支持的 CTeX 版本没有独立 TOC 接口，只能在一个集中且有版本测试的窄适配层中替换该次标准目录条目的 `\numberline` 参数。不得为修正编号再写第二条 `\addcontentsline`，也不得把 `\thechapter` 临时改成中文标题编号后泄漏到标签或子计数器。

Hyperref 锚点规则如下：

| 对象 | `\theH...` 约定 | 示例 |
| --- | --- | --- |
| 普通正文 | 使用普通显示计数器，保持既有 chapter 锚点外观 | `chapter.23` |
| 未竟专题 | `LU.<unfinished-serial>` | `chapter.LU.14` |
| 未竟专题内 section | 从唯一章锚点派生 | `section.LU.14.2` |
| 附录 | `APP.<appendix-display>` | `chapter.APP.A` |
| 实际无编号顶层标题 | `UNNUMBERED.<internal-serial>`，仅作内部唯一性用途 | `chapter.UNNUMBERED.3` |
| 答案单元 | `answer.normal.<N>` 或 `answer.LU.<serial>` | `LALUanswerunit.answer.LU.14` |
| 习题组 | 包含答案单元 ID 和组次序 | `LALUexgroup.lu.14.C` |

锚点表示必须完全可展开并只含 ASCII。`\theHchapter` 同样按 `kind` 分派：普通章返回普通显示编号，未竟专题返回 `LU.<unfinished-serial>`，附录返回 `APP.<appendix-display>`，无编号状态在确需 chapter 型锚点时返回 `UNNUMBERED.<serial>`。它不能直接使用结构 `c@chapter`，否则在插入未竟专题后会改变普通章既有外链。`LUchapter` 只是标题流水号时使用 `\stepcounter`；真正承载紧随其后 `\label` 的标准 `chapter` 才使用 `\refstepcounter`，避免创建无用的第二锚点。

标准 `\chapter*` 若已由 Hyperref 使用自己的 `chapter*.<linkcounter>` 命名空间，就继续沿用该机制；`UNNUMBERED.<serial>` 只是其他包确实展开 `\theHchapter` 时的确定性后备值，不能再额外创建一个与标准星号章重叠的 destination。

为了最大限度兼容已有 PDF 外链，新实现应优先保留当前未竟专题锚点 `chapter.LU.<serial>`。普通章节锚点保持标准形式。若任何锚点必须改变，应在发布说明中列出，并考虑在一个迁移版本中添加旧 destination 别名。

默认引用语义保持如下：

- `\ref`：普通正文为 `23`，未竟专题为 `23ε`。
- `\nameref`：返回章节标题。
- `\autoref`：兼容现有 chapter 类型，输出“章 23ε”。若未来希望输出“未竟专题 23ε”，应作为单独的、有迁移说明的引用类型变更，而不在本次底层重构中夹带改变。
- `\cref`：与 `\autoref` 保持相同对象类型，格式由类集中配置。

### 1.4 目录与 PDF 书签

目录和书签应复用标准章命令产生的单一条目，不同时手写第二个 `\addcontentsline`。期望结果为：

- 普通正文目录条目显示“第 23 讲 标题”。
- 未竟专题目录的章级条目显示“未竟专题十四 标题”，而不是 `23ε 标题`。
- 未竟专题内 section 的目录编号显示 `23ε.1`。
- PDF 书签使用 Unicode 希腊字符，例如 `23ε.1`，不得含 `$...$`、`\boldsymbol` 或其他 PDF string 不可用命令。
- 目录条目和书签指向同一个唯一章锚点。

正文希腊字形与 PDF 字符串字形应分别定义，并通过 `\pdfstringdefDisableCommands` 或一个经过测试的 `\texorpdfstring` 边界连接。不能假定嵌入 `\thechapter` 的数学命令在写入 `.toc`、生成书签和二次读取时都以相同方式展开。

### 1.5 `LALU-answers` 中的答案单元

答案册不应再把普通章节和 `\LUsection` 分别实现为两套跨 section 分组逻辑。建议引入专用逻辑计数器 `LALUanswerunit` 和统一渲染接口：

```tex
\LALUAnswerUnit{
  kind=unfinished,
  normal-base=23,
  unfinished-serial=14,
  unfinished-slot=1,
  title={范畴论视角下的线性代数},
  unit-id={lu:14}
}
```

它负责：

- `\refstepcounter{LALUanswerunit}` 并重置属于答案单元的公式、定理和组计数器；
- 以 section 层级排版标题、写入目录和页眉，但不篡改全局 `section` 的意义；
- 普通单元显示“第 N 讲”，未竟单元显示“未竟专题 X”；
- `\theLALUanswerunit` 对普通单元返回 `N`，对未竟单元返回 `Nε`；
- 使用独立、唯一的 `\theHLALUanswerunit`；
- 在生成答案部分结束后无需把 `\thesection` 清空，因此不再破坏后续历年卷的公式编号。

可使用 `\section*` 加一次受控的目录/书签写入来保持现有视觉层级；公式和定理通过专用 `LALUanswerunit` 建立 reset 关系。这样答案册后半部分的普通 `\chapter`、`\section` 完全不受生成答案部分影响。

新答案清单必须携带完整章节记录，不能让答案册根据“上一个 section 是什么”猜测 `normal-base` 或希腊 slot。`\LUsection` 保留为过渡包装器，仅用于读取旧的已生成文件，并给出一次性弃用警告；干净构建生成的新文件不得再输出 `\setcounter{LUsection}{...}` 或 `\LUsection`。

### 1.6 兼容迁移

迁移期建议遵循以下策略：

- `\LUchapter{...}` 原调用不需要修改；新增短标题是向后兼容扩展。
- 暂时保留 `LUchapter`、`LUgreekchap`、`LUsection` 计数器名字，并由新状态模型同步其值；新增的 `LALUnormalchapter` 是普通正文显示编号的权威来源。
- `\LUgroupsancheck` 在一个过渡版本中保留为空操作并发出弃用提示，使主文件尾部或缓存生成文件不会立即报未定义；确认仓库内调用移除后再删除。
- 旧的 `\@exist@LUchapter@...`、`\@exist@LUsection@...` 不再作为真值来源。若外部文档确有探测，可在过渡版只生成兼容标记，但内部逻辑不得读取它们。
- 保留已有标签键和 `chapter.LU.<serial>` destination；所有 `.aux`、`.toc`、`.out`、`.fdb_latexmk` 应在切换实现时统一清理并完整重编译。
- 新旧答案生成器先并行输出到不同构建目录，比较单元顺序、标题、组号、题号和答案正文；一致后才切换 `LALU-answers.tex` 的输入。
- 旧 `LALU-ans-contents.tex` 是可丢弃构建产物，不承诺由新 reader 永久支持。过渡包装器只用于平滑升级，不能成为新状态模型的长期分支。

这一首选模型会有意改变裸 `\value{chapter}`、`\arabic{chapter}` 和直接读取 `c@chapter` 的内部语义：它们返回的是“第几个有编号顶层单元”，不再等于普通正文的显示编号。迁移前必须全仓库扫描这些用法，并根据意图替换为语义 API：

- 需要普通正文显示编号时使用 `\LALUNormalChapterNumber` 或对应 expl3 查询接口；
- 需要当前可见引用时使用 `\thechapter`/`\LALUCurrentReferenceNumber`；
- 需要判断普通章或未竟专题时读取 `kind`，不能比较计数器；
- 只有 reset、checkpoint 或底层诊断代码可以读取结构 `c@chapter`。

当前仓库扫描结果是：`\value{chapter}`、`\arabic{chapter}`、`\setcounter{chapter}`、`\addtocounter{chapter}` 或 `c@chapter` 这类 raw 模式只出现在主模板现有章节逻辑中；答案模板还会重定义 `\thechapter` 并根据章节状态组织答案，因此两份模板都必须迁移，而正文内容没有直接依赖这些 raw 模式。迁移后应对全部手写源码启用静态检查。检查只允许统一章节模块中为 reset、checkpoint 或底层断言而登记的少量结构用途白名单，并排除 `.aux` 等生成文件；不能仅检查正文而放过模板内部继续把 raw chapter 当显示号的代码。

如果实际外部使用者证明必须保持“裸 `chapter` 值等于普通正文显示编号”，才启用次选兼容方案：在一个严格封装且带断言的适配器中，未竟专题调用前把 `chapter` 临时减一，再交给标准 `\chapter` 加回。该方案仍优于跨章开放分组，但会复用结构 counter、增加异常恢复复杂度，也使 hook 观察到的计数值依赖调用阶段；它不应作为默认设计。

### 1.7 章节与链接测试矩阵

最小回归文档应覆盖下列序列：

| 场景 | 预期标题/引用 | 重点断言 |
| --- | --- | --- |
| `chapter A` | 第 1 讲；`\ref=1` | 结构 counter=1，普通显示 counter=1，锚点 `chapter.1` |
| `chapter A -> LU B` | 未竟专题一；`\ref=1ε` | 结构 counter=2，普通显示 counter 仍为 1 |
| `chapter A -> LU B -> LU C` | 未竟专题一/二；`1ε`/`1δ` | 结构 counter=3；slot 与 LU 流水号连续 |
| 上述序列后 `chapter D` | 第 2 讲；`\ref=2` | 结构 counter=4，普通显示 counter=2，锚点 `chapter.2` |
| 新普通章后首个 LU | `2ε` | slot 正确重置为 1 |
| 同章后第六个 LU | `Nθ` | 既有六项映射不变 |
| 同章后第七个 LU | 明确错误 | 不产生空编号或重复锚点 |
| `LU -> chapter* -> chapter* -> chapter` | 最终普通章正常 | 两个星号章不递增结构/显示 counter，各自内部锚点唯一；无错误分组 |
| `chapter -> LU -> chapter* -> LU` | `1ε` 后为 `1δ` | 星号章不重置最近普通章或 slot；第二个 LU 建立新结构单元 |
| frontmatter 中的 `\chapter` 与 `\chapter*` | 无编号标题正常 | 结构、普通、LU counter 均不递增，current unit 为空，LU 调用被拒绝 |
| mainmatter 首个普通章之前调用 LU 或 `exercise` | 明确错误 | 当前单元为空，所有 counter 与样式保持初始状态 |
| mainmatter 中重复调用 `\mainmatter` | 章节编号和页码连续 | 不重置结构、普通、LU、slot 或页码，不产生重复 page destination |
| backmatter 中的 `\chapter`、`\chapter*` 与 index | 无编号标题正常 | current unit 为空，counter 全部冻结，不依赖手工 sanity check |
| 重复 `\appendix` 或从 appendix/backmatter 回到 mainmatter | 前者保持下一附录字母，后者明确报错 | 不重置 appendix/结构 counter，不产生重复 `A` |
| `appendix -> chapter A -> chapter B` | 附录 A/B | 结构 counter 各递增一次；标题、ref、TOC、outline 使用附录 provider；普通/LU counter 冻结；锚点在 APP 命名空间；chapter-dependent counter 正常 reset |
| appendix 入口空隙或附录章中的 `exercise` | 明确错误 | 不沿用最后一个正文单元；首阶段不生成不完整的 appendix 答案记录 |
| 降低 `secnumdepth` 后的普通章与 LU | 普通章走无编号路径，LU 原子失败 | 不预先递增任何逻辑 counter，不残留 LU 样式 |
| 带短标题和数学标题 | 正文、目录、书签各自正确 | 无 PDF string 警告 |
| LU 内 section/equation/theorem/figure | 均以 `Nε` 为父编号 | 对应 `\theH...` 唯一 |
| 正文 `\label` + `xr` 外部引用 | 数值和名称正确 | 冷启动与增量构建一致 |
| `\include`/`\includeonly` 跨普通章、LU、附录 | 全量与选择构建状态一致 | LaTeX checkpoint 恢复全部 counter；非 counter 状态由可重建记录恢复或明确禁止不安全的选择构建 |
| 答案普通单元 -> LU 单元 -> 普通单元 | 显示与正文映射一致 | 后续历年卷 section/公式不受污染 |
| 全仓手写源码静态扫描 | raw chapter 只剩登记的结构用途 | 模板和正文均无把 `value/arabic/setcounter/addtocounter{chapter}` 或 `c@chapter` 当显示号的调用；生成文件被排除 |

测试至少在受支持的最旧 TeX Live 与当前 TeX Live 上使用 XeLaTeX 执行。若 LuaLaTeX 不在支持范围，应在类加载时明确拒绝，而不是让字体、索引或书签在中途失败。日志检查必须把以下内容视为失败：

- `destination with the same identifier`；
- 未定义引用或多轮后仍要求 rerun；
- 由章节编号产生的 `Token not allowed in a PDF string`；
- 章节状态断言失败；
- 目录和 PDF outline 中出现重复章条目。

## 二、答案抽取器

### 2.1 单一源数据与记录模型

答案的权威源仍是作者编辑的源码；任何生成的 `.tex` 或 manifest 都是可删除、不可手工编辑的构建产物。每条答案记录至少包含：

- schema 版本；
- 主文档身份和源文件哈希；
- 当前章节记录的完整快照；
- 习题组序号及显示标签；
- 题目在组内的逻辑序号、显示标签和稳定 ID；
- 答案状态：`answered`、`todo` 或 `omitted`；
- 正文载荷类型：`inline-token` 或 `file`；
- 可选源文件与行号，仅供诊断。

稳定 ID 的推荐缺省形式为 `<unit-id>/<group>/<item>`，例如 `lu:14/C/2`。作者可显式指定更稳定的语义 ID；一旦发布，不应因插入前一题而改变已有 ID。

生成文件应是语义事件流，而不是预先写死 `\section`、`\setcounter`、`\subsection*` 等版式命令。例如 manifest 只允许调用：

```tex
\LALUAnswerManifestBegin{...}
\LALUAnswerUnit{...}
\LALUAnswerGroup{...}
\LALUAnswerItem{...}{...}
\LALUAnswerManifestEnd{...}
```

答案册 reader 负责把记录渲染成标题和列表。这样正文编号策略改变时无需重新设计文本序列化格式。

### 2.2 编号与普通/未竟映射

进入 `exercise` 时，抽取器必须从章节状态 API 取得快照，禁止再通过某个动态控制序列是否存在来判断普通章或未竟专题。

编号规则如下：

- `exgroup` 使用普通 `\stepcounter`；若组本身支持 `\label`，其 Hyperref ID 必须包含 unit ID 和组序号。
- 可选显式组号先设置逻辑值，再通过同一条组初始化路径生成标题和锚点；不能让显式组号分支跳过锚点或 reset。
- 答案题号来自与题目绑定的逻辑 exercise item，而不是在 `answer` 结束时临时读取固定的 `\theenumi`。
- 嵌套 enumerate 只影响答案正文内部编号，不得改变所关联的外层题号。
- 答案册使用 manifest 中的 `normal-base`、`unfinished-serial` 和 `unfinished-slot` 渲染单元，绝不根据前一个已输出答案单元推算。

迁移初期可自动生成稳定 ID 并记录告警；中长期建议提供 `\exitem[id=...]` 或等价接口，让题目和答案的关联显式化。

### 2.3 token 保真与 verbatim

单一接口不能同时可靠地捕获任意 TeX token 和 verbatim 内容，因此应明确提供两种载荷：

1. `inline-token`：兼容当前 `\begin{answer}...\end{answer}`。支持平衡花括号、段落和普通宏；以未展开 token 保存，禁止使用 `\tl_to_str:n`、`x` 型完全展开或 `\scantokens`。该模式明确不支持 verbatim、minted 及依赖特殊 catcode 的内容。
2. `file`：答案正文保存在独立 `.tex` 片段中，manifest 只记录路径，答案册在正常输入阶段 `\input` 它。verbatim、listing、复杂 catcode 和很长答案必须使用该模式。

不建议编写一个外部正则表达式解析器去“理解”任意 TeX 并复制 `answer` 环境：嵌套环境、注释、条件分支和 verbatim 会使该方案不可可靠。迁移脚本可以机械地拆分已知格式，但拆分结果必须经过 TeX 编译和内容对比。

标题、题号等元数据与答案正文分开序列化。元数据需要 PDF 字符串时单独转换；不得为了生成元数据而展开正文。生成文件应保留合理换行，避免每条长答案变成单个超长输入行。

需要额外验证答案正文中的以下对象：

- `\label`、`\ref`、`\nameref` 和从正文导入的 `xr` 标签；
- 浮动体与图片相对路径；
- 脚注、索引项、局部宏；
- display math 末尾的 QED；
- verbatim 文件载荷。

为避免答案册自身标签与 `\externaldocument` 导入的正文标签相撞，建议给外部标签添加固定前缀并提供 `\mainref`/`\mainnameref` 辅助命令；是否启用应先以标签清单验证兼容性。

### 2.4 空答案与占位语义

“没有 `answer` 环境”和“显式存在一个空 `answer` 环境”都不应默认制造答案册空白页。建议规则为：

- 单元中没有 `answered` 或显式 `todo` 记录时，不输出该答案单元、目录条目或 `\clearpage`。
- 组中没有可见记录时，不输出组标题。
- 纯空白的兼容 `answer` 正文默认为 `omitted`，构建日志报告数量。
- 作者希望保留占位时必须显式写 `status=todo`；reader 统一显示“暂无答案”或配置的占位文字。
- 可提供仅供编辑审校的 `include-empty-units=true`，正式发布构建固定为 false。

每次生成结束应报告：章节单元数、习题组数、题目数、已答数、todo 数、空答案数和被省略单元数。异常下降可由 CI 阈值发现。

### 2.5 原子生成与失败恢复

TeX 主进程不应直接截断最后一个可用的正式答案文件。建议由构建辅助程序实施以下事务：

1. 在目标构建目录创建同一文件系统上的唯一临时目录或临时文件。
2. 主文档只向临时目标写 manifest 和 inline body。
3. 写入 schema header、记录总数以及结束 sentinel；关闭所有 stream。
4. 辅助程序验证退出状态、sentinel、记录计数、所有 file 载荷和依赖均存在，并可选择执行一次 reader 语法检查。
5. 验证成功后，以原子 rename 替换当前 manifest；验证失败则保留上一份成功产物并删除或隔离临时文件。

纯 TeX 无法跨平台可靠地完成最终原子 rename；该动作应放在 latexmk rule 或小型构建脚本中，而不是依赖 `\write18`。manifest 应带 schema 版本和主源码指纹，答案册发现不完整、过期或版本不兼容时必须失败，不能静默排版旧答案。

### 2.6 并发、jobname 与构建目录

固定文件名 `LALU-ans-contents.tex` 不再作为内部协议。输出身份至少由以下内容组成：

- 显式 ASCII jobname；
- 主文件规范化路径的短哈希，防止两个同名工程相撞；
- 会影响答案内容的构建变体；
- schema 版本。

推荐目录形如：

```text
build/answers/LALU-<source-hash>/
  manifest.tex
  bodies/
  generation.json
```

不同 jobname 或工程可并行写不同目录。同一身份的并发构建使用每进程唯一临时目录，并通过锁或 compare-and-swap 决定谁发布最终 manifest；绝不能共同写同一个 open stream。答案册由构建命令显式传入 manifest 路径，源码中不硬编码当前工作目录。

`make clean` 只删除已知构建目录，不删除作者维护的 file 载荷。生成内容不写回源码目录，也不参与版本控制。

### 2.7 答案抽取器测试

至少建立以下 fixture：

- 普通章 A/B/C 三组，部分题有答案；
- 普通章后的两个连续未竟专题，验证 `Nε`、`Nδ` 与中文全局序号；
- 没有任何答案的 exercise、空组、空 answer、显式 todo；
- 自定义起始组号和超过 9 的题号；
- 答案含多段、嵌套 enumerate、宏参数符号、注释、Unicode、标签和浮动体；
- file 载荷含 verbatim/listing；
- 主编译在写到一半时故意失败，旧 manifest 仍可用；
- 两个不同 jobname 并行构建以及同一 jobname 的竞争构建；
- manifest 缺尾标、记录计数错误、源哈希过期、file 载荷缺失时 reader 明确失败；
- 新旧生成器的记录顺序和可见答案逐条对比。

## 三、定理、证明与引用框架

### 3.1 职责划分

移除 `ntheorem` 与自定义手写 QED 的并行语义层，改成清晰的两层职责：

- `tcolorbox` theorem library 是 definition、example、lemma、theorem、corollary、axiom 的声明、计数器、标题栏和分页视觉层。
- `amsthm` 是 proof 的语义与 QED 栈实现，保证 `\qedhere`、嵌套证明和显示公式末尾行为正确。
- `tcolorbox` 只包装 proof/solution 的视觉外框，不自行在环境尾部附加 `$\square$`。

推荐加载顺序为：数学基础包与 `mathtools`，随后 `amsthm`，再加载 `tcolorbox` 及所需 library，最后加载 `hyperref`、`bookmark`、`cleveref`。`ntheorem` 完全退出，不保留同时生效的 theorem style 命令。

### 3.2 定理类环境

用一张集中声明表维护每个环境的属性：

| 环境 | 中文名 | 标签前缀 | 颜色 | 缺省计数父级 |
| --- | --- | --- | --- | --- |
| `definition` | 定义 | `def:` | red | chapter |
| `example` | 例 | `ex:` | blue | chapter |
| `lemma` | 引理 | `lem:` | orange | chapter |
| `theorem` | 定理 | `thm:` | violet | chapter |
| `corollary` | 推论 | `cor:` | green | chapter |
| `axiom` | 公理 | `axm:` | olive | chapter |

初次迁移应保持当前各环境分别计数的行为；是否共享 theorem 计数器属于独立的版式决策，不应在底层替换时顺便改变。正文中 `number within=chapter` 继续依赖统一章节模型，所以未竟专题内的显示编号和锚点都会自然使用唯一的 LU chapter 信息。

公共调用语法在兼容阶段保持不变：

```tex
\begin{theorem}{显示标题}{标签键}
  ...
\end{theorem}
```

第二参数非空时继续自动生成 `thm:<标签键>` 一类完整标签；空参数不生成标签。现有可选 tcolorbox 参数若实际有人使用，也必须由兼容层透传。未来可增加 key-value 新接口，但应先提供机械迁移工具和弃用周期，不能要求一次性人工修改全部现有环境。

视觉样式集中为一个基础 style，再按环境覆盖颜色。基础 style 负责 breakable、边框、标题字体、正文字体、首/中/末段虚线等当前特征。任何视觉调整都通过小型 golden PDF 检查，避免语义迁移意外改变分页。

### 3.3 proof、solution 与 QED

`proof` 使用 amsthm 的 QED 栈；tcolorbox 可通过 `\tcolorboxenvironment` 装饰 amsthm proof，或使用一个严格转发到 amsthm begin/end 的包装器。无论采用哪种方式，都必须满足：

- 普通文本结尾自动出现且只出现一个 QED；
- display math、`equation`、`align` 末尾的 `\qedhere` 位置正确，不再另加第二个方块；
- 跨页 breakable box 的 QED 位于证明内容结束处和盒子内部；
- 嵌套 proof 使用独立 QED 栈；
- 可选说明保持当前“证明 + 说明文字”的可见效果。

`solution` 复用同一视觉框架，但默认不显示 QED，以保持现有行为。若未来需要 QED，使用显式选项，不通过正文手写 `\square` 推测。

当前 `proof`/`solution` 的第二个可选参数可暂时作为原始 tcolorbox 选项透传；应扫描实际调用，若未使用则标记弃用，改由受控 key 列表代替任意样式注入。

### 3.4 `autoref` 与 `cleveref`

所有定理环境创建后，由同一声明表生成引用名称，不能散落手写私有 counter 名：

- 为 Hyperref 配置对应 `...autorefname`，保证 `\autoref` 输出“定理 3.2”等。
- 为 cleveref 同时配置 `\crefname`、`\Crefname`、range 和 multiple 格式。
- 现有标签前缀保持不变；`\ref`、`\autoref`、`\cref`、`\crefrange`、`\nameref` 都加入回归测试。
- 如果 tcolorbox 的内部 counter 名随版本变化，类应在一个内部适配层解析，正文不得依赖 `tcb@cnt@...`。
- 新正文优先推荐 `\cref`；`\autoref` 作为兼容接口继续受支持。

每个 theorem counter 的 `\theH...` 必须从统一的唯一 chapter anchor 派生。正常章和连续未竟专题中，同样显示为 `.1` 的定理也必须拥有不同 destination。

### 3.5 定理框架迁移兼容

迁移步骤如下：

1. 统计所有环境调用形式、可选参数、空标题、空标签和手写 `\qedhere`/`\square`；建立代表性 fixture。
2. 在测试分支中用 `amsthm` 替换 `ntheorem`，但保留现有 tcolorbox theorem 公共环境名、双参数语法、标签前缀和分别计数方式。
3. 将 `\theoremheaderfont`、`\theorembodyfont` 的效果搬到集中 tcolorbox style；删除 ntheorem 命令前先验证字体快照。
4. 用 amsthm proof + tcolorbox wrapper 替换手写尾部方块，修正真正需要 `\qedhere` 的少量正文。
5. 集中生成 autoref/cleveref 配置并比较旧、新 `.aux` 中的标签值。
6. 一个发布周期内保留旧 proof/solution 可选参数适配；弃用告警应包含源文件和行号。

迁移提交必须要求干净重编译辅助文件。除明确列出的 QED 位置修正外，既有标签键和可见定理编号不得变化。

### 3.6 定理测试矩阵

| 场景 | 验收点 |
| --- | --- |
| 六类环境各一个，标题/标签为空或非空 | 标题栏、字体、颜色、编号、标签前缀正确 |
| 同一普通章内各环境多个实例 | 分别计数规则与当前行为一致 |
| 两个连续未竟专题 | 显示父编号为 `Nε`/`Nδ`，锚点无重复 |
| 长定理跨两页 | 首/中/末边框正确，链接目标在标题处 |
| proof 以普通段落结束 | 恰有一个 QED |
| proof 以 equation/align 结束并使用 `\qedhere` | 方块位置正确且不重复 |
| 嵌套 proof | QED 栈正确 |
| solution | 缺省无 QED |
| `ref/autoref/cref/crefrange/nameref` | 中文名称、范围格式、链接目标正确 |
| 答案单元中的定理与证明 | 使用答案单元父编号，不污染后续历年卷 |

## 四、Breaking changes 与源码迁移

这里把“breaking change”限定为：升级模板后，旧源码、构建命令、生成文件或外部链接即使重新编译，也不能自动保持原有语义，必须改写或显式迁移。单纯的内部实现替换、可接受的分页微调，以及清理辅助文件后即可恢复的变化不单独算作 breaking change。

### 4.1 明确会发生的兼容性中断

| 计划中的变化 | 中断点 | 当前仓库中受影响的位置与定位方法 | 新写法或迁移动作 |
| --- | --- | --- | --- |
| 标准 `chapter` 改作所有有编号顶层单元的结构流水号 | 裸 `\value{chapter}`、`\arabic{chapter}`、`c@chapter` 及手工 `\setcounter`/`\addtocounter` 不再表示普通“第 N 讲” | 当前只在 `讲义/线性代数荣誉课辅学讲义.tex` 的章节和答案生成逻辑中出现；`讲义/专题/` 与 `讲义/其它/` 的手写正文目前没有这类依赖 | 普通讲次改读 `\LALUNormalChapterNumber`，当前可见引用改读 `\LALUCurrentReferenceNumber` 或 `\thechapter`，单元类型改查 `kind`；正文不得再直接改 `chapter` |
| 无编号状态不再泄漏上一编号单元 | frontmatter、backmatter 或 `\chapter*` 后的 `\thechapter` 将为空；依赖其继续返回上一章号的宏会改变行为 | 当前手写正文没有这种读取；迁移时仍须把 `\thechapter`/`\theHchapter` 纳入模板和扩展宏扫描 | 需要最近普通讲次时显式查询 `\LALUNormalChapterNumber`，需要当前单元时先检查状态 API；不得把无编号边界后的 `\thechapter` 当历史快照 |
| 删除跨章分组和历史 marker | `\LUgroupsancheck`、`\LUsection` 以及 `\@exist@LUchapter@...`/`\@exist@LUsection@...` 最终会消失 | 定位于主模板、`讲义/LALU-answers.tex` 和生成的 `LALU-ans-contents.tex`；各个未竟专题文件开头的 `\LUchapter{...}` 不受影响 | 删除主文件末尾等处的手工 sanity check；答案册只消费 `\LALUAnswerUnit` 等 manifest 事件，不再手工设置 `LUsection` 或探测 marker |
| 对 phase 和单元位置执行严格校验 | 以前“碰巧可编译”的 `mainmatter` 首章之前的 LU、front/backmatter 或 appendix 中的 LU/习题、`secnumdepth` 抑制编号时的 LU，以及非法阶段回退将直接报错；重复 phase 命令不再重置页码或计数器 | 从主文件的 `\frontmatter`、`\mainmatter`、`\appendix`、`\backmatter` 和各文件的 `\LUchapter`、`\begin{exercise}` 定位；当前正式输入顺序没有发现违规项 | 在 `\mainmatter` 中先建立有编号的普通 `\chapter`，再写 `\LUchapter`；习题只放在普通章或 LU 单元内。`\chapter*` 后不能直接挂习题，但仍可继续写下一个 LU；首阶段不支持附录习题 |
| 答案交换格式和输出路径改为带版本的 manifest | `LALU-ans-contents.tex` 的文件名、内容语法和可从源码目录直接 `\input` 的约定不再保证 | 主要影响主模板、`讲义/LALU-answers.tex`、`讲义/Makefile`、latexmk 配置和 CI，不要求逐篇改普通 inline 答案 | 构建命令把 manifest 路径显式传给答案册；只通过 manifest reader 读取 `\LALUAnswerUnit`/`Group`/`Item` 事件，生成文件一律不可手工编辑 |
| 空答案和空答案单元获得明确状态语义 | 纯空白 `answer` 将变为 `omitted`；完全没有 `answered`/`todo` 的习题单元不再输出标题、目录项或空白分页 | 当前有 29 个纯空 `answer`，集中在第 2、3、4、7、8、9、10 讲；另有 15 个完全没有 `answer` 的单元，分布在第 6、12、16–25 讲及未竟专题 4、8、14 | 确实没有答案时删除空环境或接受省略；需要保留答案项或整个单元时，至少加入一个计划接口 `\begin{answer}[status=todo] ... \end{answer}` |
| 答案单元和附录采用新的机器锚点命名空间 | 答案单元切换到 `LALUanswerunit.answer.*`，附录使用 `chapter.APP.*`；硬编码到旧 href 的外部 PDF 深链接或工具会失效 | 当前正文没有编号附录，正文的 `\label`/`\ref` 键原则上不改；主要迁移面是答案 PDF 的外部链接和下游 PDF 工具 | 发布说明列出 href 映射并尽可能提供一个版本的旧锚点别名；同时清理辅助文件并重建主讲义、答案册和 `xr` 缓存 |
| `ntheorem` 退出依赖面 | 外部文档若依赖类“顺便加载” `ntheorem`，或直接调用它的样式/声明命令，将不再工作 | 当前正文没有直接调用；仓库内只有 `LALUbook.cls` 的 `\theoremheaderfont`、`\theorembodyfont` 等模板设置需要迁移 | 自定义定理接入新的集中声明表和 amsthm/tcolorbox 适配层；外部文档不得依赖类的传递包依赖 |

上表中的“计划接口”只是目标写法，目前尚不是可执行命令；它们应在实现对应阶段时固定下来。一旦进入作者文档和正文，就必须遵循正常的弃用周期，不能在后续提交中无迁移说明地再次改名。

### 4.2 条件性 breaking changes 与迁移审计

下列事项要么只有在采用可选设计时才构成兼容性中断，要么必须人工审计，但不能预先假定所有命中处都需要改写：

- 如果答案册为 `\externaldocument` 加固定前缀，答案正文中指向主讲义的 `\ref`/`\nameref`/`\autoref`/`\cref` 要改为 `\mainref`/`\mainnameref`/`\mainautoref`/`\maincref` 等成套 helper。当前入口位于 `讲义/LALU-answers.tex`；启用前应先扫描所有 `answer` 载荷中的引用并验证没有同名标签。
- 如果稳定习题 ID 从“建议”升级为“必填”，`exercise` 中相应的普通 `\item` 要改为 `\exitem[id=...]`。迁移初期必须继续自动生成 ID 并告警，不能一次性破坏现有全部习题。
- amsthm proof 的公共语法和普通自动 QED 行为应保持兼容，但 QED 的具体位置需要视觉审计。当前有 27 个 proof 直接以 display 结束；典型位置为 `讲义/专题/9 矩阵运算进阶.tex:443`、`讲义/专题/13 多项式.tex:542` 和 `讲义/专题/21 线性代数与几何.tex:49`。只有希望方块留在最后一行公式内的用例才需加入 `\qedhere`，不能把 27 处一律机械改写；当前正文没有手写 `\square`。
- inline `answer` 本来就不保证支持 verbatim、minted、listing 或特殊 catcode，因此 file payload 是新增能力而非当前仓库的 breaking change。若外部答案依赖这些“碰巧工作”的未保证行为，则应把内容移到独立片段，并通过计划接口 `\answerfile{answers/<stable-id>.tex}` 引用；当前基线没有此类载荷。
- 如果在兼容期后移除 proof/solution 的第二个原始 tcolorbox 选项，外部源码中的 `\begin{proof}[说明][任意样式]` 或同形 `solution` 要迁移到受控 key。当前仓库只有第一可选参数用例，没有发现第二可选参数。
- 如果在更晚版本移除 `LUchapter`、`LUgreekchap`、`LUsection` counter alias，外部源码对这些 counter 的直接读取也会失效；当前仓库中的直接使用只在模板和生成文件内，应先由统一状态 API 取代。
- 其他 section、公式、定理等后代 destination 只有在确认现值冲突时才改变；普通章和 LU 的既有 destination 仍以保留为默认。若不得不改变，必须按外部 PDF 深链接的 breaking change 处理，而不能把“清理 `.aux` 即可”当作充分迁移说明。
- 支持引擎矩阵尚未最终确定；一旦决定不支持 LuaLaTeX 等引擎，这些非正式构建流程将改为早期失败。当前 Makefile 使用 XeLaTeX，不影响正文；若要把其他引擎列为受支持，须先补齐字体、索引、书签和回归测试。
- 如果无法可靠恢复 `\includeonly` 所需的非 counter 状态，可以明确禁止跨普通章/LU 的选择构建。当前主讲义使用 `\input`，因此仓库正文不受影响；外部使用者若依赖 `\includeonly`，应改为完整构建或等待 checkpoint 协议落地。

### 4.3 明确保留兼容的作者接口

以下常用正文写法不应因本次重构而修改：

- 普通章继续写 `\chapter{标题}`，未竟专题继续写 `\LUchapter{标题}`；新增的 `\LUchapter[短标题]{完整标题}` 只是可选扩展。
- 普通 inline 答案继续写 `\begin{answer} ... \end{answer}`；只有空占位和特殊 catcode 载荷需要迁移。
- definition、example、lemma、theorem、corollary、axiom 继续保留当前双参数形式和标签前缀。
- `\begin{proof}[说明]` 与 `\begin{solution}[说明]` 的第一可选参数继续有效。
- 既有 `\label` 键、普通章 `chapter.N` 和未竟专题 `chapter.LU.N` 锚点应保留；正常的 `\ref`、`\nameref`、`\autoref` 和 `\cref` 源码不要求批量改写。

### 4.4 迁移时的定位清单

实现各阶段前至少运行下列扫描；它们用于定位受影响的写法，不代表匹配项都必须修改：

```zsh
rg -n --glob '*.tex' --glob '*.cls' '\\(value|arabic|setcounter|addtocounter)\{chapter\}|c@chapter|\\theH?chapter' .
rg -n --glob '*.tex' '\\LUgroupsancheck|\\LUsection|@exist@LU(chapter|section)' 讲义
rg -n --glob '*.tex' '\\(frontmatter|mainmatter|appendix|backmatter|LUchapter)|\\begin\{exercise\}' 讲义
rg -n -U --glob '*.tex' '\\begin\{answer\}(?:[[:space:]]|%[^\n]*\n)*\\end\{answer\}' 讲义
comm -23 <(rg -l -F '\begin{exercise}' 讲义/专题 讲义/其它 | sort) <(rg -l -F '\begin{answer}' 讲义/专题 讲义/其它 | sort)
rg -n --glob '*.tex' --glob '*.cls' '\\(theoremheaderfont|theorembodyfont|newtheorem)' .
rg -n --glob '*.tex' '\\begin\{(proof|solution)\}|\\(qedhere|square)' 讲义
rg -n -U --pcre2 --glob '*.tex' '(?:\\\]|\\end\{(?:equation\*?|align\*?|alignat\*?|gather\*?|multline\*?|displaymath)\})\s*(?:%[^\n]*\n\s*)*\\end\{proof\}' 讲义
```

迁移顺序应是：先改类、主模板、答案模板和构建脚本，再处理扫描命中的少量正文特殊情况，最后删除全部生成文件做一次冷构建。普通章节与未竟专题文件不应为了适配内部状态机而插入新的 helper 命令。

## 五、分阶段落地顺序

### 阶段 0：冻结基线与建立检查工具

- 保存当前完整 PDF、答案 PDF、`.aux` 标签表、`.toc`、PDF outline 和归一化日志。
- 建立章节、答案、定理三个最小回归文档。
- 在 CI 中加入重复 destination、未定义引用、PDF string 警告和残留 rerun 的失败规则。
- 统计当前答案记录数、空答案数、定理环境调用签名及标签前缀。

这一阶段只增加测试和基线，不改变产物。

### 阶段 1：引入统一章节状态，保持公共接口

- 在类中加入结构 `chapter`、普通显示计数器、LU 流水号、slot、appendix 显示计数器以及统一章节记录。
- 提供语义查询 API，替换模板内部所有把裸 `chapter` 当普通显示编号的用法，并启用静态检查。
- 在既有 `\chapter`、`\LUchapter` 外观后切换到新状态机。
- 移除跨章节开放分组和历史 marker 的内部依赖；兼容命令暂留。
- 同时改造答案册的统一 `LALUanswerunit` renderer，但暂不切换答案生成格式。
- 通过章节测试矩阵和旧、新目录/outline 对比后合并。

### 阶段 2：答案 manifest 双写与原子发布

- 定义带版本的记录 schema、reader 和 file 载荷接口。
- 构建系统同时生成 legacy 文件与新 manifest，分别编译答案册并逐记录比较。
- 加入临时目录、结束 sentinel、原子 rename、过期检查和并发测试。
- 将含 verbatim 或 catcode 风险的答案迁移到 file 载荷。
- 对比通过后切换答案册读取新 manifest；保留一次发布周期的 legacy reader。

### 阶段 3：统一定理和证明框架

- 移除 ntheorem 语义层，引入 amsthm proof/QED。
- 保持 tcolorbox theorem 的旧环境名、双参数语法、计数器行为和标签前缀。
- 集中 theorem 声明、样式及 autoref/cleveref 配置。
- 修正并测试 display math 的 `\qedhere`，进行分页与字体 golden PDF 对比。

### 阶段 4：清理兼容层

- 在确认仓库及外部维护文档已迁移后，删除 `\LUgroupsancheck`、动态 `@exist@...` marker、旧答案 reader 和原始样式透传。
- 更新作者文档，说明 `\LUchapter`、稳定习题 ID、todo、file answer、proof 和引用的推荐写法。
- 将旧生成文件从构建流程和清理规则中彻底移除。

每个阶段独立可回退，不把章节、答案和定理三类高风险改动塞进同一个不可审查提交。

## 六、总体验收标准

重构实现完成的最低标准如下：

1. 从干净工作区运行正式构建，主讲义与答案册均成功；第二次运行不再要求 rerun。
2. 章节测试序列精确得到“第 1 讲、未竟专题一 `1ε`、未竟专题二 `1δ`、第 2 讲”；四个有编号顶层单元使结构 `c@chapter` 最终等于 4，而普通显示计数器最终等于 2。
3. 普通正文显示编号不因插入或删除未竟专题而改变；结构 counter 对每个有编号顶层单元恰递增一次，未竟专题全局中文序号和同章希腊 slot 各自按约定变化。
4. 主 PDF 与答案 PDF 日志中不存在重复 destination；所有内部链接和 `xr` 链接指向正确页面。
5. 目录与 PDF outline 中每个单元恰有一条记录，中文标题、Unicode 希腊编号和层级正确。
6. `\ref`、`\nameref`、`\autoref`、`\cref` 以及范围引用覆盖普通章、未竟专题、附录、节、公式和六类定理，输出与兼容约定一致。
7. 没有可见答案的单元不生成答案册标题页；显式 todo 按配置生成统一占位。
8. 主文档失败、答案记录损坏或载荷缺失时，不覆盖上一份有效 manifest，并以非零状态失败。
9. 不同 jobname 可并发构建；同一目标的竞争构建不会产生交叉或半截文件。
10. 普通 inline 答案保持宏、段落和标签语义；file 答案可正常包含 verbatim/listing。
11. 现有定理环境调用在兼容阶段无需批量人工修改，编号和标签键不变。
12. proof 在普通文本、display math、跨页和嵌套场景中 QED 恰好出现一次；solution 缺省无 QED。
13. 章节状态、答案 renderer 和 theorem 声明各有单一实现来源，不再存在两套通过历史 marker 或开放分组彼此模仿的逻辑。
14. frontmatter、backmatter 和星号章不递增结构/显示 counter；重复阶段命令不重置任何章节状态，非法阶段回退明确失败；每个编号附录只递增结构与 appendix 显示 counter，且不会把结构 `c@chapter` 清零。
15. 全部手写源码（包括类和模板）都不存在把裸 `\value{chapter}`、`\arabic{chapter}`、`\setcounter{chapter}`、`\addtocounter{chapter}` 或 `c@chapter` 当作普通显示编号的调用；确需读写结构值的统一模块代码均在静态检查白名单中。
16. 受支持 TeX Live 版本的测试全部通过；不支持的引擎在加载早期给出明确错误。
17. 无普通章、错误 phase、编号被抑制或希腊 slot 超界时，`\LUchapter` 都在改变任何 counter、当前记录或样式前失败；修正输入后可继续构建，不受半初始化状态污染。
18. frontmatter、mainmatter 首章之前、backmatter、星号章、appendix 入口空隙及首阶段的附录章都不会把习题误挂到上一个正文/LU 单元；不支持的附录答案以明确错误结束。
