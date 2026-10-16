---
name: fix-loop-breaker
description: Thoát vòng "bảo AI sửa mãi mà lỗi vẫn còn". Dừng vá, rồi đi đủ 6 bước có cổng chặn - tái hiện lỗi và gom sự thật → 3 lăng kính (nguyên lý gốc, bóc cấu trúc, thứ tự Hỏi lại → Xoá → Đơn giản → Tăng tốc → Tự động) kèm sơ đồ luồng hiện tại → hai AI cùng tìm phương án → viết lại spec → sơ đồ luồng mới rồi mới code → kiểm thật bằng chính ca lỗi ban đầu. Dùng khi sửa cùng một chỗ lần thứ 2 cho cùng một loại lỗi, một lỗi đã quá 60 phút, hoặc người dùng nói "sửa mãi không được", "fix cả ngày vẫn lỗi", "luẩn quẩn", "lại lỗi đó", "lâu thế", "stuck", "same error again", "going in circles", "AI keeps breaking it".
---

# fix-loop-breaker

Thoát vòng "bảo AI sửa mãi mà lỗi vẫn còn".

Sửa cả ngày không xong hiếm khi do AI kém. Thường là vì nó **đoán nguyên nhân rồi vá luôn, không đo**.
Vá sai thì nó vá thêm một lớp nữa lên trên. Code càng rối, cuộc chat càng đầy những lần sửa hỏng,
nên lần đoán sau càng sai. Skill này cắt vòng đó bằng 6 bước theo đúng thứ tự, mỗi bước có cổng chặn:
chưa qua cổng thì không sang bước sau.

```
0 Tái hiện ─► 1 Ba lăng kính ─► 2 Hai AI ─► 3 Spec ─► 4 Sơ đồ mới + code ─► 5 Kiểm thật
                   ▲                                                          │
                   └──────────── vẫn hỏng: quay lại với sự thật mới ──────────┘
```

---

## Khi nào dùng

Bắt buộc dùng khi gặp **một trong ba** điều sau, không chờ cảm giác "hình như đang luẩn quẩn":

- Sửa **cùng một file hoặc cùng một bộ phận lần thứ 2** cho cùng một loại lỗi
- Một lỗi đã kéo **quá 60 phút**
- Người dùng nói kiểu "lâu thế", "luẩn quẩn", "vẫn chưa xong", "lại lỗi đó"

**KHÔNG dùng khi:** lỗi vặt sửa một dòng là xong lần đầu, thông báo lỗi đã chỉ thẳng nguyên nhân
(thiếu thư viện, sai đường dẫn, thiếu key) và sửa xong chạy được ngay. Khi đó chỉ cần bước 0 và bước 5.

**Quyền DỪNG:** từ lúc skill này chạy, **cấm sửa code** cho tới khi xong bước 3. Mọi lần "thử sửa nhanh
cái này xem sao" trong lúc đang phân tích chính là cái vòng mà skill này sinh ra để cắt.

---

## Bước 0. Tái hiện lỗi và gom sự thật

Không có sự thật thì 3 lăng kính chỉ là suy luận trên đồ đoán, và AI thứ hai cũng đoán theo.

Ghi vào `FIX-NOTES.md` ở gốc dự án:

| Mục | Cần có |
|---|---|
| Lỗi nguyên văn | dán đúng thông báo lỗi, stack trace, hoặc mô tả "mong A, ra B" kèm số |
| Lệnh tái hiện | một lệnh hoặc một chuỗi thao tác, chạy là ra lỗi |
| Đầu vào gây lỗi | file, dữ liệu, URL cụ thể |
| Môi trường | phiên bản ngôn ngữ, thư viện, hệ điều hành, biến môi trường liên quan (chỉ ghi TÊN biến, không ghi giá trị khoá) |
| Bản còn chạy được | commit hoặc phiên bản cuối cùng chạy đúng (`git log`, `git bisect` nếu cần) |
| Đã thử những gì | mỗi lần thử **một dòng**: đã đổi gì → kết quả. Không dán lại cả lịch sử chat |

**Cổng:** có một lệnh chạy ra đúng lỗi. Chưa tái hiện được thì việc đầu tiên là thêm log hoặc in giá trị
thật ra cho tới khi tái hiện được. Không tái hiện được thì cũng không có cách nào chứng minh đã sửa xong.

Hay gặp: tới bước này thì lỗi đã lộ nguyên nhân (sai phiên bản, sai đầu vào, cache cũ). Khi đó nhảy
thẳng sang bước 5.

---

## Bước 1. Ba lăng kính, kèm sơ đồ luồng HIỆN TẠI

### 1a. Vẽ sơ đồ luồng hiện tại

Vẽ bằng Mermaid: dữ liệu đi từ đâu, qua những bước nào, ra đâu. **Đánh dấu chỗ hỏng bằng số đo**
(giá trị thật in ra ở mỗi bước), không phải bằng "có lẽ ở đây".

```mermaid
flowchart LR
  A[Đọc file CSV<br/>1.200 dòng ✓] --> B[Lọc trùng<br/>1.180 dòng ✓] --> C[Gọi API<br/>❌ 0 kết quả] --> D[Ghi kết quả]
```

Chưa có số ở một bước thì việc đầu tiên là đo bước đó.

### 1b. Lăng kính 1: nguyên lý gốc

Lập ba cột:

| Sự thật (đã đo, đã thấy) | Giả định (đang tin nhưng chưa kiểm) | Ràng buộc thật (không đổi được) |
|---|---|---|

Mỗi giả định ghi kèm **cách kiểm** và **kiểm luôn những cái rẻ**. Phần lớn lỗi kéo dài cả ngày nằm ở
một giả định chưa ai kiểm: "API trả đúng định dạng", "file này được đọc", "hàm này có chạy tới".

Rồi suy ngược từ các sự thật lên giải pháp, không suy theo kiểu "người khác làm thế nên mình làm thế".

### 1c. Lăng kính 2: bóc cấu trúc, tìm chỗ tắc THẬT

Dựa vào sơ đồ 1a, chỉ ra **một** chỗ tắc kèm số đo. Sửa ở chỗ không tắc thì sửa cả ngày cũng không
nhanh hơn, không đúng hơn.

### 1d. Lăng kính 3: đúng thứ tự

**Hỏi lại yêu cầu → Xoá → Đơn giản hoá → Tăng tốc → Tự động hoá.** Đi đúng thứ tự này:

1. Yêu cầu nào đang gây khó? Ai cần nó, có thật cần không?
2. Bước nào **xoá** được? Lỗi hay gặp nhất là cố sửa một bước lẽ ra phải xoá.
3. Phần còn lại đơn giản hoá thế nào?
4. Chỉ sau đó mới nghĩ tới nhanh hơn hay tự động hơn.

**Cổng:** viết được 3 dòng:
1. Giả định nào chưa từng kiểm (và đã kiểm được những cái nào)
2. Chỗ tắc nằm ở đâu, kèm số đo
3. Bước nào xoá được, và một thử nghiệm dưới 15 phút để chứng minh

---

## Bước 2. Hai AI cùng tìm phương án

Một AI đã kẹt trong vòng thì hay bám vào cách nghĩ cũ. Thêm một AI khác, ví dụ đang dùng Claude thì
hỏi thêm Codex, Gemini hoặc ChatGPT, và ngược lại.

Luật:

- **Đưa SỰ THẬT, không đưa lịch sử.** Đưa `FIX-NOTES.md`, sơ đồ, 3 dòng ở bước 1, các `file:dòng` liên
  quan. Không dán cả cuộc chat sửa hỏng, vì nó kéo AI thứ hai vào đúng cái vòng cũ.
- **AI thứ hai chỉ đọc, không sửa.** Nó đề xuất, không động vào code.
- **AI thứ hai không phải trọng tài.** Hai bên khác nhau thì không chọn bên nói tự tin hơn. Chọn bằng
  một **thử nghiệm dưới 15 phút** cho ra số.

Mẫu câu hỏi cho AI thứ hai:

```
Đây là một lỗi đã sửa nhiều lần không xong. Chỉ đọc, không sửa file.
SỰ THẬT: <dán FIX-NOTES.md>
SƠ ĐỒ: <dán mermaid>
CHỖ TẮC ĐÃ ĐO: <dòng 2 của bước 1>
FILE LIÊN QUAN: <file:dòng>
Hỏi: (1) nguyên nhân gốc theo bạn là gì, giả định nào bạn nghi nhất;
(2) bước nào XOÁ được thay vì sửa; (3) một thử nghiệm < 15 phút phân biệt giả thuyết của bạn với giả thuyết sau: <giả thuyết của AI thứ nhất>.
```

Ví dụ gọi Codex CLI ở chế độ chỉ đọc, chạy nền:

```bash
codex exec --skip-git-repo-check -s read-only -C "<thư mục dự án>" - < prompt.txt > codex-out.txt
```

**Cổng:** có một phương án được chọn, kèm lý do và kết quả thử nghiệm phân xử (nếu hai bên lệch nhau).
Báo người dùng **cả hai ý**, kể cả ý không được chọn.

---

## Bước 3. Viết lại spec

Lỗi kéo dài cả ngày thường là vì spec ban đầu mơ hồ hoặc sai, AI cứ code theo cái sai đó mãi. Viết lại
vào `SPEC.md` (hoặc sửa spec cũ):

```markdown
## Mục tiêu         — một câu: tool này làm gì cho ai
## Đầu vào          — định dạng, ví dụ thật
## Đầu ra           — định dạng, ví dụ thật
## Luồng            — các bước, khớp với sơ đồ mới ở bước 4
## Đã xoá           — những bước/yêu cầu bỏ đi ở bước 1, và lý do
## Ràng buộc        — giới hạn API, thời gian, chi phí...
## Nghiệm thu       — điều kiện "chạy đúng", đo được. BẮT BUỘC có:
                      - lệnh tái hiện ở bước 0 không còn ra lỗi
                      - một lần chạy với đầu vào thật, kết quả mong đợi bằng số
## Ngoài phạm vi    — những gì lần này không làm
```

**Cổng:** mục Nghiệm thu có ít nhất một điều kiện đo được và chứa ca lỗi của bước 0. Có chỗ phải để người
dùng chốt (bỏ tính năng, đổi hướng) thì hỏi họ ở đây, không tự quyết.

---

## Bước 4. Sơ đồ luồng MỚI, rồi mới code

1. Vẽ sơ đồ luồng mới bằng Mermaid, đặt cạnh sơ đồ cũ để thấy rõ bước nào bị xoá, bước nào thay.
2. Lưu lại trạng thái hiện tại trước khi sửa (`git commit` hoặc `git stash`), để quay lại được.
3. Code theo spec. Phần bị hỏng thì **viết lại theo spec** thay vì vá tiếp lên code đã sửa nhiều lần.
4. Đi từng bước nhỏ, mỗi bước chạy được thì mới làm bước tiếp.

**Cổng:** code khớp sơ đồ mới; không có bước nào trong code mà không có trong sơ đồ.

---

## Bước 5. Kiểm thật

AI code xong rồi tự báo "đã sửa" thì **chưa tính là xong**. Đó chính là cách vòng lặp bắt đầu.

1. Chạy **lệnh tái hiện ở bước 0**. Phải hết lỗi.
2. **Đối chứng âm:** chạy cùng phép thử đó trên bản cũ (hoặc bỏ phần vừa sửa). Bản cũ phải **hỏng**.
   Bản cũ cũng qua nghĩa là phép thử không đo đúng thứ cần đo, và "đã sửa" là giả.
3. Chạy với **đầu vào thật**, đúng như điều kiện Nghiệm thu trong spec.
4. **Người dùng nhìn kết quả bằng mắt**: con số, màn hình, file đầu ra. Agent báo cáo những gì đã chạy
   và thấy gì, không chỉ nói "pass".

**Hỏng:** quay lại **bước 1** (không phải bước 4) với những sự thật mới vừa đo được, ghi thêm vào
`FIX-NOTES.md`. Quay lại bước 1 lần thứ 2 mà vẫn hỏng thì dừng hẳn, báo người dùng: đã biết gì,
còn nghi gì, cần họ quyết gì.

**Đạt:** ghi một bài học vào `LESSONS.md` của dự án: lỗi trông thế nào, gốc thật là gì, lần sau nhận
ra sớm bằng dấu hiệu nào. Bẫy cắn từ 2 lần trở lên thì đáng đưa vào luật của dự án (`CLAUDE.md`,
`AGENTS.md`...).

---

## Bẫy đã gặp

| Bẫy | Trông thế nào | Chặn bằng |
|---|---|---|
| Vá chồng vá | mỗi lần sửa thêm một `if`, một `try/catch`, một lần thử lại | Quyền DỪNG; bước 1d hỏi "xoá được gì" |
| AI tự báo xong | "Đã sửa xong, giờ chạy được rồi" mà không có lệnh nào được chạy | Bước 5: chạy lại ca lỗi, người nhìn kết quả |
| Phép thử giả | lần chạy thứ 2 luôn qua vì cache; test không chạm tới chỗ hỏng | Bước 5.2: đối chứng âm, bản cũ phải hỏng |
| Lỗi bị nuốt | công cụ trả "thành công" hoặc kết quả rỗng trong khi thật ra đã lỗi | Bước 1a: đo giá trị thật ở từng bước, không tin trạng thái báo về |
| Mở chat mới mà dán cả lịch sử | AI mới lặp lại đúng các hướng sai cũ | Bước 2: chỉ đưa `FIX-NOTES.md` và sơ đồ |
| Chọn phương án theo độ tự tin | AI nói chắc nịch nên làm theo | Bước 2: thử nghiệm < 15 phút phân xử |
| Sửa ở chỗ không tắc | tối ưu một bước vốn chạy đúng | Bước 1c: chỉ ra chỗ tắc kèm số đo |

---

## Báo cáo cho người dùng

Sau mỗi bước, báo ngắn gọn:

```
Bước <n> xong.
- Thấy gì: <sự thật mới, kèm số>
- Quyết gì: <lựa chọn + lý do>
- Tiếp theo: <bước kế>
- Cần anh/chị: <việc cần người dùng quyết, hoặc "không">
```

Lần cuối báo: lỗi gốc là gì, đã xoá gì, đã đổi gì trong spec, kết quả kiểm thật (lệnh đã chạy + kết quả),
bài học đã ghi ở đâu.
