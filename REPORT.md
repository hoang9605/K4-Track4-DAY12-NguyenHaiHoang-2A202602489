# REPORT — Lab Day 12

| | |
|---|---|
| Họ và tên | Nguyễn Hải Hoàng |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | `2A202602489` |
| GitHub | `hoang9605` |
| Seed chia dữ liệu / seed mô hình | Split seed 4502; model seeds 1684, 1685, 1686, 1687, 1688 |
| Số seed huấn luyện bạn dùng (`--seeds`) | 5 |

## Kết quả cuối

Nguồn: `evidence/openloop.json`, `closedloop.json`, `gate.json`. L2 của EgoMLP và PercMLP là min–max qua 5 seed; PercMLP+ là trung bình ± độ lệch chuẩn vì evidence chỉ lưu dạng tổng hợp này. Va chạm của mô hình học là min–max qua seed trên cùng 2.883 quỹ đạo test. Quyết định “qua” chỉ có ý nghĩa trong phạm vi cổng mô phỏng của bài lab.

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity (`const-vel`) | 1,2595 | 18,9733 | Chặn |
| IDM | 1,5118 | 0,0694 | baseline |
| EgoMLP | 0,7501–0,7554 | 17,2390–45,1960 | Chặn |
| PercMLP | 0,6158–0,6232 | 0,5550–2,7749 | Chặn |
| PercMLP+ (sau khi sửa) | 0,8034 ± 0,0128 | 0,2775–0,3469 | Qua 7/7 quy tắc |

## Câu 1 · Dữ liệu

Theo `data_audit.json`, dữ liệu có 992.679 dòng, 9.993 quỹ đạo; loại 383 quỹ đạo trùng hoàn toàn còn 9.610 quỹ đạo sạch, lấy mẫu mỗi 0,1 s.
Tương quan Acc_LV với đạo hàm Speed_LV chỉ là 0,0454, trong khi tương quan tương ứng của xe theo sau đạt 0,9291; vì vậy không thể tin Acc_LV để tìm ca phanh của xe dẫn.
Tôi dùng `logs.a_lead`, suy ra từ tốc độ xe dẫn đã làm mượt, giúp giảm ảnh hưởng của nhiễu khi lấy đạo hàm; khoảng cách cũng được kiểm tra phù hợp hiệu vị trí trừ chiều dài xe 4,5 m.
Phải loại trùng trước khi chia vì cùng một chuyển động xuất hiện trong cả train và test sẽ làm điểm đánh giá lạc quan giả tạo, kể cả khi ID của chúng khác nhau.
Chia theo quỹ đạo giữ các khung tương quan và cửa sổ chồng lấn trong cùng một tập; chia ngẫu nhiên từng khung sẽ làm rò rỉ thông tin giữa train/test.

## Câu 2 · Phát lại

Theo `replay.json`, trên 600 đoạn, chạy lại cùng mã có 0 đoạn khác chính xác; refactor float32 khác chính xác ở cả 600 đoạn nhưng 0 đoạn vượt dung sai, sai khác tuyệt đối lớn nhất chỉ 0,0004031 m/s².
Tiêu chí so sánh là `abs(new − golden) <= 0.001 + 0.001 * abs(golden)` tại từng lệnh; hai NaN đệm cùng vị trí được coi bằng nhau. Sai số làm tròn và thứ tự phép toán giải thích việc khác chính xác nhưng vẫn đạt dung sai.
AEB làm 12 đoạn khác chính xác, trong đó 11 đoạn vượt dung sai, với sai khác lớn nhất 1,9764 m/s². Đọc đoạn 57 trong `work/replay_log.jsonl` và đối chiếu công thức IDMStream cho thấy các khung 98, 99, 101 có lệnh bị thay đổi.
Ở khung 98, gap = 4,3033 m, closing = 2,9189 m/s, TTC = 1,4743 s và lệnh đổi −5,0657 → −6 m/s²; khung 99 có TTC = 1,4333 s, lệnh −5,8820 → −6; khung 101 có TTC = 1,4896 s, lệnh −5,8156 → −6. Các chỉ số khung bắt đầu từ 0.
AEB tác động khi closing > 0,001 m/s, TTC < 1,5 s và lệnh gốc chưa phanh mạnh tới −6 m/s². Đây là thay đổi hành vi có điều kiện, khác với nhiễu làm tròn.
Trên 59.844 khung, phát nhanh có 0 khung trễ/rớt, còn thời gian thật có 583 khung trễ và 574 khung rớt, gây 376 đoạn vượt dung sai. Chi phí xử lý mô phỏng vượt deadline 100 ms hoặc làm bộ điều khiển bận khiến lệnh cũ bị giữ lại; phát nhanh không thể hiện vấn đề thời hạn này.

## Câu 3 · Kịch bản

Theo `scenarios.json`, lưới gồm 60 × 45 = 2.700 ô, cruise 14 m/s, gap từ 2,7601 đến 59,3123 m và closing từ 0 đến 14 m/s; khai thác được 614 sự kiện phanh, tương đương 6,3892% quỹ đạo sạch.
Với closing dương, xe mất thêm `dv * t_react` khoảng cách trong thời gian phản ứng và `dv² / (2 * b_max)` để triệt tiêu tốc độ tiến gần khi phanh hết mức; khoảng cách nhỏ hơn tổng này là bất khả thi, với b_max = 9 m/s².
Khi trễ tăng 0 → 0,6 s, tỷ lệ bất khả thi tăng 3,6296% → 10,2222%. Vùng đâm của IDM cũng tăng đúng như vậy, tức 6,5926 điểm phần trăm; diện tích tránh được nhưng IDM vẫn đâm bằng 0 ở cả hai trễ.
FullBrake cho 0 ô không khớp với biên vật lý ngoài vùng sát biên ở cả hai trễ; phép so được thực hiện trên lần lượt 2.682 và 2.662 ô đủ xa biên, cho thấy bộ mô phỏng phù hợp kiểm chứng vật lý này.
Giới hạn phanh mạnh theo phân vị 99,9% của người lái là 3,6248 m/s²; 12,6667% toàn lưới yêu cầu phanh mạnh hơn. Đây là giới hạn theo phân vị, không phải cực trị tuyệt đối của mọi mẫu.
Quan sát `scenarios_idm.png`, các chấm trắng tập trung ở closing thấp, chủ yếu dưới khoảng 4 m/s, còn vùng closing lớn và gap ngắn gần biên thất bại rất thưa. Do đó log bám xe chưa phủ tốt vùng cắt vào khó dù số cửa sổ huấn luyện lớn.

## Câu 4 · Vòng hở và vòng kín

Bảng dưới dùng L2 và tỷ lệ va chạm trung bình qua seed, đúng với `closedloop.json/ranking`; bảng kết quả cuối phía trên dùng dải seed để thể hiện độ dao động.

| Bộ điều khiển | L2 trung bình (m) | Hạng vòng hở | Va chạm trung bình (%) | Hạng vòng kín |
|---|---|---|---|---|
| const-vel | 1,2595 | 3 | 18,9733 | 3 |
| IDM | 1,5118 | 4 | 0,0694 | 1 |
| EgoMLP | 0,7518 | 2 | 27,0135 | 4 |
| PercMLP | 0,6195 | 1 | 1,2764 | 2 |

Có 4 cặp đảo thứ tự: const-vel/IDM, const-vel/EgoMLP, IDM/EgoMLP và IDM/PercMLP. Đặc biệt, IDM đứng cuối theo L2 nhưng đầu theo tỷ lệ va chạm, chỉ 2/2.883 quỹ đạo bị đâm.
L2 đo mức giống người lái trên trạng thái trong log ở các mốc 1, 2, 3 s; IDM giữ khoảng cách theo luật phản hồi và dự báo xe dẫn giữ tốc độ, nên sai khác quỹ đạo người lái không đồng nghĩa điều khiển kém an toàn.
EgoMLP có L2 tốt hơn const-vel nhưng thiếu khoảng cách và tốc độ xe dẫn, nên không thể phản ứng đầy đủ khi xe trước phanh; vòng kín còn tích luỹ sai số do tự tạo trạng thái mới.
Lát “xe dẫn sắp phanh” chỉ chiếm 3,9512% cửa sổ test. L2 toàn bộ/lát phanh lần lượt là const-vel 1,2595/1,9043; IDM 1,5118/1,5635; EgoMLP 0,7518/1,1030; PercMLP 0,6195/0,8613 m: trung bình toàn bộ che bớt lỗi ở lát khó.

## Câu 5 · Một lần chạy chưa đủ

Theo `closedloop.json/logs/EgoMLP/per_seed`, tỷ lệ va chạm của 5 seed 1684–1688 lần lượt là 22,5460%; 45,1960%; 21,1932%; 17,2390%; 28,8935%, dao động 27,9570 điểm phần trăm.
Seed tốt nhất 1687 có 497/2.883 quỹ đạo va chạm, CI Wilson 95% là [15,9040%; 18,6612%]; seed xấu nhất 1685 có 1.303/2.883 ca, CI là [43,3868%; 47,0179%].
Hai khoảng không chồng lấn, cho thấy biến động seed lớn hơn độ bất định lấy mẫu được thể hiện ở mỗi khoảng; tuy nhiên đây không thay thế phép kiểm định cặp trên cùng quỹ đạo hay đánh giá trên tập dữ liệu mới.
Nếu chỉ báo seed 1687, EgoMLP có thể trông tốt hơn const-vel 18,9733%; nếu chỉ báo seed 1685 thì nó tệ hơn rất nhiều. Trong khi đó L2 của EgoMLP chỉ dao động 0,7501–0,7554 m, không cảnh báo rõ mức bất ổn vòng kín.
Wilson phản ánh bất định của tỷ lệ quan sát trên tập quỹ đạo hữu hạn, không bao quát toàn bộ thiên lệch dữ liệu và mô phỏng; ngay cả khi quan sát 0 va chạm, cận trên vẫn có thể dương.
Tôi báo toàn bộ dải seed và CI, còn cổng lấy seed xấu nhất. Các seed cùng dùng một test set nên không được coi là những bộ thử độc lập; cần thêm quỹ đạo và bối cảnh giữ lại để kiểm tra tổng quát hoá.

## Câu 6 · Tác nhân phản ứng

Theo `agents.json`, T hiệu chỉnh là 1,2 s, cho khoảng cách Wasserstein giữa phân bố headway bằng 0,3481, thấp hơn 0,8173 của xe không phản ứng; trung vị headway người lái là 1,8327 s. T là tham số IDM, không phải trực tiếp trung vị headway.
Bảng sau lấy từ `rear_comparison_t_react_0.6`, đơn vị là phần trăm toàn bộ lưới; xe sau có khoảng cách ban đầu 18,8 m và trễ phản ứng 0,6 s. Các mô hình học được tổng hợp theo cách lấy trường hợp xấu nhất của bộ khung.

| Bộ điều khiển | Tổng va chạm, xe sau phát lại (%) | Tổng va chạm, xe sau phản ứng (%) | Va chạm sau, phát lại (%) | Va chạm sau, phản ứng (%) |
|---|---|---|---|---|
| const-vel | 80,6296 | 80,6296 | 0 | 0 |
| IDM | 82,9630 | 10,2222 | 72,7407 | 0 |
| EgoMLP | 79,3704 | 79,3704 | 0 | 0 |
| PercMLP | 76,9630 | 39,8519 | 37,1111 | 0 |
| PercMLP+ | 82,0000 | 11,0741 | 71,2963 | 0 |

Xe sau phát lại giữ tốc độ dù xe thử đã phanh để tránh xe cắt vào, khiến IDM bị tính nhiều va chạm phía sau. Trong thế giới này IDM trông kém const-vel: 82,9630% so với 80,6296%; khi xe sau phản ứng, IDM tốt hơn rõ: 10,2222% so với 80,6296%.
Đây là va chạm gây ra bởi giả định thiếu phản ứng của mô phỏng; không thể suy ra mọi va chạm sau đều “giả” ngoài thực tế. Hiệu chỉnh headway chỉ làm xe sau gần dữ liệu hơn theo một tiêu chí, chưa chứng minh phản ứng của nó đại diện mọi người lái.

## Câu 7 · Cổng phát hành và sửa

Theo `gate.json`, const-vel bị chặn bởi R2, R3, R4, R6; EgoMLP và PercMLP đều bị chặn bởi R2–R6. R1 và R7 đạt ở mọi ứng viên; R7 có 614 sự kiện phanh và 2.700 ô, vượt ngưỡng 100 và 2.000.
Với PercMLP, R2 có cận trên CI95 3,4402% > 0,0694% + 1 điểm; R3 có cận trên lát phanh 16,2451% > 0,5682% + 5 điểm; R5 có dải va chạm rộng 2,2199 điểm, vượt giới hạn 1 điểm. Các lỗi trên log phù hợp với vấn đề tích luỹ sai số khi mô hình tự lái, nhưng không tự chứng minh duy nhất nguyên nhân đó.
R4 có diện tích tránh được nhưng vẫn đâm 29,6296% > 0% + 2 điểm; R6 có tổng va chạm trong thế giới phản ứng 39,8519% > 10,2222% + 2 điểm. Kết hợp độ phủ log thưa ở vùng khó, đây là bằng chứng cho khoảng trống tình huống cắt vào.
Cách sửa dùng 20.000 tình huống tổng hợp cắt vào có nhãn IDM và 2 vòng DAgger: thu trạng thái mô hình tự đi vào trên train rồi gắn dự báo IDM 3 s tại đúng trạng thái đó. Hai phần lần lượt bổ sung độ phủ tình huống và trạng thái lệch khỏi log; do chạy kết hợp nên chưa tách riêng mức đóng góp của từng phần bằng ablation.
Sau sửa, R2–R6 đều chuyển sang qua, PercMLP+ đạt 7/7. Seed xấu nhất còn 10/2.883 ca va chạm (0,3469%, cận trên 0,6374%); cận trên lát phanh 3,1478%; diện tích R4 còn 0,8519%; độ rộng dải seed 0,0694 điểm và tổng va chạm reactive còn 11,0741%.
L2 tăng 0,6195 → 0,8034 m dù va chạm vòng kín giảm: điểm bắt chước vòng hở không đủ để ra quyết định an toàn. PercMLP+ qua cổng lab nhưng chưa tốt hơn IDM, vốn chỉ có 2 ca đâm trên log và diện tích R4 bằng 0; học từ IDM cũng không tự chứng minh vượt thầy.
Ở cruise giữ lại 20 m/s trong `fix.json`, diện tích quy trách nhiệm của PercMLP+ còn 0,3704–4,0370%, IDM vẫn 0%. Vì vậy chưa nên thay IDM ngoài thực tế; cần kiểm thử độc lập trên tốc độ và bối cảnh mới, thêm nhiễu/trễ thực, chạy shadow và cơ chế dự phòng trước quyết định phát hành.

## Câu 8 · Bài lab này không chứng minh được gì

Thứ nhất, mô phỏng chỉ xét chuyển động dọc 1 chiều, không đánh giá đánh lái, người đi bộ hoặc giao lộ; cần mở rộng sang tình huống 2D và tương tác nhiều bên, vì tránh đâm trong một trục chưa thể đại diện khả năng lái thực.
Thứ hai, quan sát lý tưởng và nhiễu đơn giản chưa đại diện đầy đủ lỗi cảm biến, mất dấu hay trễ tính toán biến thiên. Khi thêm nhiễu gap 1 m và tốc độ xe dẫn 0,5 m/s, PercMLP+ có va chạm 0,3122–0,4509%, so với 0,2775–0,3469% khi không nhiễu; cần log cảm biến và kiểm thử thời gian thực với lỗi có cấu trúc.
Thứ ba, xe dẫn phát lại không phản ứng với xe thử, còn xe sau IDM chỉ đại diện một họ hành vi. Tối ưu headway ở T = 1,2 s chưa xác nhận khả năng phản ứng khi phanh gấp; cần thử nhiều kiểu người lái và hiệu chỉnh trên dữ liệu độc lập.
Thứ tư, 2.700 ô và 614 ca phanh không chứng minh độ phủ tình huống hiếm. Tại cruise giữ lại 20 m/s, PercMLP+ có tới 4,0370% diện tích tránh được nhưng vẫn đâm, trong khi tại lưới 14 m/s và trễ 0,6 s chỉ 0,8519%; ngoài ra ở trễ 0 s tại 14 m/s, diện tích này vẫn tới 4,6296%, nhưng R4 chỉ xét 0,6 s. Cần thử nhiều tốc độ, độ trễ và ca sát biên, đồng thời giữ tập kiểm thử mới độc lập với quá trình sửa mô hình.
Thứ năm, 5 seed và CI95 không loại bỏ thiên lệch mô phỏng hoặc rủi ro hiếm, và mô hình học từ IDM chưa được chứng minh vượt thầy. Cần tăng mức phơi nhiễm, xác định điều kiện vận hành, kiểm tra tiện nghi/tiến độ và chạy shadow có giám sát; kết luận phù hợp là qua cổng lab, chưa đủ bằng chứng phát hành thực tế.
