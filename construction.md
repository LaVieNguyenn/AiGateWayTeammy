0) Mục tiêu cuối
User gõ 1 câu mô tả vấn đề/ý tưởng.
Assistant:
hỏi 1–3 câu để đủ bối cảnh (nếu thiếu),
đưa ra draft (task/backlog/milestone + checklist/debug/test) để user sửa,
hiển thị “có task tương tự” (dedupe) dựa vào status,
user bấm Confirm → hệ thống mới gọi AI “bơm data” + gọi API tạo/đổi dữ liệu.
1) Nguyên tắc “2-phase” (quan trọng)
Phase A — Draft (NO side effects)
Không tạo task/backlog/milestone thật.
Chỉ trả về:
câu trả lời cho user,
draft JSON,
danh sách “similar items”.
Phase B — Commit (có side effects)
Chỉ chạy khi user confirm.
Lúc này mới:
(tuỳ bạn) gọi AI lần 2 để “bơm data” hoàn chỉnh theo draft user đã sửa,
gọi các API project-management để tạo/update.
2) Dedupe: đúng như bạn muốn
Chỉ cần nói: “Mình thấy có task tương tự ở đây … có vẻ đã xử lý rồi”
Cách kết luận dựa vào status:

Nếu status ∈ {done/archived/resolved} → “có vẻ đã xử lý”
Nếu status ∈ {in_progress/blocked/ready/planned} → “đang được xử lý / chưa xong”
Nếu status không rõ → “cần kiểm tra thêm”
Output dedupe chỉ là thông tin cho user, không chặn tạo mới.

2.1) Status allow-list theo code hiện tại (Teammy)
- Backlog status (allow-list): planned | ready | in_progress | blocked | completed | archived
- Milestone status (allow-list): planned | in_progress | completed | blocked | archived | slipped
- Task status: hiện tại là string? và không thấy allow-list cứng trong DTO; hệ thống đang dựa vào Column.IsDone để suy ra backlog status khi task di chuyển.

2.2) Dedupe theo độ giống nội dung (>= 85)
- Khi phát hiện 1 board task gần đây có độ giống nội dung >= 85 (thang 0–100) so với yêu cầu user:
	- Agent PHẢI hỏi user chọn 1 trong 3:
		(A) “Không giống” → tạo mới như bình thường
		(B) “Giống, cần update” → Phase B sẽ update task/backlog hiện có (không tạo mới)
		(C) “Bỏ qua” → không tạo mới, không update
- Nếu user confirm “không giống” thì dù score >= 85 vẫn tạo mới.
- Dedupe theo similarity chỉ là gợi ý; quyết định cuối cùng thuộc user.

3) Bối cảnh bắt buộc phải “ghi rõ cho AI” (để AI trả lời đúng)
Trước khi gọi AI (kể cả phase draft hay phase bơm data), backend phải cung cấp “facts” kiểu tool-assisted:

3.1 Context chung
groupId, userId, role của user (member/leader/mentor)
policy/team info (nếu có): quy mô nhóm, kỳ hiện tại, …
constraint: no action without confirm
3.2 Snapshot dữ liệu hiện có (để dedupe & gợi ý)
Backlog items hiện tại: {id,title,status,updatedAt,milestoneId?}
Milestones hiện tại: {id,title,status,start/end}
Board tasks hiện tại: {id,title,column,status?,assignees?,backlogItemId?}
Một số status allow-list đang dùng trong hệ thống (để AI không bịa status)
3.3 Prompt rule cho AI
“Không bịa facts”
“Nếu thiếu info → hỏi tối đa 1–3 câu”
“Trả JSON theo schema”
“Dedupe: phải mention 1–3 items tương tự kèm status và nhận định ‘có vẻ đã xử lý’ nếu done/archived”
4) Luồng chi tiết end-to-end
Step 1 — User gửi yêu cầu
UserText ví dụ: “Login bị lỗi không lưu session sau redirect”.

Backend nhận: groupId, userText, (optional) preferredMode=backlog-first.

Step 2 — Backend lấy facts (tool-assisted)
Fetch backlog list
Fetch milestones list
Fetch board + tasks/columns
Tạo similarCandidates (top 3–5) bằng heuristic đơn giản:
match keyword + title similarity + recency
Step 3 — Agent hỏi (nếu thiếu)
AI phải chọn 1 trong 2:

Đủ bối cảnh → đi thẳng tạo draft.
Thiếu bối cảnh → hỏi 1–3 câu, ví dụ:
“Lỗi xảy ra trên môi trường nào (local/tunnel/prod)?”
“Browser nào?”
“Có steps tái hiện tối thiểu không?”
Backend trả về cho user: questions[] + một draft “tạm” (optional).

Step 4 — AI tạo Draft (Phase A, NO side effects)
AI trả về JSON draft gồm:

title
type (bug/feature/chore)
priority
description.context / observed / expected / impact
reproSteps[]
debugPlan[]
testChecklist[]
acceptanceCriteria[]
suggestedActions[] (chỉ là đề xuất)
dedupeNote + similarItems[] (đính kèm status + nhận định)
Backend render draft cho user chỉnh sửa.

Step 5 — User chỉnh sửa draft
User được phép sửa:

title/priority
reproSteps/debugPlan/testChecklist
chọn milestone / chọn column / chọn assignees (nếu muốn)
tick “create new” hoặc “link to existing” (nếu dedupe)
Step 6 — User Confirm
User bấm Confirm → gửi approvedDraft + userEdits.

Step 7 — “Bơm data” bằng AI (Phase B – chỉ chạy sau confirm)
Mục tiêu: biến draft (có thể hơi thô) thành bản cuối “đẹp, đúng format, nhất quán”.

AI nhận input:

draft user đã sửa
facts hiện tại (group, milestones, columns)
rule không bịa
AI output:

finalBacklogItem (title/description/status)
finalMilestone (nếu cần)
finalTask (nếu promote hoặc task-first)
finalComments[] (nếu bạn muốn checklist/debug/test nằm ở comment)
executionPlan[] (list API actions cụ thể + arguments)
Step 8 — Execute commit (backend gọi API nội bộ)
Theo “executionPlan” (deterministic), ví dụ backlog-first:

POST tracking/backlog (create backlog item)
nếu user chọn milestone:
POST tracking/milestones/{id}/items
nếu user chọn promote:
POST tracking/backlog/{backlogItemId}/promote (cần columnId)
nếu muốn checklist/debug/test ở comment:
POST board/tasks/{taskId}/comments
assignees:
PUT board/tasks/{taskId}/assignees
move column (optional):
POST board/tasks/{taskId}/move
Step 9 — Trả kết quả cho user
Backend trả:

IDs đã tạo: backlogId/milestoneId/taskId/commentIds
Link/route để mở item
Tóm tắt “đã tạo gì” + “vì sao” + “nếu muốn sửa” hướng dẫn chỗ sửa
5) Output mẫu “dedupe statement” (đúng kiểu bạn muốn)
Trong câu trả lời (không nhất thiết trong JSON), AI nói ngắn:

“Mình thấy có task tương tự: #123 Login session mất sau redirect (status: done) — có vẻ đã xử lý rồi. Bạn vẫn muốn tạo task mới hay update task cũ?”
6) 3 quyết định nhỏ bạn cần chốt (để mình viết schema cuối cùng)
Backlog-first hay task-first?
- Phụ thuộc lựa chọn của user.
- Nếu user không nói gì: MẶC ĐỊNH backlog-first (tạo backlog trước) rồi promote lên task.

Checklist/debug/test lưu ở description hay comment?
- Nếu có field description: ghi rõ checklist/debug/test trong description (ưu tiên dùng description).

Dedupe theo similarity >= 85 (task gần đây có nội dung tương tự):
- Nếu phát hiện >= 85: hỏi user (Không giống / Giống cần update / Bỏ qua).
- User chọn “Không giống” → tạo mới.
- User chọn “Giống cần update” → update item hiện có.
- User chọn “Bỏ qua” → không tạo mới, không update.

Với các quyết định này, schema Phase A + Phase B cần hỗ trợ:
- draft.mode (backlog-first|task-first|auto)
- draft.description (nơi chứa checklist/debug/test khi có description)
- draft.dedupeDecision (create_new|update_existing|ignore)
- similarItems[] kèm similarityScore (0–100)