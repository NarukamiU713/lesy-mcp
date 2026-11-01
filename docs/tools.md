# 工具参考

每个工具需要 Token 的一个权限（见 [Token 和权限](tokens.md)）。每个工具都有对应的 `lesy` 命令（`lesy call <tool> '<json>'` 可以直接调用任意工具）；MCP 客户端通过 `tools/list` 能看到完整的 JSON Schema。这里说明它们的含义和规则。

## 通用规则

- **结果**：`structuredContent` 加一份文字。
- **大小上限**：JSON 超过 40 000 个字符时，截短最长的顶层数组并标出 `truncated: true`。
- **分页**：列表工具用 `limit` 和 `offset`。`limit` 默认 50、最大 200；`list_flashcards` 例外，默认 20、最大 100。
- **错误**：可预见的失败返回 `isError: true` 和 `structuredContent: { error, message }`；参数不符合 schema 时也是 `isError` 的结果；写工具的意外错误返回 `internal_error`，不带内部信息。
- **时间段**：用 `from`、`to`（`YYYY-MM-DD`）或 `period`，同时给时 `period` 优先。最长 366 天，更长时只读最近的部分。`period` 可以是 `today`、`yesterday`、`this_week`（从周一起）、`last_week`、`last_7_days`、`last_30_days`、`this_month`（从 1 号起）、`last_month`、`this_year`。

## 只读工具

### `list_articles`（`articles`）

命令行：`lesy articles [--limit N] [--offset N]`

没有删除的文章，最近更新的在前，不含正文。

- 参数：`{ limit?, offset? }`
- 返回：`{ articles: [{ id, title, tags, tagsAuto, createdAt, updatedAt }], total, offset }`

### `get_article`（`articles`）

命令行：`lesy article <id> [--offset N] [--max-chars N]`

一篇文章的元数据、正文总长度 `length`，以及从 `offset` 开始的一页正文。

- 参数：`{ id, offset?, maxChars? }`。每页默认 12 000 个字符，最多 30 000。
- 继续读：用返回的 `nextOffset`，读完时它是 `null`。
- 错误：`not_found`（文章不存在或已删除）、`invalid_offset`（`offset` 超过正文长度）。

### `learning_history`（`records`）

命令行：`lesy history [--period P | --from YYYY-MM-DD [--to YYYY-MM-DD]]`

某一天或某段时间记录的表达和备注，默认最近 7 天。

- 参数：`{ from?, to?, period? }`
- 错误：`invalid_date`。
- 只包含记录过的表达，没有浏览器里的查词记录。

### `search_records`（`records`）

命令行：`lesy search <query...>`

在记录过的表达中查一个词或表达，也按原形查。

- 参数：`{ query }`，1–200 个字符。

### `review_quiz`（`records`）

命令行：`lesy quiz [--period P | --from … [--to …]] [--count N]`

用记录生成复习卡片和答案，默认最近 7 天；这段时间没有记录时用最新的记录。

- 参数：`{ from?, to?, period?, count? }`，`count` 是 4–8。
- 错误：`invalid_date`。
- 卡片不保存，也不显示给读者。结果里有答案，agent 应该自己一题一题地问，不要一开始就给出答案。

### `list_flashcards`（`records`）

命令行：`lesy flashcards [--limit N] [--offset N]`

“Meine Karteikarten”：文章的卡片组和学习助手回答里的复习卡片，按卡片组分页，新的在前。卡片的文字截到 1000 个字符。

- 参数：`{ limit?, offset? }`

### `get_definition`（`records`）

命令行：`lesy definition <word>`

读者自己的释义（“Eigene Definition”），也找大小写不同的同一个词。

- 参数：`{ word }`
- 返回：没有时是 `{ definition: null }`，不算错误。

### `lookup_word`（`dictionary`）

命令行：`lesy lookup <word> [--context TEXT]`

查词典，和在文章中点这个词一样。返回原形、读法、候选和最多 8 个词条（词典、词头、摘要，Markdown 正文截到 2000 个字符）。

- 参数：`{ word, context? }`。`word` 最多 120 个字符；`context` 是这个词所在的句子，最多 500 个字符。
- 错误：`lookup_failed`（词典出错）。

## 写工具

写工具需要 `write` 权限。它们只新增或修改记录，**从不删除**任何东西，每次写入都在一个事务里。

应该只在读者明确要求时才写入。

### `set_definition`

命令行：`lesy set-definition <word> --meaning TEXT [--part-of-speech TEXT] [--notes TEXT]`

写入读者自己的释义。同一个词（区分大小写）已有的释义**整条覆盖**，省略的 `partOfSpeech`、`notes` 变成空字符串。

- 参数：`{ word, meaning, partOfSpeech?, notes? }`
- 返回：`{ definition, previous }`。`previous` 是原来的释义，没有时是 `null`；可以用同一个工具恢复。

### `set_expression_note`

命令行：`lesy set-note <articleId> <expressionId> <note>`

只改一条记录的备注。

- 参数：`{ articleId, expressionId, note }`。`note` 最多 2000 个字符，可以为空。
- 返回：`{ record, previous }`
- 错误：`not_found`。

浏览器里还没保存的面板修改会在下次保存时覆盖这条备注。

### `save_flashcards`

命令行：`lesy save-flashcards <articleId> --file <cards.json|->`

**只新建**一篇文章的卡片组，已经有卡片组时什么也不改。

- 参数：`{ articleId, flashcards: [{ kind, front, back, hint?, expressionId? }] }`
- 返回：`{ deck }`
- 错误：`deck_exists`（已经有卡片组）、`not_found`（文章不存在或已删除）、`invalid_expression`（`expressionId` 不是这篇文章的有效记录）。

卡片的规则：

- 3–12 张。
- `kind` 是 `meaning`、`cloze`、`translate`、`recall`、`comprehension` 之一。
- `front`、`back` 1–300 个字符，`hint` 最多 200 个字符；都是纯文本。
- `cloze` 的 `front` 必须有 `___`。
- `front` 不能包含答案，也不能重复。

## annotations

| 工具                                    | `readOnlyHint` | `destructiveHint` | `idempotentHint` |
| --------------------------------------- | -------------- | ----------------- | ---------------- |
| 只读工具                                | `true`         | `false`           | `true`           |
| `set_definition`、`set_expression_note` | `false`        | `true`            | `true`           |
| `save_flashcards`                       | `false`        | `false`           | `false`          |

所有工具的 `openWorldHint` 都是 `false`。`destructiveHint: true` 表示会覆盖旧值，不表示会删除。
