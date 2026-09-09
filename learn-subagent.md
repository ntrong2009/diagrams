# Subagent — dàn bài

Nguồn: <https://code.claude.com/docs/en/sub-agents>

---

## 1. [Subagent UI run](https://code.claude.com/docs/en/context-window)

## 2. [Subagent scope](https://code.claude.com/docs/en/sub-agents#choose-the-subagent-scope)

## 3. [Lúc start thì load gì](https://code.claude.com/docs/en/sub-agents#what-loads-at-startup)

## 4. [Cách gọi subagent](https://code.claude.com/docs/en/sub-agents#invoke-subagents-explicitly)

## 5. [Cấu trúc subagent file](https://code.claude.com/docs/en/sub-agents#write-subagent-files)

### 5.1. [Cách chọn model](https://code.claude.com/docs/en/sub-agents#choose-a-model)

### 5.2. [`tools` / `disallowedTools`](https://code.claude.com/docs/en/sub-agents#available-tools) — **demo**

## 6. [Control capabilities](https://code.claude.com/docs/en/sub-agents#control-subagent-capabilities)

### 6.1. [Scope MCP servers to a subagent](https://code.claude.com/docs/en/sub-agents#scope-mcp-servers-to-a-subagent) — **demo**

**Đo thử trước khi tin** — `/context` trên session: schema MCP mặc định ở dạng **deferred**, context chỉ chứa tên tool, nạp theo yêu cầu qua tool search. Nên 89 MCP tool chỉ tốn **950 token**, không phải 34.9k. Con số 34.9k là chi phí _nếu_ nạp hết.

> Vậy "tiết kiệm context ở hội thoại chính" chỉ đúng khi bạn đặt `ENABLE_TOOL_SEARCH=false` (nạp hết upfront), hoặc khi trong lúc làm việc Claude phải nạp dần nhiều schema.

Giá trị còn lại của `mcpServers` — vẫn đáng dùng:

- **Vượt trần**: đây là cửa **duy nhất** để subagent có tool mà session chính không có. Mọi field khác chỉ cắt bớt.
- **Vòng đời**: server inline chỉ kết nối khi subagent chạy, ngắt khi xong.
- **Sạch cấu hình**: không phải nhồi server vào `.mcp.json` dùng chung.

#### Lệnh demo

Hai agent trong `.claude/agents/`, `tools: Read, mcp__demofs` y hệt nhau. Chỉ `demo-mcp-scoped` có thêm `mcpServers` khai báo inline server `demofs` (filesystem MCP, gốc `/tmp`).

```text
# 1. Agent CÓ mcpServers
@agent-demo-mcp-scoped report your tools

# 2. Agent KHÔNG có mcpServers — cùng dòng tools y hệt
@agent-demo-mcp-noscope report your tools

# 3. Session chính có thấy demofs không?
/mcp
```

Kỳ vọng:

1. `Read` + một loạt `mcp__demofs__*` → subagent có tool mà session chính **không** có
2. Chỉ `Read`, `TOTAL: 1` → `mcp__demofs` không resolve vì server không tồn tại
3. `demofs` **không** xuất hiện → session chính không tốn context cho schema của nó

Lưu ý: inline server trong `.claude/agents/` của project cần trust folder trước, nếu chưa thì Claude Code bỏ qua server và ghi lý do vào debug log (`--debug`).

### 6.2. [Permission mode](https://code.claude.com/docs/en/sub-agents#permission-modes) — **demo**

Chọn cách subagent xử lý khi cần xin phép dùng tool. 6 giá trị:

| Giá trị             | Hành vi                                    |
| ------------------- | ------------------------------------------ |
| `default`           | Hỏi                                        |
| `acceptEdits`       | Có phạm vi theo đường dẫn lúc start claude |
| `auto`              | Classifier duyệt                           |
| `dontAsk`           | Tự từ chối                                 |
| `bypassPermissions` | Bỏ qua hỏi, không phân biệt đường dẫn      |
| `plan`              | Read-only                                  |

**Kế thừa — chỉ nới lỏng được, không siết chặt được:**

- Không đặt → kế thừa session chính.
- Session chính đang hỏi từng bước (Manual / `default`) → subagent muốn tự do hơn: **được**
- Session chính đang cho tự do (`auto` / `acceptEdits` / `bypassPermissions`) → subagent muốn hỏi lại cho chắc: **không được**, cứ tự do theo session chính

#### Lệnh demo

```bash
@agent-demo-ask-permission run the demo
```

| Lượt | Main session | Subagent                   | Kết quả       | Ý nghĩa                               |
| ---- | ------------ | -------------------------- | ------------- | ------------------------------------- |
| 0    | Manual       | `default`                  | **Hỏi**       | Mốc so sánh                           |
| 1    | Manual (ask) | `bypassPermissions` (free) | **Không hỏi** | Nới lỏng → frontmatter **thắng**      |
| 2    | auto (free)  | `default` (ask)            | **Không hỏi** | Siết chặt → frontmatter **bị bỏ qua** |

### 6.3. [Preload skills](https://code.claude.com/docs/en/sub-agents#preload-skills-into-subagents)

`skills:` nạp **toàn bộ nội dung** skill vào context subagent lúc start. Lấy từ project / user / plugin / built-in — chỉ cần ghi tên.

```yaml
skills:
  - code-review
```

|              | Có `skills:`                | Không khai báo             |
| ------------ | --------------------------- | -------------------------- |
| Lúc start    | Nạp toàn bộ nội dung        | Chỉ thấy tên + description |
| Khi cần dùng | Có sẵn                      | Mất một lượt gọi `Skill`   |
| Context      | Tốn ngay, dù dùng hay không | Chỉ tốn khi thực sự cần    |

- Không khai báo thì subagent **vẫn tự gọi được** skill qua tool `Skill`.
- Chỉ nên preload khi chắc chắn cần mỗi lần chạy.
- Chặn hẳn: bỏ `Skill` khỏi `tools`, hoặc thêm vào `disallowedTools`.
- Không preload được skill có `disable-model-invocation: true`.

### 6.4. [Persistent memory](https://code.claude.com/docs/en/sub-agents#enable-persistent-memory) — **demo**

`memory:` cho subagent một **thư mục ghi chú** không mất khi session kết thúc. Lần sau chạy, nó đọc lại được.

```yaml
memory: project
```

| Scope     | Vị trí                               | Dùng khi                                 |
| --------- | ------------------------------------ | ---------------------------------------- |
| `user`    | `~/.claude/agent-memory/<name>/`     | Nhớ xuyên mọi project                    |
| `project` | `.claude/agent-memory/<name>/`       | Theo project, commit được — **nên dùng** |
| `local`   | `.claude/agent-memory-local/<name>/` | Theo project, không commit               |

Hai điều cần biết:

- **Bật memory thì agent tự có `Write` và `Edit`** (để nó ghi file nhớ) → agent bạn tưởng read-only sẽ không còn read-only. Cùng loại bẫy với permission mode và `disallowedTools`.
- **Nó không tự học.** Phải nói "xem memory trước khi làm" và "xong thì lưu lại" — hoặc viết sẵn câu đó trong body file agent.

#### Lệnh demo

Agent `demo-memory` (`memory: project`). Mỗi lần chạy: đọc memory, báo lại, rồi ghi thêm một dòng `- run <n> · <ngày> · <fact>`.

```text
# 1. Lần đầu — memory rỗng
@agent-demo-memory run the memory demo

# 2. Kiểm chứng trên đĩa
!cat .claude/agent-memory/demo-memory/MEMORY.md

# 3. Lần hai — nó phải nhớ được run 1
@agent-demo-memory run the memory demo
```

Kỳ vọng:

1. Báo `MEMORY EMPTY`, rồi ghi `run 1`
2. File tồn tại ở `.claude/agent-memory/demo-memory/MEMORY.md` với đúng dòng đó
3. Báo lại được nội dung `run 1`, rồi ghi thêm `run 2`

Bước 3 là điểm cốt lõi: instance mới, context trắng, nhưng vẫn biết chuyện của lần chạy trước — vì `MEMORY.md` được nạp vào system prompt lúc start.

> Agent này chỉ khai báo `tools: Read, Bash` mà vẫn ghi được file, vì `memory` tự cấp thêm `Write` và `Edit`. Đúng cái bẫy nói ở trên.

Phụ thuộc auto memory: tắt `autoMemoryEnabled` hoặc `CLAUDE_CODE_DISABLE_AUTO_MEMORY` thì field này vô hiệu. Chỉ 200 dòng đầu (hoặc 25KB) của `MEMORY.md` được nạp.

### 6.5. [Hooks in frontmatter](https://code.claude.com/docs/en/sub-agents#hooks-in-subagent-frontmatter) — **demo**

`hooks:` khai ngay trong frontmatter → chạy **chỉ khi subagent đó active**, xong là dọn. Dùng `PreToolUse` để đặt **luật có điều kiện**: cho dùng tool nhưng chặn từng thao tác cụ thể — khác `tools`/`disallowedTools` vốn chỉ cho/cấm **cứng** cả tool.

Event hay dùng trong frontmatter:

| Event         | Matcher   | Fire khi                                       |
| ------------- | --------- | ---------------------------------------------- |
| `PreToolUse`  | tên tool  | Trước khi subagent gọi tool                    |
| `PostToolUse` | tên tool  | Sau khi gọi tool                               |
| `Stop`        | (không)   | Khi subagent xong (runtime đổi thành `SubagentStop`) |

```yaml
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "$CLAUDE_PROJECT_DIR/.claude/scripts/validate-no-delete.sh"
```

Cơ chế: Claude Code truyền input **JSON qua stdin** cho script. **`exit 0`** = cho qua; **`exit 2`** = chặn, đẩy message ở stderr về subagent.

Vài điểm cần nhớ:

- **Frontmatter vs `settings.json`**: hook frontmatter chỉ sống trong subagent; hook `settings.json` là toàn session và cũng fire bên trong subagent. Cả hai cùng áp lên một tool call.
- **Trust**: hook frontmatter của agent **project-level** cần trust folder trước; chưa trust thì subagent vẫn chạy nhưng **bỏ qua hook** và ghi lý do vào debug log. Agent user-level (`~/.claude/agents/`), `--agents`, và hook trong `settings.json` **không** cần trust. (Chặt hơn hook settings: trust folder cha không đủ, phiên `-p` không tính là trusted.)

#### Lệnh demo

Agent `demo-block-delete` (`tools: Bash`, có `PreToolUse` hook trỏ tới `validate-no-delete.sh`). Nó cố chạy `rm` xóa `demo-ask-permission.md`; script thấy `rm` → `exit 2`.

```text
# 1. Chạy — hook phải chặn
@agent-demo-block-delete run the delete-block demo

# 2. Kiểm chứng file còn sống
!ls gridsz-frontend/.claude/agents/demo-ask-permission.md
```

Kỳ vọng:

1. Lệnh `rm` **bị hook chặn** (`Blocked by hook: ...`), không kịp xóa
2. File `demo-ask-permission.md` **vẫn còn**

> Bỏ hook đi thì mất lớp chặn tự động — nhưng `rm` vẫn qua tầng **permission** (`default` → vẫn hỏi). Muốn "bỏ hook là xóa luôn, không hỏi" phải thêm `permissionMode: bypassPermissions`.

## 7. [Resume subagent](https://code.claude.com/docs/en/sub-agents#resume-subagents)

## 8. [Fork trong subagent](https://code.claude.com/docs/en/sub-agents#fork-the-current-conversation)

### 8.1. `/subtask` vs `/fork`

Khác biệt chính **không phải** worktree, mà là: `/subtask` tạo **subagent trong session này**, `/fork` tạo **một session riêng**. Worktree chỉ là hệ quả — mọi background session đều tự chuyển vào worktree trước khi sửa file.

|                 | `/subtask`                                                     | `/fork`                                                  |
| --------------- | -------------------------------------------------------------- | -------------------------------------------------------- |
| Là gì           | Subagent                                                       | **Session riêng**                                        |
| Kế thừa context | ✅                                                             | ✅                                                       |
| Kết quả         | Về hội thoại của bạn như một message                           | **Không về gì** — phải tự mở xem                         |
| Sửa file ở đâu  | Thẳng checkout của bạn                                         | Worktree riêng                                           |
| Dùng khi nào    | Việc phụ cần ngữ cảnh hiện tại, làm xong là quay về mạch chính | Việc dài/độc lập, chạy song song mà không chiếm terminal |

> **Cách nhớ:** `/subtask` là **nhánh con** báo cáo về cho bạn. `/fork` là **bản sao** tự đi làm việc riêng.

**Lưu ý phiên bản:**

- `v2.1.161`–`v2.1.211`: `/fork` chính là lệnh tạo forked subagent.
- Từ `v2.1.212`: lệnh đó đổi tên thành `/subtask`, còn `/fork` được dùng lại cho việc copy session.
- Nếu agent view bị tắt: `/fork` quay về nghĩa cũ và `/subtask` không khả dụng.

## 9. Non-fork vs fork subagent

> "non-fork" chỉ có một nghĩa: **context trắng**. Cứ thấy kế thừa context thì đó là fork.

| Thuật ngữ                  | Là gì                                  | Context         | Tạo bằng                                                          |
| -------------------------- | -------------------------------------- | --------------- | ----------------------------------------------------------------- |
| **Non-fork subagent**      | Subagent tạo từ file định nghĩa        | Trắng, cô lập   | `@agent-<name>`, Claude tự delegate, Explore/Plan/general-purpose |
| **Fork (forked subagent)** | Subagent kế thừa hội thoại             | Toàn bộ history | `/subtask`, hoặc Claude spawn với `subagent_type: "fork"`         |
| **Session copy**           | Không phải subagent — là session riêng | Toàn bộ history | `/fork`                                                           |

**Sai lầm thường gặp:** gọi `/subtask` là non-fork. Nó chính là fork. Còn `/fork` thì thuật ngữ fork/non-fork không áp dụng, vì nó không phải subagent.

## 10. `hooks` · `isolation: worktree` — để cuối cùng mới đụng
