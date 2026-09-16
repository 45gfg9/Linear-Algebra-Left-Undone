# LALU 模板重构方案

> 状态：本轮实施与迁移记录。本文同时记录已落地的内部重构和明确延后的提案；“后续提案”不构成本轮验收要求。

## 本轮范围与兼容边界

本轮以仓库正文（`讲义/专题/`、`讲义/历年卷/`、`讲义/其它/`）和最终 PDF 为兼容边界。`LALUbook.cls`、主文件 `讲义/线性代数荣誉课辅学讲义.tex`、答案模板 `讲义/LALU-answers.tex` 中仅用于支撑实现的 counter、helper、marker、生成文件协议和 href anchor **不是兼容接口**；只要正文无需改写、既有标签/引用语义和 PDF 可见行为不变，它们可以直接删除、改名或换成新 schema，不设过渡包装器。

本轮状态如下：

| 子系统 | 本轮状态 | 兼容要求 |
| --- | --- | --- |
| 定理与证明 | 已切换到集中声明的 tcolorbox theorem + amsthm proof；已删除 proof/solution 未使用的第二个原始 tcolorbox 选项 | 六类定理的双参数接口、标签前缀、分别计数、proof/solution 第一可选说明及可见 QED/盒子行为不变 |
| 正文/未竟专题章节状态 | 已切换到显式状态机并完成 fixture 与整书回归 | `\chapter`、`\LUchapter`、中文标题编号、“上一正文章节 + 希腊字母”的引用编号、目录和 PDF 可见版式不变；旧 counter alias、开放分组和 marker 无需保留 |
| 答案生成与答案册 reader | 已切换到带版本的语义 manifest、独立 reader 和 staging 发布 | 正文继续使用普通 `answer` 环境；答案项、空答案、无答案单元及其既有分页在本轮保持可见行为不变；旧生成文件语法与路径无需兼容 |
| 稳定题目 ID、file payload、空单元省略策略及其余条件性变化 | 延后 review | 本轮不要求“在前面插题后已有 ID 不变”，不引入需要正文改写的新答案接口，也不启用其他条件性 breaking change |

这里的“正文无需改写”只约束作者源文件的既有公共调用。干净重编译、删除旧辅助文件、更新 Makefile 以及同步修改上述三份 supporting structure 都属于本轮正常迁移，不算正文 breaking change。

## 目标与不变量

本次重构的目标不是改变讲义的编排，而是把目前依赖跨章节分组、动态控制序列标记和隐式文件副作用的实现，替换成可检查、可测试的显式状态模型。

必须保持以下用户可见约定：

- 普通正文继续使用 `\chapter`，显示为“第 N 讲”。
- 未竟专题继续使用 `\LUchapter{标题}`；标题编号显示为“未竟专题 X”，其中 `X` 是未竟专题的全局中文序号。
- 未竟专题的数字引用继续由“上一普通正文编号 + 希腊字母”组成。例如第一讲后的前两个未竟专题分别引用为 `1ε`、`1δ`。
- 未竟专题内的节、公式、图表和定理继续沿用该数字引用，例如 `1ε.2`。
- 正文和答案册中的既有标签键、引用结果、目录层级、内容、顺序及分页应保持；内部辅助文件格式可改变，但必须通过干净重编译得到一致产物。
- 所有 Hyperref destination 必须在单次产物中唯一、确定且只含适合机器处理的 ASCII 内容；显示编号不得直接充当可能冲突的锚点编号。旧 PDF 的原始 href 名称不属于本轮兼容承诺。

以下三个概念应始终分离：

| 概念 | 示例 | 用途 |
| --- | --- | --- |
| 标题显示编号 | `未竟专题十四` | 章标题、答案册单元标题、目录章条目 |
| 数字引用编号 | `23ε` | `\ref`、节/公式/定理的父编号 |
| 机器锚点编号 | `chapter.LU.14` | Hyperref destination（内部实现，可改名，不承诺旧 PDF 深链接） |

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
| `unit-id` | 当前 manifest 内唯一的逻辑单元标识 | `lu:14` |

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
| 内部 unfinished serial | 只在未竟专题开始时递增，表示“未竟专题 X”中的全局序号 | 是 |
| 内部 unfinished slot | 同一普通正文之后的未竟专题次序 | 通过希腊字母显示 |
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
2. 将当前普通正文计数快照为 `normal-base`，设置 `kind=unfinished`，并分别递增内部的全局 unfinished serial 与当前普通章的 unfinished slot。普通章入口已经把 slot 归零，因此这里不再覆盖 `normal-base` 后做含义不清的“是否同章”比较。
3. 调用同一个最终标准 `\chapter`；结构 `c@chapter` 再递增一次并执行全部标准 reset/hook。
4. `\thechapter` 在本次调用及其正文期间返回保存的 `normal-base + Greek(slot)`；章标题的中文序号则来自内部 unfinished serial。

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
| 答案单元 | 使用 manifest 内的顺序 unit ID | `LALUanswerunit.answer.answer-unit:29` |

锚点表示必须完全可展开并只含 ASCII。`\theHchapter` 同样按 `kind` 分派：普通章返回普通显示编号，未竟专题返回 `LU.<unfinished-serial>`，附录返回 `APP.<appendix-display>`，无编号状态在确需 chapter 型锚点时返回 `UNNUMBERED.<serial>`。它不能直接使用结构 `c@chapter`，否则在插入未竟专题后会改变普通章既有外链。`LUchapter` 只是标题流水号时使用 `\stepcounter`；真正承载紧随其后 `\label` 的标准 `chapter` 才使用 `\refstepcounter`，避免创建无用的第二锚点。

标准 `\chapter*` 若已由 Hyperref 使用自己的 `chapter*.<linkcounter>` 命名空间，就继续沿用该机制；`UNNUMBERED.<serial>` 只是其他包确实展开 `\theHchapter` 时的确定性后备值，不能再额外创建一个与标准星号章重叠的 destination。

机器锚点属于 supporting structure，本轮不为旧 PDF 深链接保留 destination 别名。实现可以继续使用 `chapter.LU.<serial>`，也可以改用其他唯一 ASCII 命名；验收只检查正文标签键和 `\ref`/`\nameref`/`\autoref`/`\cref` 的语义、链接目标页面以及无重复 destination。引用锚点改名后必须干净重建 `.aux`、`.toc`、`.out` 和答案册的 `xr` 数据。

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

新答案清单必须携带渲染所需的章节语义快照，不能让答案册根据“上一个 section 是什么”猜测 `normal-base` 或希腊 slot。`\LUsection`、对应 counter 和旧 marker 只存在于 supporting structure，可随新 reader 一次删除；干净构建生成的新文件只输出 `\LALUAnswerUnit` 等当前 schema 事件。

### 1.6 内部替换范围与源码影响

本轮不为 supporting structure 设置过渡期，迁移策略如下：

- `\chapter{...}`、`\LUchapter{...}` 以及现有可选短标题等作者接口不修改。
- `LUchapter`、`LUgreekchap`、`LUsection` counter alias、`\LUgroupsancheck`、`\@exist@LUchapter@...`、`\@exist@LUsection@...` 和跨章开放分组可直接删除，不同步、不警告，也不保留空操作包装器。
- 旧 `LALU-ans-contents.tex` 及其中的 `\LUsection`/手工 counter 协议是可丢弃构建产物；新 reader 不读取它。Makefile 负责先生成并验证当前 manifest，再编译答案册。
- 标签键和引用可见结果必须保持；原始 href 名称、内部 counter 值和生成文件事件名可以变化。切换后统一清理 `.aux`、`.toc`、`.out`、`.fdb_latexmk` 与旧答案产物并完整重编译。
- 用旧、新 PDF 的页面数、抽取文本、目录/outline、代表页面和链接目标作回归比较；这种对比是验证可见行为，不是要求同时维护两套生产 reader。

这一首选模型会有意改变裸 `\value{chapter}`、`\arabic{chapter}` 和直接读取 `c@chapter` 的内部语义：它们返回的是“第几个有编号顶层单元”，不再等于普通正文的显示编号。迁移前必须全仓库扫描这些用法，并根据意图替换为语义 API：

- 需要普通正文显示编号时使用 `\LALUNormalChapterNumber` 或对应 expl3 查询接口；
- 需要当前可见引用时使用 `\thechapter`/`\LALUCurrentReferenceNumber`；
- 需要判断普通章或未竟专题时读取 `kind`，不能比较计数器；
- 只有 reset、checkpoint 或底层诊断代码可以读取结构 `c@chapter`。

当前仓库扫描结果是：`\value{chapter}`、`\arabic{chapter}`、`\setcounter{chapter}`、`\addtocounter{chapter}` 或 `c@chapter` 这类 raw 模式只出现在主模板现有章节逻辑中；答案模板还会重定义 `\thechapter` 并根据章节状态组织答案，因此两份模板都必须迁移，而正文内容没有直接依赖这些 raw 模式。迁移后应对全部手写源码启用静态检查。检查只允许统一章节模块中为 reset、checkpoint 或底层断言而登记的少量结构用途白名单，并排除 `.aux` 等生成文件；不能仅检查正文而放过模板内部继续把 raw chapter 当显示号的代码。

不再提供“裸 `chapter` 值等于普通正文显示编号”的次选兼容方案。仓库扫描表明这种读取只属于 supporting structure；继续临时回退 `chapter` 会重新引入重复锚点和 hook 状态歧义。若未来新增作者级查询需求，应暴露语义 API，而不是把结构 counter 重新定义成公共接口。

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

本轮只要求使用本机 MacTeX 的 XeLaTeX 完成最小 fixture 和整书构建，不额外验证 Docker、其他 TeX Live 版本或 LuaLaTeX。日志检查必须把以下内容视为失败：

- `destination with the same identifier`；
- 未定义引用或多轮后仍要求 rerun；
- 由章节编号产生的 `Token not allowed in a PDF string`；
- 章节状态断言失败；
- 目录和 PDF outline 中出现重复章条目。

## 二、答案抽取器

### 2.1 单一源数据与记录模型

答案的权威源仍是作者编辑的源码；任何生成的 `.tex` 或 manifest 都是可删除、不可手工编辑的构建产物。本轮 manifest 包含：

- schema 版本和构建 job 身份；
- 渲染所需的章节语义快照（kind、normal base、unfinished serial/slot、unit ID 和标题）；
- 习题组序号及显示标签；
- 题目在组内的逻辑序号、显示标签和本次生成中唯一的内部 ID；
- 答案状态：`answered`、`todo` 或 `omitted`；
- 未展开的 inline-token 正文；
- 记录数和结束 sentinel，供发布脚本与 reader 校验。

本轮 schema 2 使用 manifest 内的连续顺序 ID：`answer-unit:N`、`answer-group:N`和 `answer-item:N`。它们只承诺在单份 manifest 中唯一且与事件顺序一致，因而同章多个 `exercise`、重复显示组号或同题多个 `answer` 也不会制造内部冲突。在前面插入题目后保持已有 ID 不变的“发布级稳定 ID”明确不在本轮范围，也不新增 `\exitem[id=...]` 等作者接口。该目标留待后续单独设计。

生成文件应是语义事件流，而不是预先写死 `\section`、`\setcounter`、`\subsection*` 等版式命令。例如 manifest 只允许调用：

```tex
\LALUAnswerManifestBegin{...}
\LALUAnswerUnit{...}
\LALUAnswerGroupBegin{...}
\LALUAnswerItem{...}{...}
\LALUAnswerGroupEnd{...}
\LALUAnswerManifestEnd{...}
```

每个事件前还有一条受限 ASCII marker，供发布脚本在不解析任意 TeX 正文的前提下校验顺序、ID 和计数。答案册 reader 再使用独立状态机校验同一事件流并渲染标题和列表。每个 item 单独占一个物理行，答案 body 作为不透明、未展开 token 载荷传递。

### 2.2 编号与普通/未竟映射

进入 `exercise` 时，抽取器必须从章节状态 API 取得快照，禁止再通过某个动态控制序列是否存在来判断普通章或未竟专题。

编号规则如下：

- `exgroup` 使用普通 `\stepcounter`；若组本身支持 `\label`，其 Hyperref ID 必须包含 unit ID 和组序号。
- 可选显式组号先设置逻辑值，再通过同一条组初始化路径生成标题和锚点；不能让显式组号分支跳过锚点或 reset。
- 答案题号来自与题目绑定的逻辑 exercise item，而不是在 `answer` 结束时临时读取固定的 `\theenumi`。
- 嵌套 enumerate 只影响答案正文内部编号，不得改变所关联的外层题号。
- 答案册使用 manifest 中的 `normal-base`、`unfinished-serial` 和 `unfinished-slot` 渲染单元，绝不根据前一个已输出答案单元推算。

本轮按 manifest 事件顺序自动生成内部 ID，显示单元/组/题号另作元数据保存，不对正文发迁移告警。显式语义 ID 若将来确有跨版本引用需求，再作为会改变作者写法的独立提案 review。

### 2.3 inline token 边界

本轮只保留现有 `\begin{answer}...\end{answer}` 的 inline-token 写法。答案可包含用于普通文字和数学排版的 LaTeX token、平衡花括号、段落和常规宏；生成器以未展开 token 保存，禁止使用 `\tl_to_str:n`、`x` 型完全展开或 `\scantokens`。

`answer` 明确不支持 verbatim、minted、listing 或依赖特殊 catcode 的内容，本轮测试也不覆盖这些输入。这不是对当前正文的兼容性中断：仓库扫描没有此类答案，用户已确认可以把 inline answer 视为普通 LaTeX token。不要为不存在的特殊载荷编写外部正则解析器。

把答案人工拆到独立文件或新增 `\answerfile`/file payload 会改变作者写法，当前不实施。若以后确有需求，应先 review 是引入显式 file 接口，还是用其他方式从同一源代码生成两份 PDF；不能把特殊 catcode 支持悄悄塞进当前 inline 捕获器。

标题、题号等元数据与答案正文分开序列化。元数据需要 PDF 字符串时单独转换；不得为了生成元数据而展开正文。schema 2 刻意让每个 item 的命令及不透明 body 各占一个完整物理行，发布器只校验该行的固定外层结构，不展开或按行解析 body。

需要额外验证答案正文中的以下对象：

- `\label`、`\ref`、`\nameref` 和从正文导入的 `xr` 标签；
- 浮动体与图片相对路径；
- 脚注、索引项、局部宏；
- display math 末尾的 QED；

为避免答案册自身标签与 `\externaldocument` 导入的正文标签相撞，建议给外部标签添加固定前缀并提供 `\mainref`/`\mainnameref` 辅助命令；是否启用应先以标签清单验证兼容性。

### 2.4 空答案与占位语义

本轮以 PDF 可见行为不变为先，继续区分“没有 `answer` 环境”和“显式存在一个空 `answer` 环境”，并保留旧答案册已经排出的空答案项、无答案单元、目录条目和单元间 `\clearpage`：

- 每个 `exercise` 都输出答案单元记录，即使其中没有答案；reader 仍按原顺序排标题和分页。
- 含至少一个 `answer` 的组继续输出组标题；空的 `answer` 记为 `omitted`，但 reader 仍排出对应的空编号项。
- 非空答案记为 `answered`；内部 schema 可识别显式 `todo`，但本轮不要求正文迁移到 todo 写法。
- 日志报告总单元、组、题目、answered、todo 和 omitted 数量，以便和基线核对。

“省略没有可见答案的单元”“删除空答案项”或引入正式 todo 政策都会改变现有答案 PDF，均延后为需要单独 review 的条件性变化。

### 2.5 原子生成与失败恢复

TeX 主进程不应直接截断最后一个可用的正式答案文件。建议由构建辅助程序实施以下事务：

1. 在目标构建目录创建同一文件系统上的唯一临时目录或临时文件。
2. 主文档只向临时目标写 manifest 和 inline body。
3. 写入 schema header、记录总数以及结束 sentinel；关闭所有 stream。
4. 发布脚本先校验 schema/job、Begin/End 唯一性、marker 与 reader 命令对应、事件顺序、ID 连续唯一性和 unit/group/item/status 汇总计数。
5. 答案册直接读取这份临时 manifest，并在唯一 staging 目录中完整编译；reader 独立校验 schema/job、状态转移、ID 和实际计数，同时让 TeX 验证不透明答案 body 的可编译性。主 PDF 同样暂存在 ASCII jobname 输出中。
6. 只有主 PDF、答案 PDF 和 manifest 全部生成并验证成功后，Makefile 才集中将三者发布到正式路径；任一编译或发布前校验失败都保留上一份成功产物。manifest 使用同目录原子 rename，PDF 只在所有高风险步骤成功后才移入正式文件名。

纯 TeX 无法跨平台可靠地完成最终发布；临时路径、集中发布和 manifest 的同目录原子 rename 都由 Makefile 与小型发布脚本负责，不依赖 `\write18`。本轮 manifest 带 schema 版本和 job 身份；答案册发现缺失、不完整、事件顺序错误或版本/job 不匹配时必须失败，不能静默排版错误答案。源文件指纹与更强的过期检测可后续增加，不作为本轮完成条件。

### 2.6 并发、jobname 与构建目录

固定文件名 `LALU-ans-contents.tex` 不再作为内部协议。本轮 Makefile 使用显式 ASCII jobname 和独立构建目录，形如：

```text
build/answers/
  LALU.manifest.tex
```

主文档写入与正式 manifest 同目录的唯一临时文件；构建命令把该临时路径和 job 身份显式传给答案册，并把答案 PDF 编译到唯一 staging 目录。不同 jobname/工程隔离、同一目标并发锁、source hash 和变体身份属于后续增强，本轮不宣称支持并行写同一个发布目标。

`make clean` 删除已知答案构建目录和遗留生成文件；作者维护的正文不受影响。新生成内容不参与版本控制。

### 2.7 答案抽取器测试

至少建立以下 fixture：

- 普通章 A/B/C 三组，部分题有答案；
- 普通章后的两个连续未竟专题，验证 `Nε`、`Nδ` 与中文全局序号；
- 没有任何答案的 exercise、空组、空 answer、显式 todo；验证既有空项、单元和分页仍保留；
- 自定义起始组号和超过 9 的题号；
- 答案含多段、嵌套 enumerate、宏参数符号、注释、Unicode、标签和浮动体；
- 主编译在写到一半时故意失败，旧 manifest 仍可用；
- manifest 缺尾标、记录计数错误、schema/job 不匹配时发布器或 reader 明确失败；
- 新旧 PDF 的单元顺序、标题、组号、题号、答案正文、空项和分页逐项对比。

## 三、定理、证明与引用框架（本轮已实施）

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

仓库扫描未发现 `proof`/`solution` 的第二个原始 tcolorbox 选项用法；它只属于模板 supporting structure，已直接删除，不保留弃用包装器。第一可选说明仍是作者接口，继续支持 `\begin{proof}[说明]` 和 `\begin{solution}[说明]`。

### 3.4 `autoref` 与 `cleveref`

所有定理环境创建后，由同一声明表生成引用名称，不能散落手写私有 counter 名：

- 为 Hyperref 配置对应 `...autorefname`，保证 `\autoref` 输出“定理 3.2”等。
- 为 cleveref 同时配置 `\crefname`、`\Crefname`、range 和 multiple 格式。
- 现有标签前缀保持不变；`\ref`、`\autoref`、`\cref`、`\crefrange`、`\nameref` 都加入回归测试。
- 如果 tcolorbox 的内部 counter 名随版本变化，类应在一个内部适配层解析，正文不得依赖 `tcb@cnt@...`。
- 新正文优先推荐 `\cref`；`\autoref` 作为兼容接口继续受支持。

每个 theorem counter 的 `\theH...` 必须从统一的唯一 chapter anchor 派生。正常章和连续未竟专题中，同样显示为 `.1` 的定理也必须拥有不同 destination。

### 3.5 已实施的迁移

本轮实施结果如下：

1. 用 `amsthm` 替换 `ntheorem`，保留现有 tcolorbox theorem 公共环境名、双参数语法、标签前缀和分别计数方式。
2. 将六类 theorem 的名称、前缀、颜色、样式及 autoref/cleveref 名称集中声明。
3. 用 amsthm proof/QED 栈配合 tcolorbox 外观替换手写尾部方块；普通 proof、嵌套 proof、display/align 中的 `\qedhere` 已由最小 fixture 覆盖，solution 仍无 QED。
4. 删除无人使用的 proof/solution 第二可选原始样式参数；正文不需修改，第一可选说明保持。

整书审计已确认主讲义 892 页和答案册 272 页在 144 dpi 下逐页像素一致；主讲义 raw/layout 文本与基线一致，答案册 raw 文本一致。迁移要求干净重编译辅助文件；既有标签键、可见定理编号与默认 QED 版式均保持不变。

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

这里区分两类变化：

- **作者/PDF breaking change**：`讲义/专题/`、`讲义/历年卷/`、`讲义/其它/` 中的既有公共写法必须改写，或最终 PDF 的可见内容、编号、顺序、分页、标签/引用语义发生变化。本轮原则上不允许，除非下面明确列出并经 review。
- **supporting-structure breaking change**：只影响类、主模板、答案模板、Makefile、生成文件协议、内部 counter/helper/marker 或原始 href 名称。用户已确认这些无需向后兼容；只要正文和 PDF 可见行为不变，可以直接发生。

因此，清理辅助文件并冷构建是本轮迁移动作；它不意味着要为旧 manifest、旧 counter alias 或旧 PDF destination 增加兼容层。

### 4.1 本轮明确发生的内部兼容性中断

| 本轮变化 | 被移除/改变的 supporting structure | 正文受影响位置与新写法 |
| --- | --- | --- |
| 章节状态改为显式记录 | 裸 `chapter` 变为结构流水号；`LUchapter`、`LUgreekchap` counter alias 可删除；无编号状态不再把上一单元伪装成当前单元 | 仓库扫描只在主模板内部命中 raw counter；`讲义/专题/`、`讲义/历年卷/`、`讲义/其它/` 无需改写，仍用 `\chapter`/`\LUchapter` 和正常 `\label`。模板内部改读语义状态 API |
| 删除跨章开放分组和历史 marker | `\LUgroupsancheck`、`\LUsection`、`\@exist@LUchapter@...`、`\@exist@LUsection@...` 及配套 counter/marker 直接消失 | 命中只在主模板、答案模板和旧生成文件；正文各未竟专题开头的 `\LUchapter{...}` 不变，无需加入替代 helper |
| 答案交换格式与路径改为带版本 manifest | `LALU-ans-contents.tex` 的文件名、命令语法和直接 `\input` 约定不再支持 | 只修改主模板、`讲义/LALU-answers.tex`、`讲义/Makefile` 和发布脚本；所有正文继续原样写 `exercise`/`exgroup`/`answer` |
| 内部题目/单元 ID 与 href 命名空间改变 | 旧 manifest ID、`chapter.LU.*` 等旧 PDF 原始 destination 不承诺保留别名 | 正文 `\label` 键及 `\ref`/`\nameref`/`\autoref`/`\cref` 写法不变；只需清理辅助文件并重建，不需要逐篇修改 |
| 定理语义层从 ntheorem 切换为 amsthm | 类不再传递加载 ntheorem，其私有 style 命令不可用 | 正文未直接调用 ntheorem；六类 theorem、proof、solution 的保留接口不变，因此正文无需改写 |
| 删除 proof/solution 第二个原始 tcolorbox 选项 | `\begin{proof}[说明][任意样式]` 和 solution 同形内部签名不再接受第二项 | 全仓正文没有第二项调用；第一可选说明仍原样使用，无正文修改。若将来需要受控样式，应另行设计作者接口 |

这些变化无需弃用周期，因为删除对象没有出现在约定范围内的正文中。任何新命令一旦写入正文或作者文档，才转为需要稳定和迁移说明的作者接口。

### 4.2 条件性 breaking changes 与迁移审计

下列提案本轮**不启用**；若以后采用，必须另做正文扫描、PDF 基线和迁移 review：

- 如果答案册为 `\externaldocument` 加固定前缀，答案正文中指向主讲义的 `\ref`/`\nameref`/`\autoref`/`\cref` 要改为 `\mainref`/`\mainnameref`/`\mainautoref`/`\maincref` 等成套 helper。当前入口位于 `讲义/LALU-answers.tex`；启用前应先扫描所有 `answer` 载荷中的引用并验证没有同名标签。
- 如果以后要求“在前面插题后已有答案 ID 不变”，就需要语义 ID 或显式 `\exitem[id=...]` 一类设计；当前自动 ID 允许随位置变化，本轮不要求正文标注。
- amsthm proof 的公共语法和普通自动 QED 行为应保持兼容，但 QED 的具体位置需要视觉审计。当前有 27 个 proof 直接以 display 结束；典型位置为 `讲义/专题/9 矩阵运算进阶.tex:443`、`讲义/专题/13 多项式.tex:542` 和 `讲义/专题/21 线性代数与几何.tex:49`。只有希望方块留在最后一行公式内的用例才需加入 `\qedhere`，不能把 27 处一律机械改写；当前正文没有手写 `\square`。
- file payload/`\answerfile` 仍只是未来提案；当前 inline `answer` 明确只支持普通 LaTeX token，不支持 verbatim、minted、listing 或特殊 catcode，仓库正文没有此类载荷。
- 省略空答案项、完全无答案的单元、目录条目或分页，以及要求用显式 todo 保留它们，都会改变答案 PDF；本轮继续保留这些可见对象，不采用省略策略。
- 改变六类 theorem 的双参数语法、标签前缀、分别计数方式，或改 proof/solution 第一可选说明，都仍是条件性 breaking change，本轮不动。
- 支持引擎矩阵尚未最终确定；一旦决定不支持 LuaLaTeX 等引擎，这些非正式构建流程将改为早期失败。当前 Makefile 使用 XeLaTeX，不影响正文；若要把其他引擎列为受支持，须先补齐字体、索引、书签和回归测试。
- 如果无法可靠恢复 `\includeonly` 所需的非 counter 状态，可以明确禁止跨普通章/LU 的选择构建。当前主讲义使用 `\input`，因此仓库正文不受影响；外部使用者若依赖 `\includeonly`，应改为完整构建或等待 checkpoint 协议落地。

### 4.3 明确保留兼容的作者接口

以下常用正文写法不应因本次重构而修改：

- 普通章继续写 `\chapter{标题}`，未竟专题继续写 `\LUchapter{标题}`；新增的 `\LUchapter[短标题]{完整标题}` 只是可选扩展。
- 普通 inline 答案继续写 `\begin{answer} ... \end{answer}`；既有空 `answer` 也不需迁移。特殊 catcode 载荷不在该接口的支持范围内，本轮不引入替代语法。
- definition、example、lemma、theorem、corollary、axiom 继续保留当前双参数形式和标签前缀。
- `\begin{proof}[说明]` 与 `\begin{solution}[说明]` 的第一可选参数继续有效。
- 既有 `\label` 键及正常的 `\ref`、`\nameref`、`\autoref` 和 `\cref` 源码不要求批量改写；这些引用的可见结果和目标页面应保持。原始 href destination 字符串是内部细节，不在此承诺内。
- 空答案项、无答案单元、答案目录项及既有分页继续保留；正文不需要新增 `status=todo` 才能维持现有 PDF。

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

### 阶段 0：冻结基线（已完成）

- 保存当前完整 PDF、答案 PDF、`.aux` 标签表、`.toc`、PDF outline 和归一化日志。
- 记录主讲义和答案册的页面数、页面尺寸和抽取文本哈希。
- 扫描答案记录、空答案、定理环境调用签名、proof 可选参数及标签前缀。

基线只用于回归比较，不加入生产构建。

### 阶段 1：统一定理和证明框架（已实施）

- 移除 ntheorem 语义层，引入 amsthm proof/QED。
- 保持 tcolorbox theorem 的旧环境名、双参数语法、计数器行为和标签前缀。
- 集中 theorem 声明、样式及 autoref/cleveref 配置。
- 删除无人使用的 proof/solution 第二可选原始样式参数；保留第一可选说明。
- 用最小 fixture 验证普通、嵌套和 display/align `\qedhere`，再纳入整书 PDF 对比。

### 阶段 2：引入统一章节状态（已实施）

- 在主模板中加入结构 `chapter`、普通显示计数器、LU 流水号、slot、appendix 显示计数器以及统一章节记录。
- 提供语义查询 API，替换模板内部所有把裸 `chapter` 当普通显示编号的用法，并启用静态检查。
- 在既有 `\chapter`、`\LUchapter` 外观后切换到新状态机。
- 直接移除跨章节开放分组、旧 counter alias 和历史 marker，不保留兼容命令。
- 通过章节测试矩阵以及旧、新目录/outline/标签值/PDF 代表页面对比后完成。

### 阶段 3：答案 manifest 与原子发布（已实施）

- 定义带版本的 inline-token 事件 schema 和统一 `LALUanswerunit` renderer。
- 主编译写同目录唯一临时文件；发布脚本校验 schema、sentinel、严格事件顺序、ID 和统计。
- Makefile 将临时 manifest 路径和 job 身份传给答案册；reader 对缺失、版本/job 不匹配、事件顺序、ID 和计数错误明确失败。主 PDF 和答案 PDF 先留在 staging 路径，答案册成功后才与 manifest 集中发布，因此答案 body 的 TeX 错误不会提前覆盖任一份正式产物。
- 发布器负向 fixture 已覆盖 Begin/End 错位、marker/command 不匹配、行尾额外 TeX、未闭合参数、重复 footer、非连续 ID 和统计不一致，这些输入均以非零状态拒绝。
- 直接切换新 reader，不双写 legacy 文件，也不保留旧 `LALU-ans-contents.tex` reader。
- 比较新旧答案 PDF 的单元、标题、组号、题号、正文、空答案、无答案单元和分页。

### 阶段 4：整书验证与收尾（已完成）

- 在本机 MacTeX 上从 `make clean` 开始完整构建主讲义和答案册，并确认构建返回值能传播索引/manifest 失败。
- 检查 duplicate destination、undefined reference、PDF string warning 和残留 rerun 提示。
- 对照基线检查页面数、页面尺寸、抽取文本、目录/outline、标签表和全书像素；最终主讲义 892/892 页、答案册 272/272 页在 144 dpi 下均无像素差异。
- 清理构建产物，确认只保留本轮源码修改和用户原有的无关工作区改动。

### 后续 review（不属于本轮）

- 设计插题后仍稳定的显式习题 ID。
- 决定是否需要 file answer，以及它是否值得改变正文作者接口。
- 评估空答案/todo/空单元省略政策；任何改变先审核答案 PDF 的可见差异。
- 逐项 review 第 4.2 节的其余条件性 breaking changes。

定理/证明改动单独提交；章节记录与答案 manifest 共享同一语义 API，以一个始终可编译的原子提交落地，文档同步另作提交。

## 六、总体验收标准

重构实现完成的最低标准如下：

1. 从干净工作区运行正式构建，主讲义与答案册均成功；第二次运行不再要求 rerun。
2. 章节测试序列精确得到“第 1 讲、未竟专题一 `1ε`、未竟专题二 `1δ`、第 2 讲”；四个有编号顶层单元使结构 `c@chapter` 最终等于 4，而普通显示计数器最终等于 2。
3. 普通正文显示编号不因插入或删除未竟专题而改变；结构 counter 对每个有编号顶层单元恰递增一次，未竟专题全局中文序号和同章希腊 slot 各自按约定变化。
4. 主 PDF 与答案 PDF 日志中不存在重复 destination；所有内部链接和 `xr` 链接指向正确页面。
5. 目录与 PDF outline 中每个单元恰有一条记录，中文标题、Unicode 希腊编号和层级正确。
6. `\ref`、`\nameref`、`\autoref`、`\cref` 以及范围引用覆盖普通章、未竟专题、附录、节、公式和六类定理，输出与兼容约定一致。
7. 空 `answer` 仍生成原有空编号项；完全没有答案的单元仍保留原有标题、目录项和分页，答案 PDF 的可见顺序与基线一致。
8. 主文档失败或临时 manifest 的 schema、sentinel、统计校验失败时，不覆盖上一份有效的主 PDF、答案 PDF 或 manifest，并以非零状态失败；reader 对缺失、损坏或 job/version 不匹配明确失败。
9. 答案构建只读已结束且校验通过的临时 manifest，并将答案 PDF 写入 staging 目录；主编译、结构校验或答案 TeX 任一失败都不提前覆盖正式产物。全部成功后才集中移入两份 PDF 并以同目录原子 rename 替换 manifest；同一发布目标的并发构建不在本轮支持范围内。
10. 普通 inline 答案保持宏、段落和标签语义；测试边界明确排除 verbatim、minted、listing 和特殊 catcode。
11. 现有定理环境调用无需批量人工修改，编号、标签键、双参数接口和 proof/solution 第一可选说明不变。
12. proof 在普通文本、display math、跨页和嵌套场景中 QED 恰好出现一次；solution 缺省无 QED。
13. 章节状态、答案 renderer 和 theorem 声明各有单一实现来源，不再存在两套通过历史 marker 或开放分组彼此模仿的逻辑。
14. frontmatter、backmatter 和星号章不递增结构/显示 counter；重复阶段命令不重置任何章节状态，非法阶段回退明确失败；每个编号附录只递增结构与 appendix 显示 counter，且不会把结构 `c@chapter` 清零。
15. 全部手写源码（包括类和模板）都不存在把裸 `\value{chapter}`、`\arabic{chapter}`、`\setcounter{chapter}`、`\addtocounter{chapter}` 或 `c@chapter` 当作普通显示编号的调用；确需读写结构值的统一模块代码均在静态检查白名单中。
16. 本机 MacTeX/XeLaTeX 的最小 fixture 和完整 `make clean && make` 均通过；Docker、其他 TeX Live 版本和其他引擎不作为本轮验收项。
17. 无普通章、错误 phase、编号被抑制或希腊 slot 超界时，`\LUchapter` 都在改变任何 counter、当前记录或样式前失败；修正输入后可继续构建，不受半初始化状态污染。
18. frontmatter、mainmatter 首章之前、backmatter、星号章、appendix 入口空隙及首阶段的附录章都不会把习题误挂到上一个正文/LU 单元；不支持的附录答案以明确错误结束。
19. 主讲义和答案册的可见文本、章节/答案顺序、空答案项、无答案单元及分页与冻结基线一致；允许内部 counter、manifest 命令和 href destination 名称变化。
