# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Nguyễn Thị My

Công cụ gán nhãn đã dùng: CVAT

## 1. Dữ liệu và cách chia tập

Camera đứng cố định ở một điểm trên đường cao tốc và liên tục chụp ảnh. Mỗi chiếc xe đi qua có thể xuất hiện trong nhiều frame liên tiếp, nghĩa là hai ảnh gần nhau về thời gian gần như là cùng một cảnh. Vì vậy, tập huấn luyện và tập kiểm thử phải được chia theo trục thời gian — lấy một đoạn thời gian đầu để học và đoạn thời gian sau để kiểm tra, với vùng đệm ở giữa để hai tập không dùng chung xe.

Nếu chia ngẫu nhiên, cùng một chiếc xe xuất hiện ở nhiều frame có thể vừa rơi vào tập học vừa rơi vào tập kiểm thử. Khi đó AI đã "nhìn thấy" chiếc xe đó lúc học, nên khi kiểm tra lại nó nhận ra dễ dàng hơn — điểm AP50 trông cao hơn thực tế. Mô hình thực ra không tổng quát hóa được ra ngoài đoạn video đó, nhưng số đo trên tập test lại đẹp một cách giả tạo.

## 2. Mô hình khởi đầu lạnh (cold start)

Dòng vòng 0 từ `rounds_table.md`:

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |

Mô hình cold start đạt AP50 = 0.771 trên 20 ảnh kiểm thử với 403 box tham chiếu. Precision khá cao (0.925) — khung nào AI vẽ thì hầu hết là xe thật — nhưng Recall rất thấp (0.489), tức là AI bỏ sót hơn một nửa số xe. Nhìn `outputs/compare_round0.jpg`, thấy rõ nhiều xe ở giữa ảnh không có khung dù nhìn bằng mắt thường vẫn phân biệt được thân xe và đèn.

Recall theo kích thước xe cho thấy rõ vấn đề: xe nhỏ (small) chỉ đạt recall 0.182 — AI bỏ sót gần 82% xe nhỏ. Đây là những xe ở xa, gần chân cầu, chỉ còn vài pixel. Xe vừa và xe lớn tốt hơn nhưng vẫn dưới 0.56. Một trường hợp cần lưu ý: nhãn tham chiếu của tập kiểm thử do mô hình tạo tự động và chưa có người rà lại từng khung. Có thể một số xe nhỏ thực ra bị nhãn tham chiếu bỏ sót, khiến recall của AI trông thấp hơn thực tế.

## 3. Chiến lược chọn mẫu

Mỗi ảnh trong pool được tính một điểm theo công thức: `score = W_U·U + W_A·A + W_D·D`, trong đó:
- **U** (Uncertainty): mức độ AI không chắc — tính từ điểm confidence của từng box; AI phân vân càng nhiều thì U càng cao. Trọng số W_U = 0.5.
- **A** (Ambiguity): tỷ lệ số box AI còn lưỡng lự trên tổng số box — box nào có confidence gần ngưỡng thì tính là lưỡng lự. Trọng số W_A = 0.3.
- **D** (Diversity): ảnh cách ảnh gần nhất trong lô bao nhiêu giây theo thời gian — để tránh chọn hai ảnh gần như giống hệt nhau. Trọng số W_D = 0.2.

`MIN_GAP_S` là khoảng cách thời gian tối thiểu giữa hai ảnh trong cùng một lô (ít nhất 2 giây). Quy tắc này đảm bảo AI không lãng phí công sửa nhãn vào hai ảnh chụp cùng một cảnh: frame_0372.jpg (t=148.8s, điểm 0.910) bị bỏ qua vì cách frame_0369.jpg (t=147.6s) chỉ 1.2 giây — dù điểm cao hơn frame_0107.jpg (0.888) vẫn được chọn.

Ba frame trong SELECTION.md minh họa rõ cách cân nhắc: frame_0182.jpg (hạng 1, điểm 0.959) có U=0.918 và A=1.0 — AI cực kỳ phân vân ở ảnh này; frame_0099.jpg (hạng 8, điểm 0.906) có U=0.946 cao nhất trong lô; frame_0392.jpg (hạng 15, điểm 0.887) thấp hơn nhưng đứng ở t=156.8s đủ xa về thời gian. Điểm bất định cao không có nghĩa là sửa ảnh đó sẽ làm AI giỏi hơn — nó chỉ nói AI đang phân vân, không nói việc sửa sẽ giải quyết điểm yếu nào của mô hình.

## 4. Các vòng học chủ động (active learning)

Bảng từ `rounds_table.md`:

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vong 1..1 | 12 | 169 | 0.378 | -0.393 | 1.000 | 0.030 | 0.058 | 0.000 | 0.020 | 0.146 |

Từ `outputs/round1_diff.md`: trong 12 ảnh vòng 1, model đề xuất 169 box. Sau khi rà soát trên CVAT: 169 box được giữ nguyên (accepted=169), không có box nào bị kéo lại (edited=0), xóa (deleted=0) hay thêm mới (added=0). Accept rate = 100%.

Sau khi fine-tune trên 12 ảnh với 169 box, AP50 giảm mạnh từ 0.771 xuống 0.378 (Δ = -0.393). Recall sụp đổ từ 0.489 xuống 0.030 — AI gần như không còn tìm thấy xe nào (chỉ tìm được 12/403 box tham chiếu, tp=12, fn=391). Trong khi đó Precision đạt 1.000 — những khung AI vẽ đều là xe thật, nhưng nó gần như không vẽ khung nào. Recall xe nhỏ giảm về 0, xe vừa từ 0.547 xuống 0.020, xe lớn từ 0.561 xuống 0.146.

Mở `compare_round0.jpg` và `compare_round1.jpg`: ở vòng 0 AI vẽ nhiều khung dù còn sai vị trí; ở vòng 1 hầu như không còn khung nào xuất hiện trong ảnh kiểm thử — mô hình bị overfitting nặng trên 12 ảnh train, mất khả năng tổng quát hóa. Đây là hiện tượng catastrophic forgetting điển hình khi fine-tune với ít dữ liệu (12 ảnh) mà không giữ lại dữ liệu COCO gốc.

So với quan sát độc lập trong BLIND_SCAN.md (đếm được khoảng 18 xe ở frame_0099.jpg), AI đề xuất 13 khung prelabel — con số hợp lý. Tuy nhiên sau khi fine-tune, mô hình vòng 1 chỉ tìm được 12 box trên toàn bộ 20 ảnh test, cho thấy fine-tune đã làm mất kiến thức COCO. Cần kiểm tra lại cách đóng gói data trước khi train thêm.

## 5. Kết luận và giới hạn

AP50 vòng 1 = **0.378**, giảm mạnh so với cold start 0.771 (Δ = -0.393). Tôi chọn **dừng lại** và không train thêm vòng 2 vì:

1. Điểm đã giảm rõ rệt — fine-tune với 12 ảnh không cải thiện mà còn làm mất kiến thức COCO của mô hình. Trước khi tiếp tục cần kiểm tra lại cách đóng gói data và quy trình train.
2. Recall sụp đổ về 0.030 — mô hình gần như không phát hiện xe nào trên tập test. Đây là dấu hiệu overfitting hoặc data pipeline có lỗi, cần debug trước khi đổ thêm nhãn vào.

Hai chỗ còn yếu nếu muốn cải thiện ở vòng sau:
1. **Xe nhỏ ở xa (small recall = 0.000 ở vòng 1, 0.182 ở vòng 0)**: cần ảnh có xe nhỏ rõ ràng, nhưng chi phí rà nhãn cao và nguy cơ chọn ảnh sát nhau lớn. Nên chọn ảnh cách nhau tối thiểu 5 giây thay vì 2 giây để tránh ảnh trùng.
2. **Xe bị cắt mép**: hai ảnh frame_0099.jpg và frame_0392.jpg có xe bị cắt mép, AI vòng 1 không còn nhận ra — cần bổ sung nhãn dạng này trong lô tiếp theo.

Giới hạn của đánh giá: tập kiểm thử chỉ 20 ảnh, bỏ qua 14 box cao dưới 16px, nhãn tham chiếu do mô hình tạo chưa có người rà thủ công — nên kết quả AP50 có thể không phản ánh đúng khả năng thực tế. Nếu muốn train tiếp, cần kiểm tra lại: (a) data.zip có đủ cả ảnh pool cũ + nhãn round1 không, (b) các file .txt round1 có đúng định dạng YOLO không, (c) số epoch có quá nhiều với 12 ảnh không.
