# Ghi chú phân tích · Day 04

Họ và tên: Bùi Hữu Phúc
MSSV: 2A202603572

## 1. Case được chọn

- ID case: H7 – kiểm tra trạng thái GitHub Actions và Cloudflare Spectrum khi một công cụ gặp timeout.
- Câu trả lời của agent: Agent xác nhận GitHub Actions hoạt động bình thường dựa trên dữ liệu tool, nhưng với Cloudflare Spectrum, dù tool báo TimeoutError, Agent vẫn viết rằng dịch vụ đang hoạt động bình thường. Đây là kết luận không có bằng chứng.
- Điểm ở lần chạy 1: Cần lấy trực tiếp điểm H7 từ eval_before.json để ghi chính xác; không dùng điểm của một lần chạy thử khác thay thế.

## 2. Bằng chứng trong trace

Dán đoạn trace chứng minh lỗi và nêu nguyên nhân có thể xảy ra.

```
Trace của một lần kiểm tra H7 cho thấy: 

step 1: get_status(provider="github", component="actions")
        -> overall: All Systems Operational; Actions: operational
step 1: get_status(provider="cloudflare", component="spectrum")
        -> {"ok": false, "error": "TimeoutError"}
step 2: model trả câu trả lời cuối cùng
        -> "... kết luận rằng Cloudflare Spectrum đang hoạt động bình thường."
- Nguyên nhân: Công cụ đã không xác minh được trạng thái Cloudflare, nhưng model vẫn đưa ra kết luận operational. Chỉ hướng dẫn trong system prompt chưa đủ để ngăn lỗi này. Ở một lần thử, phần Validation Gate dựa vào cụm từ cố định cũng không phát hiện được cách diễn đạt khác của model.
```

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: Cải tiến agent.py, bổ sung quy tắc chỉ kết luận trạng thái khi có dữ liệu từ tool; ghi nhận các component có get_status thất bại; thêm bước kiểm tra câu trả lời khi có trạng thái chưa xác minh. Giữ lại ReAct Loop, xử lý lỗi, trace và cơ chế kiểm tra URL incident.
- Vì sao chọn thay đổi này: Ngăn Agent biến lỗi timeout thành kết luận dịch vụ hoạt động bình thường; đồng thời vẫn giữ được kết quả hợp lệ của các dịch vụ khác đã xác minh thành công.
- Kiểm tra: python -m unittest discover -s tests -v đạt 37/37 PASS trong lần chạy đã báo cáo.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng |81.00/100  |81.36/100 (kết quả `eval_final.json`, cần sao chép thành `eval_after.json` sau khi xác nhận cấu hình)  |
| Điểm case đã chọn(H7) |Cần đọc từ file  |Cần đọc từ trace  |
| Số model calls |51  |48  |
| Token, chi phí nếu có |0.062$  |0.022$  |

- Lỗi đã giải quyết: Agent có thêm cơ chế kiểm tra khi tool lỗi; trong lần Eval Final, H7 đạt 80 điểm. Điểm tổng tăng 0.36 điểm so với Before.
- Lỗi còn lại hoặc mới xuất hiện: H7 chưa ổn định: một lần chạy riêng sau đó chỉ đạt 55/100. Một số case khác vẫn chưa đạt điểm tối đa. Cần kiểm tra lại câu trả lời và trace nếu muốn cải thiện thêm.

## 5. Giới hạn

- Điểm judge: Là kết quả đánh giá tự động, có thể khác nhau giữa các lần chạy; không chứng minh Agent luôn trả lời đúng.
- Snapshot: Công cụ chỉ quan sát dữ liệu tại thời điểm snapshot và một phạm vi lịch sử incident giới hạn; không thể suy ra toàn bộ lịch sử hay trạng thái ngoài phạm vi được cung cấp.
- Một lần chạy mỗi phiên bản: Điểm 81.36/100 của lần Final chỉ phản ánh một lần chạy 22 case. Kết quả H7 khác nhau giữa lần chạy đầy đủ và chạy riêng cho thấy chưa thể kết luận tính ổn định nếu không chạy lặp lại nhiều lần.
