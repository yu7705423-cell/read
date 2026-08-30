# 喵喵书阁(meow-reader.html)项目交接文档

## 这是什么

一个纯前端、单文件 HTML 的"伪阅读器"应用,用来存放 AIRP(AI角色扮演)过程中 user 和 char 之间产生的番外剧情、长篇小说,并附带一些配套工具(搜索、文件解析导入、语音朗读、AI总结前文)。

- **无后端**,所有数据存在浏览器的 **IndexedDB** 里(数据库名 `meowReaderDB`,四个 object store:`characters`、`books`、`materials`、`presets`)
- **无构建流程**,纯 vanilla JS + 原生 HTML/CSS,一个 `.html` 文件打开就能用
- 引用了三个 CDN 库(写死在 `<head>` 里):
  - `JSZip 3.10.1` — 解压/生成 zip
  - `mammoth 1.6.0`(browser 版)— 解析 docx 提取文本
  - `pako 2.1.0` — 解压 gzip(给 tar.gz 用)
- 当前文件体积约 115KB,全部逻辑都在一个 `<script>` 标签里

仓库里的文件是 `index.html`(通过 GitHub Pages 直接访问)。

---

## 原始需求 vs 完成情况

用户最初的需求(按对话原文整理),逐条标注完成度:

| 需求 | 状态 |
|---|---|
| 简约整体风格 | ✅ 已实现,自定义配色变量,非模板化视觉 |
| 四个Tab:书架/搜索/工具/设置 | ✅ |
| 内置明暗主题 + 自定义CSS导入 | ✅ 设置页里,粘贴CSS或传.css文件 |
| 书架加号:新建书籍 / 导入章节 | ✅ |
| 新建书籍:配置char/user姓名(留存)、封面、番外页/长篇选择、自定义标签、自定义分类 | ✅ |
| 支持多个char和多个user(不止一对一) | ✅ participants数组,每个含characterId+role |
| 已有char/user可复用,也可新建(放在"作者"概念下) | ✅ 角色库独立于书籍,可跨书复用 |
| 番外页 = 多条独立剧情;长篇 = 背景+多章节 | ✅ |
| 番外结构应为:番外要求(生成指令/主题) → 标题 → 正文 | ✅ 后期修正加上了`prompt`字段 |
| 番外/长篇中可能嵌入HTML/CSS,要能正常显示和点击 | ✅ sandboxed iframe渲染,支持整段是HTML或大段文字中夹小HTML块 |
| 搜索:关键字/章节名/角色名/番外剧情 + 分类(自定义/角色/番外剧情/标签) | ✅ |
| 工具:导入txt/解析zip/rar,一键复制 | ⚠️ txt/docx/zip/tar/tar.gz 都做了,**RAR 没做**(见下方说明) |
| ZIP要能解到嵌套最深层 | ✅ 递归解包,zip里套zip无限层都会挖到底 |
| 素材库要单独分类,不进书架体系 | ✅ 完全独立的 `materials` store |
| 素材库分文件夹 | ✅ 导入时选/建文件夹,单条素材可改归属文件夹 |
| 素材库支持多选批量复制/导出 | ✅ 多选模式,复制拼接文本 / 导出打包zip |
| 素材库支持直接手写文字新建 | ✅ 右上角"+"按钮 |
| 每本书自定义背景/翻页效果/注入CSS/自定义字体URL | ✅ |
| 翻页效果可选:竖屏滑动 或 左右翻页 | ✅ 已修复并实测通过(见下方"翻页模式") |
| 语音朗读配置(自填接口)、保存预设、绑定角色 | ✅ 预设增删改 + 实际调用出声音都已接通 |
| 一键复制(背景+前文)方便去续写 | ✅ 支持选择是否带上番外要求/背景 |
| 配置API总结前文(可自定义prompt) | ✅ 预设增删改 + 实际调用生成总结都已接通 |
| 朗读也可注入语音prompt让角色读更好 | ✅ `promptInjection` 会以 `{{prompt}}` 占位符注入请求体 |
| 整体设置可导出导入,便于分享 | ✅ 一键导出/导入全部数据的JSON |
| 全站不用emoji,改用SVG图标 | ✅ 用户中途明确要求后已经全部替换,已存入用户记忆偏好 |

---

## 数据结构

### `characters`(角色库,可被任意书复用)
```js
{ id, name, roleDefault: 'char'|'user', avatar, note, createdAt }
```

### `books`(书架条目)
```js
{
  id, title, cover, type: 'extra'|'novel',
  tags: [], category: '',
  participants: [{ characterId, name, avatar, role: 'char'|'user' }], // 支持多个
  createdAt, updatedAt,

  // type==='extra' 时:
  episodes: [{ id, title, prompt, content, createdAt }],
  // prompt = 番外要求/生成指令,content = 正文

  // type==='novel' 时:
  background: '',
  chapters: [{ id, title, content, order, createdAt }],

  readerSettings: {
    background: '',       // CSS背景值,或 `url(dataURL) center/cover no-repeat`
    pageEffect: 'scroll'|'paginate'|'fade'|'vertical'|'flip'|'instant',
    showPageNav: true,     // 分页样式下是否显示底部页码条,关掉就靠手势/点击/键盘
    customCss: '',
    fontUrl: '',           // 例如 Google Fonts 的 css2 链接
    voicePresetId: '',     // 关联到 presets 里 kind:'voice' 的记录(阅读设置里选)
    summaryPresetId: ''    // 关联到 presets 里 kind:'summary' 的记录(仅长篇,阅读设置里选)
  },

  // 生成过前文总结的长篇才有:
  summary: { text, updatedAt, forChapterId }
}
```

### `materials`(素材库,独立于书架)
```js
{ id, name, path, folder, content, importedAt }
// path 记录来源路径,比如嵌套解压时会是 "外层.zip / 内层.zip / 文件名.txt"
```

### `presets`(语音预设 + 总结预设,同一个 store,用 kind 区分)
```js
// 语音预设
{ id, kind:'voice', name, characterId, endpoint, apiKey, voiceId, extraParams(JSON字符串), promptInjection,
  bodyTemplate,     // 请求体 JSON 模板,占位符 {{text}} {{voice}} {{prompt}} {{apiKey}},留空用默认结构
  headersTemplate,  // 请求头 JSON 模板,支持 {{apiKey}},留空默认 Authorization: Bearer <key>
  audioPath,        // 响应里音频字段的点号路径,如 data.audio_url,留空则自动猜测
  chunkSize,        // 单次请求的字数上限,默认 1500,超出会按段落切分逐段合成
  provider,         // 用过哪个快捷配置(minimax/openai/elevenlabs/siliconflow/custom),拉音色列表时要用
  createdAt }

// 总结预设
{ id, kind:'summary', name, endpoint, apiKey, model, promptTemplate, provider, createdAt }
```

---

## 各模块实现要点

### 阅读器(第3块)
- 全屏 `.reader-overlay`,正文用 **sandboxed iframe**(`sandbox="allow-scripts"`)渲染,通过 `srcdoc` 注入生成的 HTML 文档,这样自定义CSS/字体/嵌入HTML都不会污染外层App样式,也能保证番外里嵌的HTML小组件正常显示和交互。
- 内容渲染逻辑(`renderContentHtml`):按空行分段落包成 `<p>`,但如果某一段本身以 `<` 开头(判断是完整HTML块),原样保留不做转义包裹——这样"大段文字里插一个小HTML卡片"和"整条内容就是一段HTML"两种情况都能正常显示。
- **工具栏可折叠**:顶部工具栏默认展开,点击顶栏的收起箭头、或点击正文空白处(iframe内部通过 `postMessage` 把点击事件传回父页面,排除了链接/按钮等交互元素的点击)都会让工具栏滑走,只留一个小箭头挂件在顶部中间,点一下再唤出来。翻页模式下的页码导航条同理跟随折叠。
- **一键复制**:番外可选是否带上"番外要求";长篇默认是"背景+从第一章到当前章"的正文,也可选是否带背景。用 `navigator.clipboard.writeText`,失败时降级用隐藏 textarea + `execCommand('copy')`。

### 关于 `<script>` 标签转义的坑(重要,继续改代码时必须注意)
`buildReaderDoc()` 函数会拼一段**字符串**作为 iframe 的 `srcdoc`,这段字符串里必然要包含 `<script>...</script>`。但因为这整个函数本身是写在外层 `.html` 文件的 `<script>` 标签**内部**的 JS 源码,如果字符串字面量里直接出现 `</script>`(哪怕只是JS字符串里的普通字符),**浏览器的HTML解析器会在扫描到这个字符序列时提前结束外层的 `<script>` 标签**,导致整个应用崩掉。

解决方法:在源码里凡是要拼出 `</script>` 的地方,必须写成 `<\/script>`(反斜杠转义),JS 求值后会变成正常的 `</script>` 字符串,但 HTML 解析阶段看到的是 `<`, `\`, `/` 这样不连续的字符,不会被误判为闭合标签。**以后改这部分代码,新增任何内嵌 `<script>` 都要用这个写法**,可以用这两条命令自查:
```bash
grep -c "</script>" meow-reader.html      # 应该恰好是 1(外层真正的闭合标签)
grep -c "<\\\\/script>" meow-reader.html   # 应该等于文件里内嵌script的数量
```

### 翻页模式(已修复,实测通过)

最早用的是 CSS `column-width` + `column-fill:auto` 让浏览器"高度不够时自动往右边生成新分栏"的特性做分页,这个特性表现不稳定,反复出现"只有第一页有内容,后面全空白"。现在改成 JS 精确测量,并修掉了三个真实存在的问题:

1. **上下留白的位置错了**。原本正文的 `padding:30px 22px 40px` 写在 `.inner` 上。多栏布局在**块方向**(上下)分片时,`padding-top` 只会给第一个分片、`padding-bottom` 只会给最后一个分片,所以从第2页起正文直接贴着屏幕顶端。现在上下 padding 挪到多栏容器 `#pages` 上(容器的 padding 会让每一栏都缩进),左右 padding 留在 `.inner`(**行内方向**的 padding 每个分片都会重复,本来就是对的)。
2. **留白高度不够,正文被工具栏和页码条盖住**。iframe 是铺满整个阅读页的,工具栏和底部页码条浮在它上面。现在由父页面用 `topbar.offsetHeight` / `pagenav.offsetHeight` 实测真实高度(已含 `env(safe-area-inset-*)`)传给 `buildReaderDoc(entry, book, insets)`,正文按这个高度留白。
3. **页数会算少,最后一段被裁掉**。`Math.ceil(总高度 / 屏幕高度)` 只是下限——行不能跨栏断开,实际需要的页数永远 ≥ 这个估算值。现在先用它当起点,套上多栏后实测 `.inner` 所有子元素的**最右边界**反推真实页数(放不下的内容会溢出到 declared width 之外的 "overflow columns",所以量得到),不一致就调整页数重来,最多迭代 6 轮。

另外补了 `document.fonts.ready` 之后重新测量,避免网络字体加载完成后分页错位。

代码位置:`buildReaderDoc()` 里 `isPaginate` 分支,以及生成的 iframe 内嵌 `<script>` 里的 `layout()` / `pagesUsed()` / `goTo()`。

### 翻页样式(六种)

`PAGE_EFFECTS` 数组定义,`isPaginatedEffect()` 判断是否属于"分页"类:

| id | 名称 | 做法 |
|---|---|---|
| `scroll` | 竖屏滑动 | 不分页,正常滚动 |
| `paginate` | 左右平移 | translateX 整条内容,唯一支持实时跟手拖动的样式 |
| `fade` | 淡入淡出 | 淡出 → 无动画跳到新页 → 淡入 |
| `vertical` | 上下翻页 | 纵向滑出屏幕 → 跳页 → 从另一侧纵向滑入 |
| `flip` | 立体翻转 | body 上加 perspective,rotateY 转到 ±88° 再转回来 |
| `instant` | 无动画 | 直接跳 |

**为什么没有"覆盖""仿真书页卷曲"**:这类效果要求同屏出现两页(新页压在旧页上),而当前只有一份正文 DOM。要做就得把正文复制一份到第二层,但 `innerHTML` 复制出来的副本里,正文中嵌入的 HTML 小组件会失去交互(脚本不会重新执行,事件监听也不会跟着走)——而"番外里嵌的 HTML 要能正常显示和点击"是这个项目的核心需求之一,不能为了动画牺牲。所以六种样式全部是对同一条内容做变换。

翻页动画进行中会置 `animating = true` 并忽略新的翻页请求,否则连点会让 `pageIndex` 和实际位移对不上。

### 翻页操作方式

- 左右滑动手势:`paginate` 样式下实时跟手(`DRAG_FOLLOWS`),首尾有阻尼回弹,松手不到阈值滑回原位;其余样式没有把相邻页渲染出来,跟不了手,所以按阈值触发。
- 鼠标拖拽(桌面端)走同一套 `dragStart/dragMove/dragEnd`。
- 点击屏幕左侧 30% / 右侧 30% 翻页,中间 40% 收起/展开工具栏。
- 键盘 ←/→ / PageUp / PageDown / 空格 / Home / End。
- 底部页码条可以在阅读设置里关掉(`readerSettings.showPageNav`)。关掉之后正文的底部留白会跟着缩回去(父页面传给 `buildReaderDoc` 的 `insets.bottom` 改成只留安全区),不会白空一条。

### 自定义 CSS 的类名

阅读页生成的每个元素都带 `mm-` 前缀的类名,方便用户写 CSS:

| 选择器 | 是什么 |
|---|---|
| `.mm-body` | 阅读页正文文档的 body |
| `.mm-page` | 翻页容器(分页时每一"页"就是它的一栏) |
| `.mm-content` | 正文内容区,左右留白在这上面 |
| `.mm-prompt` | 番外顶部的"番外要求"卡片 |
| `.mm-p` | 每一个正文段落 |
| `.mm-html` | 正文里嵌入的 HTML 块的外层容器 |

`CSS_HELP` 里维护了两份速查表(`reader` / `app`),`cssHelpPanel(scope)` 生成可展开面板,`bindCssHelp()` 绑定"点示例追加进输入框"。**以后改了类名或加了新的可定制元素,记得同步更新 `CSS_HELP`**,否则速查表会骗人。

### 浮层层级(已修复)

原来 `.modal-overlay` 是 90、`.reader-overlay` 是 70。这是为了让"从阅读页里打开的更多菜单/阅读设置/复制弹窗"能盖住阅读页,但代价是**从书籍详情进入阅读页之后,详情弹窗仍然盖在阅读页上面**——下半屏被挡住、翻页按钮完全点不到。

现在两种浮层共用一个动态层级:`nextOverlayZ()` 取当前所有浮层里最高的 z-index 再 +5,谁后开谁在上。于是阅读页盖住详情弹窗,阅读页里再开的弹窗又盖住阅读页,两个需求同时满足。toast 提到 9999 保证永远在最上层。

**以后新增任何全屏浮层,记得用 `nextOverlayZ()` 而不是在 CSS 里写死 z-index。**

## 第7、8块(已完成)

### 第7块:语音朗读

阅读页"更多"菜单 → **朗读正文**。取 `book.readerSettings.voicePresetId` 对应的预设(在"阅读设置"里选,没选会提示),把正文转成纯文本后按预设的 `chunkSize` 在段落/句子边界切块,逐块合成、连续播放,底部有个悬浮播放条(播放/暂停/停止 + 第几段进度)。

因为每家语音接口的字段名都不一样,请求和响应都做成可配置的:

- **请求体模板** `bodyTemplate`:用户自己写 JSON,占位符 `{{text}}` `{{voice}}` `{{prompt}}` `{{apiKey}}`。占位符是**填进已序列化的 JSON 字符串里**的,所以替换前会先 `JSON.stringify(值).slice(1,-1)` 转义,否则正文里一个引号或换行就会把模板打碎(`fillJsonTemplate()`)。填完会先 `JSON.parse` 验一遍,不合法直接报错并把填好的模板给用户看。留空则用默认结构 `{text, voice_id, prompt}` + 额外参数。
- **请求头模板** `headersTemplate`:同样支持 `{{apiKey}}`。留空默认发 `Authorization: Bearer <key>`。
- **响应解析**:先看 `Content-Type`,是 `audio/*` 或 `octet-stream` 就直接 `URL.createObjectURL(blob)`(用完会 revoke);否则按 JSON 解析,优先用用户填的 `audioPath`(点号路径,支持数组下标),没填就用 `findAudioValue()` 浅层遍历找 key 像 audio/url/speech 的长字符串兜底。拿到的值是链接、`data:` URI 还是裸 base64 都能认(`toAudioSrc()`)。
- **解析不出来就把原始响应整段显示出来**(`showApiErrorModal()`),用户可以照着调预设。`fetch` 本身抛错会提示大概率是 CORS。

一个坑:**Safari 只允许在用户手势里调用过 `play()` 的 `<audio>` 元素之后再用脚本播放**,而第一次合成要等好几个 await。所以 `startReadAloud()` 一进来就同步 `new Audio()` 并调一次 `play()` 把它"解锁",真正的异步流程放在里面的 async IIFE 里。

### 快捷配置与拉取列表

`TTS_PROVIDERS`(MiniMax / OpenAI 语音 / ElevenLabs / 硅基流动 / 自定义)和 `LLM_PROVIDERS`(OpenAI / DeepSeek / 硅基流动 / Kimi / OpenRouter / 自定义)是**模板**不是保证:点一下只会覆盖接口地址、请求头、请求体、响应字段这几个"传输"字段,用户已经填的名称/Key/音色不动,填完还能随便改。个别服务需要按账号补东西(MiniMax 要在地址里填自己的 GroupId),这一点写在了每个服务的 note 里。

- **接口地址也支持占位符**(`resolveEndpoint()`),因为 ElevenLabs 这类服务把音色 ID 拼在 URL 路径里。
- **MiniMax 返回的是十六进制音频**,不是 base64。`toAudioSrc()` 会判断:全 `[0-9a-f]`、偶数长度、超过 100 字符就当十六进制解码成 Blob(真实音频的 base64 几乎必然含有 hex 之外的字符,所以这个判断是安全的),否则才当 base64。
- **拉取音色列表** `fetchVoiceList()`:有固定音色的服务(MiniMax、OpenAI)直接用内置列表,不发请求;ElevenLabs 这类去 `voicesEndpoint` 拉。
- **拉取模型列表** `fetchModelList()`:OpenAI 兼容的服务都在 `/v1/chat/completions` 旁边有 `/v1/models`,`modelsUrlFor()` 从聊天地址推导出来。
- 两个列表都过 `normaliseList()`,能吃 `{voices:[{voice_id,name}]}`、`{data:[{id}]}`、`{a:{b:["x","y"]}}` 等形状;指定路径找不到就退化成"找第一个非空数组"。
- 拉到的列表填进一个 `<select>`,选中会写回上面的文本输入框——文本框始终保留,因为克隆音色、私有模型这些 ID 是列表里没有的。

### 第8块:AI总结前文

只对长篇生效,阅读页"更多"菜单 → **前文总结**。取 `summaryPresetId` 对应的预设,把 背景 + 第一章到当前章的正文 发过去,用 OpenAI Chat Completions 格式:

```js
{ model: preset.model, messages: [
    { role:'system', content: preset.promptTemplate || 默认总结指令 },
    { role:'user', content: 组装好的正文 }
]}
```

响应解析 `pickSummaryText()` 依次尝试 OpenAI(`choices[0].message.content`)、Anthropic(`content[]` 里的 text 块)、Gemini(`candidates[0].content.parts`)等常见格式,都不行就把原始响应展示出来。

结果存在 `book.summary = { text, updatedAt, forChapterId }`,弹窗里可以手动改、复制、删除,也可以完全不调接口自己手写。

**一键复制**里,长篇只要有提要就多一个开关「用前情提要代替前面章节的正文」,勾选后复制的是 背景 + 提要 + **本章正文**(本章保留原文,因为续写最需要紧邻的上下文),这样写到几十章也不会一次复制出几十万字。

## 其他注意事项

- **RAR 完全没做**:纯前端没有可靠的RAR解析方案(专有格式,只有实验性WASM移植,RAR5/固实压缩/分卷经常直接失败),已经跟用户确认过放弃。TAR/TAR.GZ 因为格式简单、纯JS就能解析,所以顺手做了。
- 所有输入框字号强制设为 `16px`,是为了避免 iOS Safari 在输入框字号小于16px时聚焦自动放大页面的问题(之前真实踩过这个坑,表现为"主页面一直在放大缩小拉伸")。
- 弹窗系统 `.modal-overlay` 的 `z-index` 必须高于阅读页 `.reader-overlay`,否则从阅读页内打开的"更多菜单/阅读设置/复制"弹窗会被阅读页盖住(之前的真实bug,现在 modal-overlay 是 90,reader-overlay 是 70)。
- FAB(书架加号按钮)的底部间距要用 `calc(78px + env(safe-area-inset-bottom))`,不能写死数字,否则在有Home Indicator的手机上会被底部导航栏遮住。

- **接口调用全部走浏览器 fetch**,所以对方接口必须允许跨域(CORS)。这是纯前端应用绕不过去的限制,已经在报错文案里写清楚了。
- 语音/总结的调试方式:仓库外可以起一个本地 mock 接口逐条验证请求体、请求头、响应解析和各种出错分支,比对着真实接口试要快得多。

祝接手顺利,喵喵书阁就交给你了 TvT
