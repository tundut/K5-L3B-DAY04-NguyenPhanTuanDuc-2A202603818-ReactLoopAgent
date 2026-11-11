# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Phan Tuấn Đức
MSSV: 2A202603818

## 1. Case được chọn

- ID case: A6
- Câu trả lời của agent: Không có câu trả lời cuối; agent trả `max_steps`.
- Điểm ở lần chạy 1: 0/100

## 2. Bằng chứng trong trace

Dán đoạn trace chứng minh lỗi và nêu nguyên nhân có thể xảy ra.

```text
eval_before.json, case A6:
tool step 1: get_incident_details
tool step 2: get_incident_details
validation_missing_source: steps 3, 4, 5, 6, 7, 8
result.status: max_steps
result.answer: ""
```

Case A6 yêu cầu đọc incident GitHub `h02n9572mx9y`, theo incident liên quan
và dẫn nguồn. Agent đã gọi đủ hai tool cần thiết nhưng model nhiều lần trả
câu trả lời thiếu URL. Cơ chế validation cũ cứ gửi yêu cầu sửa tiếp cho đến
khi hết 8 lần gọi model, nên không tạo được câu trả lời hoàn chỉnh.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: Trong `agent.py`, giới hạn validation thiếu nguồn còn tối đa
  hai lần. Nếu model vẫn không đưa URL đã quan sát, agent ghi event
  `validation_source_appended`, tự thêm các URL từ tool result vào câu trả lời
  và trả `completed`.
- Vì sao chọn thay đổi này: Trace A6 cho thấy lỗi nằm ở điều kiện dừng và
  validation, không phải ở việc chọn tool. Thay đổi này ngăn vòng lặp vô hạn,
  vẫn yêu cầu model tự sửa trước, và chỉ thêm URL đã thực sự xuất hiện trong
  kết quả tool; không bịa nguồn mới.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 83.55/100 | 90.00/100 |
| Điểm case đã chọn | 0/100 | 93/100 |
| Số model calls | 8 | 4 |
| Token, chi phí nếu có | 20,032 token; không có trường chi phí | 5,875 token; không có trường chi phí |

- Lỗi đã giải quyết: A6 không còn `max_steps`; agent hoàn tất sau hai tool
  calls, hai lần yêu cầu bổ sung nguồn và một lần append URL có thật. Điểm
  toàn bộ tăng 6.45 điểm.
- Lỗi còn lại hoặc mới xuất hiện: Một số câu trả lời vẫn cần fallback
  `validation_source_appended` thay vì tự dẫn nguồn từ model. Eval judge vẫn
  có thể chấm khác nhau giữa các lần chạy vì kết quả phụ thuộc model.

## 5. Giới hạn

Điểm judge là đánh giá xấp xỉ bằng LLM, không phải chứng minh tuyệt đối về
độ đúng; judge có thể thiên lệch hoặc chấm khác ở lần chạy khác. Snapshot
được cố định theo thời điểm của bài nên không đại diện cho trạng thái dịch vụ
hiện tại, và dữ liệu incident có thể bị cắt, như trường `truncated` trong
case A6. Mỗi phiên bản chỉ được chạy một lần, vì vậy mức tăng từ 83.55 lên
90.00 chưa chứng minh độ ổn định thống kê. Hai eval dùng cùng provider/model,
judge, dataset và `max_steps=8`, và đều chấm đủ 22/22 case.

## 6. Bonus tùy chọn

Đã mở rộng agent để dùng schema và system prompt của `BonusTools`, nên agent
hỗ trợ năm provider GitHub, Cloudflare, Vercel, npm và DigitalOcean cùng tool
đọc timeline theo trang. Agent trả observation lỗi `RATE_LIMITED` hoặc
`INVALID_ARGUMENTS` về model; riêng retry rate limit được chặn tối đa ba lần
cho cùng tool/provider để không retry vô hạn.

Đã chạy:

```text
python3 -m unittest discover -s bonus -p 'test_*.py' -v
Ran 3 tests
OK
```

Kết quả [`eval_bonus.json`](./eval_bonus.json) có đủ 10/10 case, điểm bộ
bonus 94.5/100, quy đổi 14.17/15. B10 đạt 69 điểm vì câu trả lời chưa nêu
đủ rõ số lần retry Vercel và timestamp cho từng provider; đây là giới hạn
còn lại của cách model tổng hợp nhiều kết quả trong một lượt.
