# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Nguyễn Chí Bằng

Công cụ gán nhãn đã dùng: CVAT

## 1. Dữ liệu và cách chia tập

Camera đứng một chỗ, một chiếc xe nằm trong hình vài giây. Ảnh học và ảnh kiểm tra phải cách nhau theo thời gian, với vùng đệm ở giữa, để cùng một xe không vừa nằm trong tập học vừa nằm trong tập kiểm tra. Nếu trộn ngẫu nhiên, cùng một xe có thể vừa được AI học vừa được dùng để chấm. Điểm sẽ đẹp hơn sự thật.

## 2. Mô hình khởi đầu lạnh (cold start)

Dòng vòng 0 trong `reports/rounds_table.md`: yolov8n cold start, 0 ảnh train, AP50 0.771, P 0.925, R 0.489, F1 0.640, recall xe nhỏ 0.182, xe vừa 0.547, xe lớn 0.561.

Xe ở xa bị bỏ sót nhiều hơn xe ở gần. Trên `outputs/compare_round0.jpg`, các xe nhỏ phía trên đường thường là khung vàng, nghĩa là nhãn tham chiếu có xe nhưng model không tìm thấy. Xe lớn ở gần, đèn pha rõ, thường là khung xanh. Một chỗ khung lệch là vệt sáng sát mép phải trên ảnh test `frame_0250`: model khoanh đỏ vào vệt đèn, không phải thân xe.

Nhãn dùng để chấm cũng do máy vẽ, chưa có người xem từng khung, nên có thể nhãn chấm sai chứ không phải AI của mình sai.

## 3. Chiến lược chọn mẫu

Mỗi ảnh có một điểm `score = 0.5·U + 0.3·A + 0.2·D`. Một nửa điểm là AI không chắc (`U`). Ba phần mười là AI vẽ nhiều khung còn lưỡng lự (`A`). Hai phần mười là ảnh có khác thời gian với ảnh khác (`D`). Hai ảnh trong cùng lô phải cách nhau ít nhất 2 giây (`MIN_GAP_S`), vì camera đứng yên, ảnh sát nhau gần như giống hệt.

Ba ảnh đã viết trong `reports/SELECTION.md` là frame_0182.jpg, frame_0099.jpg và frame_0326.jpg. Ảnh cân nhắc khác là frame_0372.jpg: điểm 0.910, hạng 6, cao hơn một số ảnh được chọn, nhưng cách frame_0369.jpg chỉ 1.2 giây nên bị bỏ. Sửa cả hai thì tốn công mà gần như cùng một cảnh.

Điểm cao không có nghĩa sửa ảnh đó sẽ làm AI giỏi hơn.

## 4. Các vòng học chủ động (active learning)

Bảng trong `reports/rounds_table.md`:

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vong 1..1 | 12 | 258 | 0.438 | -0.333 | 1.000 | 0.027 | 0.053 | 0.000 | 0.027 | 0.073 |

Vòng 0 chưa có nhãn người sửa. Vòng 1, theo `outputs/round1_diff.md`: 12 ảnh, model đề xuất 169 khung, sau khi sửa còn 258 khung. Giữ nguyên 149, kéo lại 13, xóa 7, thêm 96.

AP50 vòng 1 là 0.438, thấp hơn khởi đầu lạnh 0.333. So với vòng ngay trước, cũng giảm 0.333 vì vòng trước chính là vòng 0. Cả ba nhóm xe đều xấu đi trên cùng 20 ảnh test: xe nhỏ từ 0.182 xuống 0, xe vừa từ 0.547 xuống 0.027, xe lớn từ 0.561 xuống 0.073. Precision lên 1.000 nhưng recall chỉ còn 0.027, tức model gần như không còn khoanh xe.

Trên `outputs/compare_round1.jpg`, hàng `frame_0050` cho thấy một xe lớn ở gần, đèn pha sáng: vòng 0 khoanh xanh, khớp nhãn tham chiếu, còn vòng 1 để khung vàng, tức bỏ sót. Cả hàng đó vòng 1 chỉ còn 1 đúng và 17 sót, trong khi vòng 0 có 11 đúng.

Ba việc này khác nhau. Trong `reports/BLIND_SCAN.md`, mắt mình nhìn `frame_0392.jpg` thấy khoảng 28 xe, hai xe bị cắt ở mép dưới và vài xe rất xa chỉ còn hai chấm đèn. Trong `reports/REVIEW_LOG.csv`, mình thêm khung cho xe bị cắt mép dưới ở `frame_0099.jpg`, kéo khung xe trên làn cho sát thân, và xóa khung vệt sáng đèn trên `frame_0326.jpg` vì đó không phải xe. Sau khi học lại, model không khá hơn: nó bỏ sót hầu hết xe trên tập test, kể cả xe gần mà vòng 0 đã tìm thấy.

Một ca khó theo quy tắc: xe bị cắt mép ảnh chỉ được khoanh phần thân còn nhìn thấy, không khoanh vệt đèn trên mặt đường. Đó là khung mình thêm ở mép dưới `frame_0099.jpg`.

## 5. Kết luận và giới hạn

AP50 vòng 1 là 0.438, thấp hơn vòng 0 là 0.771. Mình dừng, không làm vòng 2. Học 50 epoch trên 12 ảnh đêm, cùng một camera, đã làm model quên cách tìm xe. Làm tiếp trên lô mới cũng là những ảnh sát giờ, tốn thời gian gán nhãn mà hai ảnh gần nhau gần như một cảnh.

Hai chỗ còn yếu: xe rất xa chỉ còn hai chấm đèn, như chỗ mình ghi ở phía trên `frame_0392.jpg`, và vệt sáng đèn trên mặt đường bị khoanh nhầm thành xe, như `frame_0326.jpg`. Xe quá nhỏ, cao dưới 16 pixel, bài chấm còn bỏ qua.

Tập kiểm tra chỉ có 20 ảnh. Xe quá nhỏ không tính. Nhãn chấm do máy vẽ, chưa có người kiểm, nên điểm giảm chưa chứng minh mình gán nhãn sai. Trước khi cho AI học thêm, mình sẽ xem lại 96 khung đã thêm và 7 khung đã xóa trong `outputs/round1_diff.md`, vì lô nhỏ mà nhãn lệch sẽ kéo model đi tiếp.
