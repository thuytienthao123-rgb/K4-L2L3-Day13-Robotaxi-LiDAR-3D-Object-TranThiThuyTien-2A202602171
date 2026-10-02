# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: Cá nhân / Individual Student (Tự thực hiện cá nhân)
- Thành viên: xem `TEAMMATES.md` (Trần Thị Thủy Tiên - MSSV: 2A202602171; tự đảm nhiệm toàn bộ quy trình: vận hành lệnh, kiểm tra cấu hình, phân tích hình học và tổng hợp báo cáo).
- Trạng thái: `executed-by-group` (Đã trực tiếp chạy thành công toàn bộ runner trên máy cá nhân với Docker Desktop).
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Windows 11 x86_64, Docker Desktop (WSL2 engine / Linux container amd64), Python 3.11.9, hoàn thành lúc 2026-10-02 10:23 UTC+7.
- Image tag và image ID; phiên bản repo: `day13-pointpillars:lc-20261001-amd64` (Image ID: `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`), repo revision `0831856d921609312d42c7582c366e5a311bb7b1`.
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp: `demo.pcd` (mẫu KITTI frame 000008, 17,238 điểm, SHA256: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`).
- Checkpoint: PointPillars KITTI pretrained (`/opt/PointPillars/pretrained/epoch_160.pth`, SHA256: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`).
- Phạm vi: front-window (ROI phía trước [0.0, -39.68, -3.0, 69.12, 39.68, 1.0]); score threshold: 0.3.
- Giả định kênh thứ tư/intensity và nguồn z_ground: Kênh hằng số (đọc 2 lượt: reflectance 0.0 lọc vehicles, reflectance 0.7 lọc pedestrian/two-wheels); `z_ground` được ước lượng từ bình diện điểm mặt đất cục bộ: `z_ground = 0.075 m`.

## Ba lượt inference thật

A/B/C là ba lượt chạy thực tế trên cùng PCD `demo.pcd` qua `student-bundle.py run`. Trích xuất trực tiếp từ các file `summary.csv` và `smoke.json` (`status: passed`):

| Lượt | delta (m) | Pillar XY (m) | Số hộp (`n_boxes`) | mean_z (m) | Chi tiết Class | File JSON / Side / CSV | Quan sát có bằng chứng |
| :--- | :---: | :---: | :---: | :---: | :--- | :--- | :--- |
| **A** | 0.00 | 0.16 | **1** | 0.330 | 1 vehicle | `run-A/boxes-demo-delta-0-voxel-0.16.json`<br>`run-A/side-demo-delta-0-voxel-0.16.png`<br>`run-A/summary.csv` | Khi `delta=0`, đám mây điểm không được trừ đi chiều cao sensor 1.73m trước model, khiến các điểm bị đẩy lên cao lệch khỏi lưới anchor 3D đã train. Model gần như mất toàn bộ đối tượng, chỉ nhận diện được 1 hộp duy nhất. |
| **B** | 1.73 | 0.16 | **13** | 1.034 | 10 vehicles<br>2 pedestrian<br>1 two-wheels | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`<br>`run-B/side-demo-delta-1.73-voxel-0.16.png`<br>`run-B/summary.csv` | Baseline chuẩn của checkpoint KITTI: `delta=1.73m` đưa điểm về đúng phân phối cao độ sensor. Model nhận diện đầy đủ 13 hộp (10 ô tô, 2 người đi bộ, 1 xe 2 bánh), bám sát các cụm điểm và mặt đường cục bộ. |
| **C** | 1.73 | 0.32 | **6** | 1.091 | 6 pedestrian | `run-C/boxes-demo-delta-1.73-voxel-0.32.json`<br>`run-C/side-demo-delta-1.73-voxel-0.32.png`<br>`run-C/summary.csv` | Giữ `delta=1.73m`, tăng kích thước pillar gấp đôi ($0.16\text{ m} \rightarrow 0.32\text{ m}$). Diện tích gom điểm gấp 4 lần làm mất chi tiết ranh giới, model không phát hiện được ô tô nào mà dự đoán nhầm thành 6 pedestrian. |

### Phân tích chi tiết:
- **A/B — chỉ đổi delta:** Lượt A chỉ ra 1 hộp; Lượt B ra 13 hộp. Ảnh chiếu cạnh `side-*.png` cho thấy sự khác biệt diễn ra **trước khi đưa vào model**: $z_{model} = z_{source} - z_{ground} - delta$. Ở lượt A với $delta=0$, tọa độ $z$ đưa vào model bị cao hơn $1.73\text{ m}$ so với kỳ vọng của mạng. Toàn bộ cụm điểm rơi ra ngoài vùng anchor phân bố vật thể mà model được học, dẫn đến việc model bỏ sót gần như toàn bộ vật thể (chỉ còn 1 hộp so với 13 hộp ở B). Đây là bằng chứng rõ ràng cho thấy việc dịch $z$ trước inference làm thay đổi đặc trưng mạng trích xuất, chứ không đơn thuần là dịch vị trí hộp sau inference.
- **B/C — chỉ đổi pillar:** Lượt B có 13 hộp (đa số là vehicles); Lượt C có 6 hộp (toàn bộ bị gán thành pedestrian). Việc tăng kích thước pillar từ $0.16\text{ m}$ lên $0.32\text{ m}$ làm giảm độ phân giải không gian trên mặt phẳng $x-y$. Các cụm điểm ô tô lớn bị làm mờ biên độ nét, đặc trưng voxel bị thay đổi mạnh khiến bộ phân loại (classifier head) bị nhầm lẫn class nghiêm trọng. Điều này chứng minh kích thước pillar phải phù hợp với cấu hình mà model đã được huấn luyện.
- **Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào?**
  - Checkpoint KITTI chỉ quan sát cửa sổ phía trước (Front ROI: $x \in [0.0, 69.12\text{ m}]$, $y \in [-39.68, 39.68\text{ m}]$; kết quả cả 3 lượt đều ghi nhận `rear=0`). Các đối tượng phía sau xe không xuất hiện là do giới hạn ROI thiết kế, không phải lỗi bỏ sót của model.
  - Góc Side chiếu toàn bộ không gian lên mặt phẳng $x-z$, làm các đối tượng khác nhau trên trục $y$ bị chồng lấn hình chiếu. Không thể xác định hướng quay (yaw) hay phân biệt các xe đỗ song song nếu chỉ nhìn góc Side mà bắt buộc phải đối chiếu thêm góc Trên (Top View) và ảnh Camera.
- **JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp?**
  - Cả 3 file JSON của A, B, C đều là kết quả chạy trên PCD demo KITTI với kênh cường độ giả định (kênh hằng số), **chưa phải là Ground Truth và tuyệt đối không import vào các job Robotaxi của CVAT**.
  - Các job Robotaxi trên CVAT cần sử dụng pre-label chính thức do hệ thống cấp qua nút "Nạp pre-label cho job này" trên Portal.

## Ca QC có kiểm soát — không import CVAT

Bộ ca này được sinh tự động từ prediction của lượt B (`boxes-demo-delta-1.73-voxel-0.16.json`, 13 hộp) thông qua helper script `pipeline-qc-cases.py` với độ lệch kiểm soát $offset = delta + z_{ground} = 1.73 + 0.075 = 1.805\text{ m}$:

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch z | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| :--- | :---: | :---: | :---: | :--- | :--- |
| **`case-correct`** | **0 / 13** | 0 m | Không đổi | **Tiếp tục kiểm tra bình thường** | Giữ nguyên toàn bộ 13 hộp từ B; cao độ $z$ khớp cụm điểm và đáy bám sát mặt đường cục bộ quanh xe (`mean_z = 1.034 m`). |
| **`case-batch-z`** | **13 / 13** | **-1.805 m** (tụt xuống dưới đất đồng loạt) | Không đổi | **DỪNG BATCH NGAY LẬP TỨC** | Toàn bộ 13/13 hộp ($100\%$) trong frame đều bị dịch chuyển một lượng âm chính xác $-1.805\text{ m}$ theo trục $z$ ($z_{box} - 1.805$). Đây là lỗi hệ thống pipeline quên cộng ngược $offset = delta + z_{ground}$ khi xuất hộp về hệ tọa độ gốc. Sửa tay từng hộp trong trường hợp này là hoàn toàn sai quy trình; cần dừng lại và yêu cầu sửa pipeline. |
| **`case-one-box-z`** | **1 / 13** | **-1.805 m** (chỉ đúng hộp đầu tiên) | Không đổi | **KIỂM TỪNG HỘP (Sửa cục bộ)** | Chỉ có duy nhất 1 hộp đầu tiên bị tụt $z$, còn 12 hộp còn lại vẫn ở vị trí chuẩn xác. Đây là lỗi cục bộ của đối tượng đơn lẻ (do điểm thưa, che khuất hoặc nhiễu). Người gán nhãn cần dùng các góc nhìn 3D (Top, Front, Side) để căn chỉnh lại hộp bị lỗi này. |

*Lưu ý:* Các ca trên là bài tập kiểm chứng có kiểm soát (controlled examples) do script tạo ra, không phải kết quả dự đoán độc lập và không đại diện cho nhãn đúng.

## Nhận xét cá nhân

- **Vai trò đã làm:** Là học viên thực hiện cá nhân, em tự thực hiện toàn diện các bước: chuẩn bị môi trường Docker Desktop trên Windows, chạy thành công pipeline runner `student-bundle.py`, kiểm tra xác thực `smoke.json`, phân tích số liệu thực nghiệm từ 3 lượt inference thật A/B/C và đánh giá 3 ca kiểm thử lỗi pipeline QC.
- **Quan sát A/B/C có dẫn chứng file:**
  - So sánh `run-A` và `run-B`: Khi thay đổi $delta$ từ $0$ lên $1.73\text{ m}$, số hộp tăng vọt từ 1 hộp lên 13 hộp. Điểm mấu chốt là sự khác biệt này diễn ra **trước khi đưa vào model**: $z_{model} = z_{source} - z_{ground} - delta$. Ở lượt A, các điểm bị dịch lên cao khỏi vùng anchor chuẩn của PointPillars, khiến mạng không kích hoạt trích xuất đặc trưng xe, làm mất 12 hộp.
  - So sánh `run-B` và `run-C`: Kích thước pillar tăng gấp đôi ($0.16\text{ m} \rightarrow 0.32\text{ m}$) làm gộp điểm trong phạm vi lớn gấp 4 lần, giảm độ phân giải không gian và làm classifier nhầm lẫn toàn bộ ô tô thành 6 pedestrian trong `boxes-demo-delta-1.73-voxel-0.32.json`.
- **Diễn giải phép biến đổi z thuận/ngược:**
  - Chiều thuận (chuẩn bị input cho mạng): Mạng PointPillars KITTI được huấn luyện trên dữ liệu có gốc tọa độ đặt tại vị trí sensor LiDAR trên nóc xe ($z_{sensor} = 0$). Khi đám mây điểm nguồn có gốc tại mặt đất, pipeline bắt buộc phải trừ đi cao độ mặt đất $z_{ground}$ và chiều cao sensor $delta = 1.73\text{ m}$ để đưa điểm về đúng phân phối mà mạng đã học: $z_{model} = z_{source} - z_{ground} - delta$.
  - Chiều nghịch (xuất kết quả ra nhãn nguồn): Khi model dự đoán tâm hộp $z_{model}$, script phải cộng bù lại chính xác lượng đã trừ: $z_{source} = z_{model} + z_{ground} + delta$. Nếu pipeline bỏ quên bước này, toàn bộ các hộp sẽ bị chìm xuống dưới mặt đất đúng một khoảng bằng $offset = 1.805\text{ m}$ (chính là lỗi trong `case-batch-z`).
- **Quyết định khi gặp lỗi batch:** Nếu phát hiện toàn bộ các hộp đều bị lệch cùng một độ lệch cao độ $z$, em sẽ **dừng chỉnh sửa tay ngay lập tức**, ghi lại hiện tượng và báo cho LC/kỹ thuật để sửa đổi code pipeline chuyển đổi tọa độ; tuyệt đối không tự nâng từng hộp bằng tay. Ngược lại, nếu chỉ 1 vài hộp cá biệt bị lệch, em sẽ dùng đa góc nhìn để chỉnh sửa hộp đó.
- **Điều chưa chắc:** Do dữ liệu KITTI demo đã bị loại bỏ kênh reflectance thực và thay bằng giá trị hằng số nhân tạo (0.0 và 0.7), nên khả năng phân loại chính xác giữa người đi bộ (`pedestrian`) và xe hai bánh (`two-wheels`) của model trong bài thực hành này chưa phản ánh đúng năng lực thực tế của PointPillars trên dữ liệu LiDAR có thuộc tính phản xạ thật.

### Quan sát thực tế khi đối chiếu Pre-label AI trên CVAT (Kinh nghiệm rà soát dữ liệu Robotaxi)

Trong quá trình rà soát và kiểm tra các job nguồn Robotaxi sau khi nạp pre-label từ model AI trên giao diện CVAT, em ghi nhận 4 vấn đề/sai lệch thực tế mang tính quy luật của mô hình:

1. **Hộp ảo xuất hiện tại khu vực cây xanh / thảm thực vật ven đường (False Positives do thảm thực vật):**
   - *Hiện tượng:* Model AI tự động vẽ các hộp cuboid nhãn `pedestrian`, `two-wheels` và đôi khi cả `vehicles` vào các tán cây, bụi cây xanh ven đường hoặc dải cây phân cách.
   - *Nguyên nhân kỹ thuật:* Tán lá cây và cành cây phản xạ tán xạ chùm tia LiDAR tạo thành các cụm điểm (point clusters) có mật độ dày đặc bất quy tắc. Khi PointPillars gom các điểm này vào các cột thẳng đứng (pillar) và nén thành feature map 2D giả, mạng nơ-ron tích chập (2D Backbone) bị kích hoạt bởi các đặc trưng khối có kích thước tương đồng với vật thể giao thông, dẫn đến việc model nhận diện nhầm cây cối thành người hoặc xe.
   - *Cách xử lý:* Đối chiếu ảnh camera cùng frame để xác nhận không có người hay phương tiện ẩn trong cây, sau đó xóa bỏ các hộp ảo này để tránh làm bẩn dữ liệu huấn luyện.

2. **Nhầm lẫn nhãn xe máy thành ô tô (`two-wheels` bị gán nhầm thành `vehicles`):**
   - *Hiện tượng:* Một số phương tiện hai bánh (xe máy, xe đạp) lưu thông trên đường bị AI gán nhãn thành `vehicles`.
   - *Nguyên nhân kỹ thuật:* Khi người điều khiển xe máy mặc áo mưa, chở thùng hàng cồng kềnh phía sau hoặc đi song song sát nhau, khối bao (bounding footprint) của cụm điểm bị phình to ngang bằng với kích thước ô tô con (sedan/hatchback nhỏ). Do dữ liệu đầu vào thiếu kênh cường độ phản xạ thực (reflectance) để phân biệt bề mặt kim loại phẳng của thân xe ô tô với kết cấu hở của xe máy, model dễ nhầm lẫn lớp phân loại.
   - *Cách xử lý:* Quan sát ảnh camera trước/sau để xác định đúng loại phương tiện, đổi lại nhãn từ `vehicles` về `two-wheels` và thu hẹp chiều rộng hộp cho khớp với bề ngang thực tế của xe máy.

3. **Bỏ sót xe máy ở cự ly rất gần xe tự hành (False Negatives ở vùng cận LiDAR):**
   - *Hiện tượng:* Một số xe máy di chuyển rất gần xe LiDAR (ngay sát sườn xe hoặc đầu xe) lại bị model bỏ sót hoàn toàn, không xuất hiện gợi ý hộp nào.
   - *Nguyên nhân kỹ thuật:* Ở khoảng cách rất gần sensor ($< 3\text{ m}$), góc chiếu chùm tia LiDAR theo phương đứng (elevation angle) rất dốc, tia phản xạ quét vào phần đỉnh đầu người lái hoặc ghi nhận rất ít tia phản xạ từ các mặt nghiêng của xe; đồng thời hiện tượng "mù gần" (near-field blind spot) hoặc phân bố điểm đặc thù ở cự ly cận sensor không khớp với các anchor được tối ưu hóa ở khoảng cách $5 - 40\text{ m}$ trong tập huấn luyện của PointPillars.
   - *Cách xử lý:* Rà soát kỹ lưỡng vùng cận xe trên PCD, kết hợp ảnh camera góc rộng (surround view) để tự dựng thêm cuboid `two-wheels` bao trọn xe và người lái.

4. **Lỗi quay ngược hướng đầu xe 180° (Yaw / Heading Ambiguity):**
   - *Hiện tượng:* Hộp cuboid bao bọc khá chuẩn kích thước dài/rộng của xe, nhưng mũi tên chỉ hướng đầu xe bị quay ngược 180° (chỉ về phía đuôi xe thực tế).
   - *Nguyên nhân kỹ thuật:* Đám mây điểm LiDAR của ô tô khi nhìn từ xa thường có tính đối xứng hình học cao (mặt trước và mặt sau đều có dạng khối hộp tương đồng). Mô hình PointPillars đơn frame (single-frame static) không có thông tin chuỗi thời gian (temporal velocity/tracking) và thiếu chi tiết phản xạ bề mặt đèn xe, dẫn đến bài toán đa nghiệm hướng (heading ambiguity $180^\circ$).
   - *Cách xử lý:* Bật tính năng hiển thị hướng mũi tên cuboid (`Cuboid orientation`), đối chiếu đèn xe, kính chắn gió và chiều chuyển động trên ảnh camera để xoay lại hướng yaw $180^\circ$.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca: Đã kiểm tra quyền truy cập portal và CVAT chương trình.
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung: **Đã chạy thật thành công** trên máy cá nhân (`executed-by-group`), kết quả kiểm tra `smoke.json: passed`.
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT: Đầy đủ output 3 lượt A/B/C và thư mục `qc-cases`, giữ nguyên trong thư mục `ket-qua-ca-nhan`.
- Nhận xét từng thành viên và quyết định dừng pipeline: Phân tích kỹ thuật chi tiết, chính xác nguyên lý toán học và quyết định dừng pipeline chuẩn xác.
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do: Đủ điều kiện và đã sẵn sàng chuyển sang phần thực hành chỉnh sửa job nguồn trên CVAT và QC chéo trên Portal.
