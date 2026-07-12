---
name: qc-triage-apply
version: "1.0.0"
description: Chấm loại một Jira ticket (như qc-triage) RỒI, khi người dùng duyệt, GHI kết luận lên Jira — post comment tóm tắt và (nếu cần) đổi issue type / chuyển trạng thái. Dùng trong QC Mode (đã mở Jira read/write). Luôn hỏi xác nhận trước khi ghi; không tự ý sửa khi chưa được duyệt.
argument-hint: "<ticket-key-hoặc-URL> (vd DUOCT-2346)"
allowed-tools: Read, mcp__jira__jira_get_issue, mcp__jira__jira_get_comments, mcp__jira__jira_get_transitions, mcp__jira__jira_add_comment, mcp__jira__jira_transition_issue, mcp__jira__jira_update_issue, AskUserQuestion
user-invocable: true
---

# /qc-triage-apply — QC chấm loại rồi GHI kết luận lên Jira

Mở rộng của `/qc-triage`: sau khi chấm loại ticket, skill này có thể **ghi kết luận trở lại Jira**
(comment + đổi type/trạng thái) — nhưng CHỈ khi người dùng duyệt. Dành cho QC Mode, nơi bow-agent
đã mở Jira read/write. Nếu chỉ muốn xem chẩn đoán mà không ghi, dùng `/qc-triage` thay vì skill này.

## Đầu vào

Một tham chiếu ticket: key thuần (`DUOCT-2346`) hoặc URL Jira. Bóc lấy **ticket key**
(mẫu `[A-Z][A-Z0-9]+-\d+`). Không có key hợp lệ → hỏi lại đúng 1 câu, đừng đoán.

## Bước 1 — Chấm loại (giống qc-triage)

Gọi `mcp__jira__jira_get_issue` (lấy issue type, summary, description, status, labels, custom
field bước tái hiện) rồi `mcp__jira__jira_get_comments`. Nếu còn mơ hồ và có ảnh đính kèm thì
`Read` ảnh; video thì gợi ý `/watch` (chỉ khi text chưa đủ). Đối chiếu với **định nghĩa chuẩn**:

| Loại | Bản chất | Dấu hiệu |
|------|----------|----------|
| **Bug** | Hệ thống chạy SAI so với spec đã có | kết quả-mong-đợi ≠ thực-tế; "lỗi/sai/crash/không hiển thị"; có bước tái hiện |
| **Task** | Việc kỹ thuật/hạ tầng, không hướng người dùng | "cấu hình/dựng/nâng cấp lib/dọn dẹp/script/CI" |
| **Story** | Tính năng MỚI theo góc người dùng | "Là <vai trò> tôi muốn <mục tiêu> để <lợi ích>"; hành vi chưa từng có |
| **Improvement** | Cải tiến cái đang chạy ĐÚNG | "tối ưu/nhanh/gọn/UX tốt hơn"; không phải lỗi, không phải tính năng mới |
| **Sub-task** | Mảnh việc con của ticket cha | có parent; phạm vi hẹp |

Ranh giới hay nhầm: **Bug vs Story** — tính năng đã chạy rồi mà sai → Bug; chưa có, yêu cầu tạo →
Story. **Bug vs Improvement** — sai spec → Bug; đúng spec nhưng muốn tốt hơn → Improvement.
Nếu dự án có quy ước riêng (`docs/qc-*.md`), đọc và ưu tiên nó.

In chẩn đoán gọn (như qc-triage) trước khi hỏi ghi:

```
🎫 <KEY> — <summary>
Loại hiện tại : <type đang gán>
Kết luận      : ✅ ĐÚNG | ⚠️ SAI | ❓ Chưa chắc
Loại đề xuất  : <type nên gán> (nếu khác)
Độ tin cậy    : Cao | Trung bình | Thấp
Lý do         : <2–4 gạch đầu dòng bám dữ liệu thật>
```

## Bước 2 — HỎI DUYỆT trước khi ghi

**Không bao giờ tự ghi lên Jira khi chưa được duyệt.** Dùng `AskUserQuestion` (hoặc hỏi thẳng)
để người dùng chọn hành động, đề xuất mặc định theo kết luận:

- **Chỉ comment kết luận** — post tóm tắt chẩn đoán, KHÔNG đổi type/trạng thái. (An toàn nhất;
  nên là mặc định khi độ tin cậy chỉ Trung bình/Thấp.)
- **Comment + đổi issue type** — chỉ khi kết luận "⚠️ SAI" và độ tin cậy **Cao**, và loại đề
  xuất khác loại hiện tại.
- **Comment + chuyển trạng thái** — khi quy trình QC yêu cầu (vd đưa ticket sang "Cần làm rõ").
- **Không ghi gì** — chỉ giữ chẩn đoán tại chỗ.

Nếu độ tin cậy **Thấp** → mặc định KHÔNG đổi type; chỉ comment câu hỏi cần làm rõ cho reporter.

## Bước 3 — GHI theo lựa chọn đã duyệt

- **Comment**: `mcp__jira__jira_add_comment` với nội dung bám mẫu chẩn đoán ở Bước 1 (nêu rõ đây
  là nhận định QC tự động qua bow-agent, kèm lý do trích dẫn dữ liệu). Viết tiếng Việt, gọn.
- **Đổi issue type**: `mcp__jira__jira_update_issue` đặt lại trường issue type sang loại đề xuất.
  Chỉ làm khi người dùng đã chọn phương án này.
- **Chuyển trạng thái**: trước tiên `mcp__jira__jira_get_transitions` để lấy transition hợp lệ
  của ticket, rồi `mcp__jira__jira_transition_issue` với transition id/tên đúng. Đừng đoán tên
  trạng thái — luôn lấy danh sách transition thật trước.

Sau khi ghi, in xác nhận ngắn: đã post comment id nào / đổi type từ X→Y / chuyển sang trạng thái
Z. Nếu một thao tác ghi trả lỗi (permission, transition không hợp lệ…), báo rõ lỗi và KHÔNG thử
lại mù quáng — nêu để người dùng quyết.

## Nguyên tắc

- **Bám bằng chứng**: mọi kết luận và comment phải trích từ mô tả/comment/ảnh thật.
- **Duyệt trước khi ghi**: không có bước "tự động sửa Jira". Người dùng luôn là người quyết ghi gì.
- **Thành thật về độ chắc**: thiếu dữ liệu → để "❓ Chưa chắc", chỉ comment câu hỏi, không đổi type.
- **Tối thiểu tác động**: ưu tiên comment; chỉ đổi type/trạng thái khi thật rõ và đã được duyệt.
