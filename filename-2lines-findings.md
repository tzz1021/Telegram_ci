# 群/频道里文件名"最多两行"——链路调查

工作目录: `/home/tzz/Telegram_ci` (DrKLO/Telegram @ 12.10.3, versionCode 7089)

## TL;DR

1. **"60 字符"在 MTProto 协议层并不存在。** `documentAttributeFilename.file_name` 就是一个普通
   TL `string`（schema layer 229），线上读写只是 `readString`/`writeString`，协议没有任何 60（或其它）
   长度上限。
2. **"最多两行"是纯 UI 行为**，而且是一个**硬编码的 `maxLines = 2`**：
   `ChatMessageCell.createDocumentLayout()` → `StaticLayoutEx.createStaticLayout(..., maxLines=2, ...)`。
   超过 2 行时会被截断并补一个 `…`。
3. **~60 字符只是"2 行的自然容量"**：气泡里文件名区域最大宽度 `dp(270)`、字号 `dp(15)` 粗体，
   平均每行 ~30 字符，2 行正好 ~60 字符。所以"不到 60 字符也被卡在两行"其实是
   "2 行 × ~30 字 ≈ 60 字"的巧合，不是任何协议/常量限制。
4. 你给的三个疑点（`org/telegram/ui/recyclerview`、`me/vkryl/android/ViewUtils.java`、
   `androidx/recyclerview/widget`）**都不是原因**：消息气泡是 `ChatMessageCell` 自己在 canvas 上手绘
   StaticLayout，根本不走 RecyclerView 的 TextView；`ViewUtils.java` 里也没有任何文本行数逻辑。

---

## 从服务器 → 协议 → 数据库 → UI 的"文件名"链路（只列与文件名相关的方法）

```
[服务器 / MTProto]
  messages.getHistory / messages.getMessages / updateNewMessage ...
        │  TLRPC.Message.media = messageMediaDocument{document}
        ▼
[协议解析  tgnet / TLRPC]
  TL_messageMediaDocument.readParams
    └─ TL_document.readParams
         └─ attributes[] → TL_documentAttributeFilename.readParams
              file_name = stream.readString(...)          TLRPC.java:28512
              stream.writeString(file_name)               TLRPC.java:28527
  schema: documentAttributeFilename{ file_name:string }   tlscheme/229.json  ← 无长度限制
        │
        ▼
[模型  model]
  MessageObject.getFileName() / getFileNameFast() / getFileName(TLRPC.Message)
                                                          MessageObject.java:7321/7328/7335
  FileLoader.getDocumentFileName(document)                FileLoader.java:1566
    └─ 优先用 document.file_name_fixed
       else 取 TL_documentAttributeFilename.file_name
       └─ FileLoader.fixFileName(fileName)  仅清洗非法字符，不截断   FileLoader.java:1559
        │
        ├──────────────────────────────► [持久化  database]
        │                                 MessagesStorage.putMessages(TLRPC.messages_Messages, ...)
        │                                                          MessagesStorage.java:16078
        │                                 MessageObject 序列化进 messages_v2(data BLOB)
        │                                                          table 定义 MessagesStorage.java:551
        │                                 ← 文件名内嵌在整条消息里，DB 层不做任何长度处理
        │
        ▼
[UI  消息气泡（群/频道里看到的就是它）]
  ChatMessageCell.createDocumentLayout(maxWidth, messageObject)   ChatMessageCell.java:12746
    name = FileLoader.getDocumentFileName(documentAttach)         ChatMessageCell.java:12849
    docTitleLayout = StaticLayoutEx.createStaticLayout(
        name, Theme.chat_docNamePaint, maxWidth, ...,
        TextUtils.TruncateAt.MIDDLE, maxWidth,
        maxLines = 2, false)                                      ChatMessageCell.java:12853  ★两行上限
    └─ StaticLayoutEx.createStaticLayout(...)                     StaticLayoutEx.java:56
         lineCount > maxLines ⇒ 截到第 maxLines 行 + 追加 "…"
   绘制: ChatMessageCell.drawContent() → docTitleLayout.draw(canvas)   ChatMessageCell.java:15106
```

气泡宽度来源（决定"每行多少字"）：
```
backgroundWidth = maxWidth = Math.min(
     getParentWidth() - dp(50 + ...),
     dp(270) )                                   ChatMessageCell.java:9067-9069
Theme.chat_docNamePaint.setTextSize(dp(15)); bold  Theme.java:8059/8382
```

---

## 关键证据

| 位置 | 内容 |
|---|---|
| `tlscheme/229.json` | `documentAttributeFilename{ file_name:string }` → 协议无 60 限制 |
| `TLRPC.java:28512` | `file_name = stream.readString(exception);` 线上就是个 string |
| `ChatMessageCell.java:12746` | `private int createDocumentLayout(int maxWidth, MessageObject messageObject)` |
| `ChatMessageCell.java:12853` | **`... maxLines = 2 ...`** ← 罪魁祸首（硬编码） |
| `StaticLayoutEx.java:56` | 复写 StaticLayout；`lineCount > maxLines` 时截断 + `"…"` |
| `ChatMessageCell.java:15106` | `docTitleLayout.draw(canvas)`（气泡内手绘） |
| `ChatMessageCell.java:9067-9069` | 文件名区最大宽度 `dp(270)` |
| `Theme.java:8382` | `chat_docNamePaint` 字号 `dp(15)` |

## 顺带：另一处"文件名两行"（如果你指的是"文件"Tab 列表而不是气泡）

`SharedDocumentCell`（群/频道资料页 → 文件 列表用的 cell）：
- `SharedDocumentCell.java:189` `nameTextView.setMaxLines(2)` —— 仅 `VIEW_TYPE_GLOBAL_SEARCH`（全局搜索）
- `SharedDocumentCell.java:202` `nameTextView.setMaxLines(1)` —— 默认/共享媒体/Picker 都是 **1 行**

如果用户看到的"两行"出现在文件列表里，那对应的是 global-search 分支；气泡里则是
`ChatMessageCell:12853` 的 `maxLines=2`。

## 结论

- 要放开"两行"，改 `ChatMessageCell.java:12853` 的 `maxLines`（以及 `docTitleLayout` 相关的高度计算
  `ChatMessageCell.java:8440-8441 / 9400-9401`，那里用 `(lineCount-1)*dp(16)` 算高度）即可。
- 协议/数据库层不需要动，那里根本没有长度限制。
