# bow-skill-qc

Bộ skill **QC** cho [bow-agent](https://github.com/Bow-T/bow-agent). Được đăng ký như một *stack*
trong registry và nạp **tự động** khi bow-agent chạy ở **QC Mode** (`npm run ui:qc` /
`BOW_QC_MODE=true`).

QC Mode là read-only với source code nhưng mở tool `Skill` + Jira read/write, nên các skill ở đây
có thể chấm ticket và (khi được duyệt) ghi kết luận trở lại Jira.

## Skill

| Skill | Việc | Ghi Jira? |
|-------|------|-----------|
| `qc-triage` | Chấm một ticket có gán ĐÚNG loại (Bug/Task/Story/Improvement…) không, in kết luận + lý do + độ tin cậy. | Không — chỉ đọc & báo cáo. |
| `qc-triage-apply` | Như `qc-triage`, rồi khi người dùng duyệt thì post comment kết luận / đổi issue type / chuyển trạng thái. | Có — nhưng LUÔN hỏi duyệt trước khi ghi. |

## Cấu trúc

```
bow-skill.json          # manifest: id=qc, skillsDir=skills, danh sách skill
skills/
  qc-triage/SKILL.md
  qc-triage-apply/SKILL.md
```

## Ghim phiên bản

bow-agent chỉ tải theo **tag đã ghim** trong registry (`~/.bow-agent/registry.json`). Đổi nội
dung → tạo tag mới (vd `v1.1.0`) rồi cập nhật `ref` trong registry; bản đã duyệt là bất biến.
