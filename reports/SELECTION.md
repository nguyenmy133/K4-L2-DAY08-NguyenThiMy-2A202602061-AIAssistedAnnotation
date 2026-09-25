# Vì sao chọn lô này?

Nếu chỉ được sửa 5 ảnh, tôi chọn frame_0182.jpg (hạng 1, điểm 0.959), frame_0369.jpg (hạng 2, điểm 0.932), frame_0380.jpg (hạng 3, điểm 0.917), frame_0326.jpg (hạng 4, điểm 0.916) và frame_0331.jpg (hạng 5, điểm 0.915). Năm ảnh này đứng đầu danh sách vì điểm bất định cao nhất. frame_0369.jpg (t=147.6s) và frame_0372.jpg (t=148.8s) cách nhau chỉ 1.2 giây — cảnh gần như giống hệt nhau — nên AI đã bỏ frame_0372.jpg dù điểm đứng hạng 6 (0.910), và đó là quyết định đúng để tránh lãng phí công sửa nhãn.

Ba frame thuộc lô 12 ảnh model chọn mà tôi xem kỹ nhất là frame_0182.jpg (hạng 1, điểm 0.959), frame_0099.jpg (hạng 8, điểm 0.906) và frame_0107.jpg (hạng 14, điểm 0.888). Cả ba đều có điểm U (bất định) cao — lần lượt 0.918, 0.946, 0.875 — cho thấy AI còn phân vân nhiều khung. Trong selection_round1.jpg, ba ảnh này xuất hiện với số box cao (28–33 box) và tỷ lệ ambiguous lớn (14–18 box lưỡng lự trên tổng số).

frame_0372.jpg đứng hạng 6 với điểm 0.910, cao hơn nhiều ảnh đã được chọn như frame_0107.jpg (hạng 14, 0.888), nhưng AI loại nó vì cách frame_0369.jpg chỉ 1.2 giây. Hai ảnh chụp gần như cùng một cảnh đường cao tốc, sửa cả hai tốn gấp đôi công mà AI học thêm gần như bằng không. Ngược lại, frame_0392.jpg đứng hạng 15 (điểm 0.887) nhưng vẫn được chọn vì nó nằm ở t=156.8s — đủ xa về thời gian so với các ảnh cùng lô.

Điểm cao chỉ nói lên AI đang phân vân ở ảnh đó. Nó không chứng minh rằng sửa ảnh đó xong thì AI sẽ nhận xe tốt hơn trên tập kiểm thử. Hai ảnh hạng cao có thể chụp cùng một cảnh kẹt xe, AI học xong vẫn không biết xử lý xe nhỏ ở xa hay xe bị cắt mép — đó là giới hạn chính của chiến lược chọn mẫu theo độ bất định.
