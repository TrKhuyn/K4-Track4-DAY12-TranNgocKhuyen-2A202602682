# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Trần Ngọc Khuyến |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 2A202602682 |
| GitHub | TrKhuyn |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | 3177 / 4465 |
| Số seed huấn luyện bạn dùng (`--seeds`) | 5 (đánh giá chính) / 3 (bước sửa DAgger) |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.276 | 20.88% (CI95: 19.44% – 22.40%) | BLOCK |
| IDM | 1.540 | 0.07% (CI95: 0.02% – 0.25%) | (baseline) |
| EgoMLP | 0.754 – 0.763 (tb: 0.759) | 19.60% – 26.71% (tb: 23.60%) | BLOCK |
| PercMLP | 0.617 – 0.624 (tb: 0.620) | 0.49% – 1.32% (tb: 0.80%) | BLOCK |
| PercMLP+ (sau khi sửa) | 0.817 (tb) | 0.17% – 0.24% (tb: 0.20%) | RELEASE |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Trong bước kiểm toán, tập dữ liệu gốc gồm 992.679 dòng và 9.993 quỹ đạo bộc lộ hai vấn đề lớn: tồn tại 383 quỹ đạo trùng lặp hoàn toàn nằm rải rác ở các ID xa nhau, và cột gia tốc xe trước `Acc_LV` gần như vô dụng vì độ tương quan tuyến tính với đạo hàm vận tốc $d(v)/dt$ chỉ đạt vỏn vẹn 0.045 (trong khi cột xe sau đạt 0.929). Do cảm biến gia tốc xe trước bị nhiễu nặng hoặc ghi nhận sai, tôi đã loại bỏ cột này và thay thế bằng cách làm mịn vận tốc `Speed_LV` qua bộ lọc Gaussian rồi lấy vi phân bậc một $dv/dt$. Sau khi làm sạch, còn lại 9.610 quỹ đạo độc lập. Việc chia tập train/test bắt buộc phải thực hiện ở cấp độ quỹ đạo thay vì cấp độ frame ngẫu nhiên, nhằm triệt tiêu hiện tượng rò rỉ dữ liệu (data leakage) do tính tự tương quan thời gian rất cao giữa các khung hình liên tiếp trong cùng một tình huống lái xe.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Khi refactor sang float32, toàn bộ 600/600 đoạn đều bị lệch bit (`not_bit_identical: 600`) do sai số làm tròn và thứ tự tích lũy dấu phẩy động (độ chênh tối đa $2.34 \times 10^{-5}\text{ m/s}^2$), nhưng không có đoạn nào vượt ngưỡng dung sai (`differ_beyond_tol: 0` với atol=rtol=0.001). Khi điều chỉnh ngưỡng AEB nhạy hơn, có đúng 9/600 đoạn bị lệch vượt dung sai (sai lệch lớn nhất tới 1.98 m/s²). Khi mở các đoạn này (như đoạn 56, 62, 116), sự phân kỳ xuất hiện chính xác tại các frame có khoảng cách dồn lại quá gần hoặc TTC giảm sâu: thuật toán AEB mới kích hoạt phanh khẩn cấp sớm hơn phản xạ thực tế của người lái trong log. Cuối cùng, việc phát lại theo thời gian thực (realtime) đã làm lộ ra 579 frame bị rớt và 583 frame trễ nhịp do nghẽn I/O và áp lực chu kỳ điều khiển, khiến 375/600 đoạn bị trôi lệch nghiêm trọng — đây là rủi ro trễ thời gian thực mà chế độ chạy nhanh nhất (fastest) đã hoàn toàn che giấu.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới kịch bản cut-in gồm 2.700 ô (90 mức khoảng cách từ 2.76 m đến 59.31 m, 30 mức vận tốc tương đối từ 0 đến 14.0 m/s tại vận tốc cruise $v_{ego} = 14.0\text{ m/s}$). Ở độ trễ bằng 0, có 3.63% số ô là vùng va chạm bất khả kháng về mặt vật lý (kể cả phanh hết lực vẫn đâm); khi tăng độ trễ phản ứng lên 0.6 s, vùng không thể tránh này mở rộng lên 10.22% (tức tăng thêm 6.59 điểm phần trăm). Vùng đâm của IDM tăng tương ứng từ 3.63% lên 10.22%, nhưng diện tích va chạm quy kết do thuật toán (`attributable_area`) vẫn duy trì ở mức 0.0%, nghĩa là IDM không gây thêm bất kỳ vụ đâm nào ngoài giới hạn vật lý. Dữ liệu log thực tế bao phủ rất kém các vùng hiểm nghèo này: các pha phanh gấp chỉ chiếm 6.39% log, và ngưỡng phanh mạnh nhất của con người ở phân vị 99.9% chỉ đạt 3.62 m/s² (p99 là 4.21 m/s²). Trong khi đó, lưới kịch bản có tới 12.67% số ô đòi hỏi gia tốc phanh vượt qua mức 3.62 m/s², chứng tỏ con người trong log hầu như không bao giờ tự đẩy mình vào các tình huống ngặt nghèo như thử nghiệm.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

| Bộ điều khiển | L2 vòng hở (m) | Hạng vòng hở | Tỷ lệ va chạm vòng kín (%) | Hạng vòng kín |
|---|---|---|---|---|
| PercMLP | 0.620 | #1 | 0.80% | #2 |
| EgoMLP | 0.759 | #2 | 23.60% | #4 |
| constant-velocity | 1.276 | #3 | 20.88% | #3 |
| IDM | 1.540 | #4 | 0.07% | #1 |

Hiện tượng đảo ngược thứ hạng diễn ra ở 4 cặp: (const-vel, IDM), (const-vel, EgoMLP), (IDM, EgoMLP) và (IDM, PercMLP). Điển hình nhất là IDM xếp bét bảng vòng hở nhưng lại đứng đầu vòng kín; ngược lại EgoMLP đứng nhì vòng hở nhưng lại đội sổ vòng kín. IDM có L2 kém (1.540 m) vì nó bám theo mô hình hành vi vật lý lý tưởng chứ không học vẹt từng thói quen nhỏ của tài xế con người trong log; tuy nhiên ở vòng kín, nhờ vòng lặp phản hồi khoảng cách (feedback loop) chuẩn xác, IDM triệt tiêu sai số tích lũy và phanh an toàn tuyệt đối (chỉ đâm 0.07%). Ngược lại, EgoMLP mù hoàn toàn thông tin xe trước (không có perception), nó đạt L2 tốt ở vòng hở chỉ vì đoán mò giá trị trung bình trên các đoạn xe chạy êm. Khi đưa vào vòng kín, gặp hiện tượng trôi dạt phân phối (distributional drift), EgoMLP không hề biết có xe cản phía trước để phanh và tiếp tục giữ ga, dẫn đến tỷ lệ đâm lên tới 23.60% (thậm chí 59.78% ở lát cắt phanh gấp, tệ hơn cả xe chạy đều).

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

Qua 5 seed huấn luyện ngẫu nhiên, tỷ lệ va chạm vòng kín của EgoMLP dao động rất dữ dội từ 19.60% (seed 4465) đến 26.71% (seed 4469), độ phân tán lên tới 7.11 điểm phần trăm — vi phạm nghiêm trọng quy tắc ổn định R5 (chỉ cho phép lệch $\le 1\%$). Khoảng tin cậy Wilson 95% của seed tốt nhất là [18.19%, 21.09%] hoàn toàn không giao thoa với khoảng tin cậy của seed kém nhất là [25.13%, 28.35%], khẳng định khoảng cách này là do tính bất ổn nội tại của mô hình học sâu chứ không phải do biến động ngẫu nhiên khi lấy mẫu. Nếu chỉ chạy duy nhất một seed, người phát triển có thể may mắn bốc trúng seed 4465 và báo cáo kết quả khả quan giả tạo (19.60%), bỏ qua việc mô hình có thể đâm tới 26.71% (cận trên CI chạm 28.35%) ở seed khác. Việc "cherry-picking" một lần chạy duy nhất trong hệ thống xe tự hành là vô cùng nguy hiểm vì nó che giấu điểm mù phân phối và tạo ra ảo giác an toàn chết người.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

| Bộ điều khiển (t_react = 0.6 s) | Xe sau phát lại: Bị tông sau (%) | Xe sau phát lại: Tổng va chạm (%) | Xe sau phản ứng: Bị tông sau (%) | Xe sau phản ứng: Tổng va chạm (%) |
|---|---|---|---|---|
| constant-velocity | 0.00% | 80.63% | 0.00% | 80.63% |
| IDM | 72.74% | 82.96% | 0.00% | 10.22% |
| EgoMLP | 0.00% | 84.11% | 0.00% | 84.11% |
| PercMLP | 50.44% | 80.52% | 0.00% | 31.37% |
| PercMLP+ | 72.41% | 83.15% | 0.00% | 10.74% |

Tham số headway thời gian tối ưu hiệu chỉnh được là $T = 1.2\text{ s}$, cho khoảng cách Wasserstein $W_1 = 0.344\text{ m}$ so với phân bố khoảng cách bám đuôi của người lái (thực tế trung vị headway con người trong log là 1.84 s), vượt trội so với tác nhân không phản ứng ($W_1 = 0.854\text{ m}$). Khi xe sau chỉ "phát lại mù quáng", nó lặp lại y nguyên quỹ đạo ga của tài xế trong quá khứ mà không phanh theo xe ego. Vì IDM nhận biết nguy hiểm phía trước và chủ động phanh gấp để tránh đâm, xe sau phát lại đã lao thẳng vào đuôi xe ego, khiến IDM phải chịu tới 72.74% "va chạm giả" từ phía sau và đẩy tổng va chạm lên tới 82.96%. Đánh giá trong môi trường tĩnh này dẫn tới kết luận sai lầm rằng IDM nguy hiểm ngang ngửa xe chạy bừa. Khi chuyển sang xe sau biết phản ứng (reactive), va chạm sau của IDM giảm ngay về 0.00% và tổng va chạm trở về đúng mức 10.22% bất khả kháng, chứng minh IDM hoàn toàn an toàn và lỗi thuộc về xe sau không chú ý quan sát.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

PercMLP ban đầu bị cổng an toàn từ chối (BLOCK) vì trượt 4 quy tắc: R2 (cận trên CI95 vòng kín đạt 1.80% > 1.1%), R3 (va chạm lát cắt phanh đạt 5.43% > 6.1%), R4 (diện tích va chạm quy kết trên lưới đạt 21.15% > 2.0%) và R6 (va chạm với xe sau phản ứng đạt 31.37% > 12.22%). Nguyên nhân gốc là mô hình chỉ học bắt chước (Imitation Learning) trên tập log thông thường — nơi các tình huống phanh khẩn cấp chỉ chiếm 4.19%. Khi gặp các pha cut-in ngặt nghèo ngoài phân phối (OOD), mô hình chưa từng được thấy trạng thái này nên ra lệnh phanh yếu, gây tích lũy sai số. Tôi đã khắc phục bằng thuật toán DAgger kết hợp 20.000 kịch bản cut-in tổng hợp có chuyên gia IDM gán nhãn can thiệp: cách này chữa đúng bệnh vì đưa trực tiếp các trạng thái lệch pha vào dữ liệu huấn luyện và buộc mạng nơ-ron học cách tự cứu vãn. Dù PercMLP+ đã vượt qua cả 7 quy tắc để nhận quyết định RELEASE (va chạm trên log giảm còn 0.20%, R4 còn 0.52%), nó vẫn **chưa tốt hơn** IDM (IDM chỉ đâm 0.07% trên log và attributable area là 0.00% tuyệt đối). PercMLP+ chưa nên phát hành ra đường thực ngay; tôi đề xuất phải bọc thêm tầng bảo vệ an toàn vật lý (Safety Shielding/CBF dựa trên IDM/TTC) để ghi đè khi mạng nơ-ron ra quyết định nguy hiểm, đồng thời thử nghiệm trên simulator vật lý đa vật thể độ chi tiết cao trước khi chạy thực địa.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

Ba giới hạn quan trọng nhất trong bài lab khiến kết luận phát hành chưa thể áp dụng trực tiếp ra thế giới thực gồm:
1. **Động lực học xe bị lý tưởng hóa dạng chất điểm 1D (Point-mass dynamics)**: Mô phỏng giả định gia tốc thay đổi tức thời theo lệnh điều khiển và chỉ chuyển động dọc, bỏ qua độ trễ cơ khí phanh (actuator delay ~100–200ms), độ giật (jerk), quán tính truyền động và sự biến thiên ma sát mặt đường (trời mưa, đường trơn trượt).
2. **Cảm biến và module nhận thức hoàn hảo**: Mô hình nhận đầu vào khoảng cách và vận tốc tương đối trực tiếp từ ground truth của simulator. Trên xe tự hành thực tế, camera và radar thường xuyên gặp nhiễu, sai số đo lường, độ trễ xử lý, phát hiện vật cản ảo (ghost braking) hoặc mất dấu mục tiêu do điểm mù và che khuất (occlusion).
3. **Môi trường giao thông đơn giản hóa, không có chuyển động ngang và tương tác xã hội**: Thử nghiệm chỉ bó hẹp trong kịch bản bám đuôi thẳng trên 1 làn đường. Thực tế giao thông phức tạp hơn nhiều với các tình huống chuyển làn bất ngờ, tạt đầu cắt ngang, người đi bộ và tính tương tác đa tác nhân (game-theoretic interaction) mà mô hình 1D không thể phản ánh.

Để thu hẹp khoảng cách thực tế, bước tiếp theo cần chuyển sang môi trường mô phỏng vật lý đa vật thể độ phân giải cao (như CARLA hoặc IPG CarMaker), tích hợp pipeline nhận thức trực tiếp từ camera/LiDAR, áp dụng bộ lọc an toàn hình thức (Safety Filter / Control Barrier Functions), và tiến hành thử nghiệm trên sa hình kín với mô hình xe vật lý trước khi triển khai có tài xế an toàn.
