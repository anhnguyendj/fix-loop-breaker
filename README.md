# fix-loop-breaker

**Thoát vòng "bảo AI sửa mãi mà lỗi vẫn còn".** Một skill cho Claude Code, Codex và các AI agent khác.

Khi AI sửa một lỗi cả ngày không xong, nó thường đang đoán nguyên nhân rồi vá luôn, không đo. Vá sai thì vá
thêm lớp nữa, code càng rối, lần đoán sau càng sai. Skill này bắt AI **dừng vá** và đi đủ 6 bước theo đúng
thứ tự, mỗi bước có cổng chặn:

```
0 Tái hiện ─► 1 Ba lăng kính ─► 2 Hai AI ─► 3 Spec ─► 4 Sơ đồ mới + code ─► 5 Kiểm thật
                   ▲                                                          │
                   └──────────── vẫn hỏng: quay lại với sự thật mới ──────────┘
```

| Bước | Làm gì | Cổng chặn |
|---|---|---|
| 0. Tái hiện | Gom lỗi nguyên văn, lệnh tái hiện, đầu vào, môi trường, bản còn chạy được | Có một lệnh chạy ra đúng lỗi |
| 1. Ba lăng kính | Sơ đồ luồng **hiện tại** có số đo · tách sự thật / giả định / ràng buộc · tìm chỗ tắc thật · Hỏi lại → Xoá → Đơn giản → Tăng tốc → Tự động | Viết được 3 dòng: giả định chưa kiểm, chỗ tắc + số đo, bước xoá được + thử nghiệm < 15 phút |
| 2. Hai AI | Đưa sự thật (không đưa lịch sử chat) cho một AI thứ hai, chỉ đọc. Lệch nhau thì thử nghiệm phân xử | Có phương án được chọn kèm lý do |
| 3. Spec | Viết lại đặc tả: mục tiêu, đầu vào/ra, đã xoá gì, điều kiện nghiệm thu đo được | Nghiệm thu chứa ca lỗi của bước 0 |
| 4. Sơ đồ mới + code | Vẽ luồng mới cạnh luồng cũ, lưu trạng thái, viết lại theo spec thay vì vá | Code khớp sơ đồ |
| 5. Kiểm thật | Chạy lại ca lỗi · đối chứng âm (bản cũ phải hỏng) · chạy đầu vào thật · người nhìn kết quả | Hỏng thì quay lại bước 1, đạt thì ghi bài học |

## Khi nào nó tự chạy

- Bạn báo có lỗi: "lỗi rồi", "tool lỗi", "fix lỗi", "không chạy", "ra sai", "error", "bug"...
  Lỗi vặt thì bước 0 thường đã lộ nguyên nhân, sửa luôn rồi sang bước 5 kiểm, không phải đi đủ 6 bước.
- Sửa cùng một chỗ lần thứ 2 cho cùng một loại lỗi
- Một lỗi đã quá 60 phút
- Bạn nói "sửa mãi không được", "luẩn quẩn", "lại lỗi đó", "stuck", "going in circles"...

Gọi tay: gõ `/fix-loop-breaker` trong Claude Code.

## Cài đặt

**Claude Code** (máy cá nhân, dùng cho mọi dự án):

```bash
git clone https://github.com/anhnguyendj/fix-loop-breaker ~/.claude/skills/fix-loop-breaker
```

Chỉ cho một dự án: clone vào `<dự án>/.claude/skills/fix-loop-breaker`.

**Codex CLI / Cursor / agent khác:** clone về rồi thêm vào `AGENTS.md` (hoặc file luật của agent):

```
Khi sửa cùng một chỗ lần thứ 2, một lỗi quá 60 phút, hoặc người dùng nói "luẩn quẩn":
đọc và làm theo <đường dẫn>/fix-loop-breaker/SKILL.md
```

**Claude trên web / app:** Settings → Capabilities → Skills → tải lên thư mục này dạng `.zip`.

### Bước 2 cần một AI thứ hai: Codex

Chưa cài Codex cũng không sao: skill tự kiểm máy, giải thích, xin phép rồi hướng dẫn cài từng bước. Muốn cài
trước thì làm thế này (cần Node.js 18.18+ và tài khoản ChatGPT, gói Free cũng được):

```bash
npm install -g @openai/codex
codex login
```

Thêm plugin Codex cho Claude Code (gõ trong Claude Code):

```
/plugin marketplace add openai/codex-plugin-cc
/plugin install codex@openai-codex
/reload-plugins
/codex:setup
```

Không cài được thì skill dùng đường dự phòng: một phiên Claude mới chỉ đọc sự thật, hoặc dán câu hỏi vào
ChatGPT, Gemini ở cửa sổ khác.

### Skill tự tốt lên sau mỗi lần dùng

Cuối `SKILL.md` có **Nhật ký ca** và **Sổ sửa sai**. Sau mỗi lỗi, agent tự chấm và ghi lại; bẫy lặp từ 2 lần
trở lên thì nó sửa thẳng vào skill. Dùng càng nhiều, skill càng khớp với cách bạn làm việc.

---

## English

**Break out of the "AI keeps fixing it, the bug is still there" loop.** A skill for Claude Code, Codex and other
coding agents. When it triggers (same spot fixed a 2nd time, a bug older than 60 minutes, or you say you're going
in circles), the agent stops patching and walks six gated steps: reproduce and collect facts → three lenses
(first principles, find the real bottleneck with numbers, then Question → Delete → Simplify → Accelerate →
Automate) with a diagram of the current flow → a second AI proposes independently, disagreements settled by a
<15-minute experiment → rewrite the spec with measurable acceptance → draw the new flow, then code to the spec
→ verify for real (re-run the original failing case, negative control on the old version, human looks at the
result). Still broken → back to step 1 with the new facts. The skill text is in Vietnamese; agents follow it
fine in any language.

Install for Claude Code: `git clone https://github.com/anhnguyendj/fix-loop-breaker ~/.claude/skills/fix-loop-breaker`

## License

MIT
