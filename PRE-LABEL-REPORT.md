# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: phòng C301
- Thành viên: làm cá nhân - Võ Quốc Dịnh, MSSV 2A202602318
- Trạng thái: `provided-results`
- Người thực sự chạy; ngày/giờ; hệ máy/architecture:  phân tích PCD/code ngày 02/10/2026 trên Windows 11 Pro, Python 3.14.2, không Docker.
- Image tag và image ID; phiên bản repo: image tag/ID **chưa có** (nằm trong `manifest.json` của ZIP Releases, em chưa tải). Repo ở commit `e226b93`; `VALIDATION.md` ghi gói được đóng từ base revision `0831856`, `working_tree_dirty: true`.
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp: `data/demo.pcd` — KITTI demo `000008` đã chuyển đổi (z +1,73 m, giữ x/y, bỏ reflectance, RGB=0), 17.238 điểm, `frame_id = demo`, giấy phép CC BY-NC-SA 3.0, được phép chạy trên laptop học viên. SHA256 em tự kiểm: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60` — khớp `data/provenance.json` và `VALIDATION.md`.
- Checkpoint: PointPillars KITTI có sẵn trong image, `/opt/PointPillars/pretrained/epoch_160.pth`; SHA256 theo `VALIDATION.md`: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1` (em chưa tự kiểm vì không có image).
- Phạm vi: front-window của preset KITTI (x 0→69,12 m; y −39,68→39,68 m; z_model −3→1 m), không `--full-scene`; score threshold: 0,3.
- Giả định kênh thứ tư/intensity và nguồn z_ground: PCD không có intensity thật; script đọc hai lần với reflectance hằng — 0,0 giữ `vehicles`, 0,7 giữ `pedestrian`/`two-wheels`. `z_ground` = đỉnh histogram z (bin 5 cm) của chính PCD; em tự tính ra **0,075 m**, khớp con số amd64 trong `data/ATTRIBUTION.md`.

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD. Runner chạy đủ ba lượt từ một lệnh. Lấy **Số hộp** từ `n_boxes`, **mean_z** từ `mean_z` trong `run-A/B/C/summary.csv`; không tự tính lại hoặc đoán. `mean_z` không phải điểm chất lượng. Mở `side-*.png`, đối chiếu `boxes-*.json` để ghi quan sát. Số hộp không phải đáp án cần khớp nhóm khác.

| Lượt | delta | Pillar XY | Số hộp                                     | mean_z                                 | File JSON/Side/CSV                                                            | Quan sát có bằng chứng                                                                                                                                                                                                              |
| ------ | ----- | --------- | -------------------------------------------- | -------------------------------------- | ----------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A      | 0     | 0.16      | 1 (theo`VALIDATION.md`)                    | chưa có — không có`summary.csv` | chưa có file; tên dự kiến`run-A/boxes-demo-delta-0-voxel-0.16.json`    | Với delta=0, mặt đất nằm ở z_model = 0 thay vì −1,73; em tính được 6.886/17.106 điểm trong cửa sổ XY bị cắt vì vượt z_model = 1 m, chỉ còn 10.220 điểm vào model. Lớp của hộp duy nhất: chưa có nguồn. |
| B      | 1.73  | 0.16      | 13 (10 vehicles, 2 pedestrian, 1 two-wheels) | chưa có                              | chưa có file; tên dự kiến`run-B/boxes-demo-delta-1.73-voxel-0.16.json` | Mặt đất ở z_model = −1,73 đúng giả định checkpoint; chỉ 177 điểm bị cắt trên, 1 điểm cắt dưới, 16.928 điểm vào model. Lưới 432×496, em đếm được 3.957 pillar có điểm (≈4,3 điểm/pillar).          |
| C      | 1.73  | 0.32      | 6 (cả 6 là pedestrian)                     | chưa có                              | chưa có file; tên dự kiến`run-C/boxes-demo-delta-1.73-voxel-0.32.json` | Cùng 16.928 điểm như B nhưng lưới 216×248, còn 1.898 pillar có điểm (≈8,9 điểm/pillar). Toàn bộ 10 vehicles của B không còn.                                                                                        |

- A/B — chỉ đổi delta: A có 1 hộp; B có 13 hộp (số của LC, `bundle/VALIDATION.md`). Em chưa có `side-*.png` nên không dẫn được vùng x cụ thể; bằng chứng em tự có là ở input: lượt A mất 6.886 điểm phía trên (≈40 % điểm trong cửa sổ XY) và phần còn lại lệch 1,73 m so với độ cao model đã học, lượt B gần như giữ nguyên scan. Đây là chạy lại model trên input khác, không chỉ dịch hộp cũ — nếu chỉ dịch hộp thì A vẫn phải có 13 hộp, cùng x/y, chỉ khác z. Điều em còn chưa chắc là hộp duy nhất của A thuộc lớp nào, có trùng vị trí với hộp nào của B không, và `mean_z` hai lượt chênh bao nhiêu (cần `boxes-*.json`/`summary.csv`).
- B/C — chỉ đổi pillar: B có 13 hộp; C có 6 hộp. Chưa có ảnh Side để chỉ vùng khác nhau. Số lượng/lớp/vị trí thay đổi như sau: số hộp giảm 13 → 6; vehicles 10 → 0, two-wheels 1 → 0, pedestrian 2 → 6; vị trí từng hộp chưa đối chiếu được. Có đủ bằng chứng để kết luận tốt hơn không? Không. C dùng lại checkpoint train với pillar 0,16 m nên lưới BEV đổi kích thước so với lúc train; mất hết vehicles và tăng pedestrian trông giống hệ quả lệch biểu diễn hơn là "phát hiện tốt hơn", nhưng không có nhãn tham chiếu nên em không kết luận B đúng hơn, chỉ ghi B là mốc so sánh.
- Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào? ROI chỉ lấy phía trước (x > 0, |y| < 39,68 m) nên vật phía sau hoặc ngoài cửa sổ không phải bằng chứng model bỏ sót; scan này có x từ 2,89 đến 76,83 m nên phần x > 69,12 m cũng bị loại. Side là hình chiếu x–z của cả scene: các vật khác y chồng lên nhau, không thấy được y, chiều rộng và yaw, nên chỉ dùng để đọc lệch z/đáy hộp; muốn nói miss hay sai hướng phải xem thêm Top/Front và camera.
- JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp? Không JSON nào được import: cả ba là prediction KITTI demo, khác frame Robotaxi. Riêng về độ tin: A (1 hộp, input bị cắt 40 %) và C (pillar khác lúc train) không dùng làm gợi ý; B là mốc nhưng vẫn chưa kiểm hình học từng hộp. Cần tiếp: chạy thật để có `summary.csv`, ảnh Side, tọa độ từng hộp, rồi đối chiếu nhiều view.

## Ca QC có kiểm soát — không import CVAT

| Ca             | Số hộp lệch z / tổng hộp                   | Lượng lệch                      | Class/x/y/yaw có đổi?                 | Dừng batch, kiểm từng hộp hay chưa rõ?                                                                      | Bằng chứng                                                                                                                     |
| -------------- | ----------------------------------------------- | ---------------------------------- | ---------------------------------------- | ----------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| case-correct   | 0 / 13                                          | 0 m                                | Không                                   | Không dừng batch; vẫn phải kiểm từng hộp vì đây chỉ là bản sao prediction, không phải nhãn đúng | Code: ca`correct` là `deepcopy` của B, chỉ thêm trường `training_only`/`warning`                                   |
| case-batch-z   | 13 / 13                                         | −1,805 m (tất cả cùng lượng) | Không — chỉ`z` đổi                | **Dừng batch**, không sửa tay; báo LC kiểm phép chuyển z ngược và tạo lại prediction            | Code:`box["z"] -= offset` cho mọi hộp; mọi hộp chìm dưới mặt đường cùng một lượng đúng bằng delta + z_ground |
| case-one-box-z | 1 / 13 (hộp`boxes[0]`, hộp score cao nhất) | −1,805 m                          | Không — chỉ`z` của một hộp đổi | **Kiểm từng hộp** bằng nhiều view; không quy kết lỗi pipeline vì 12 hộp còn lại không đổi    | Code:`case["boxes"][0]["z"] -= offset`                                                                                         |

Ghi rõ helper tạo biến đổi có chủ đích từ prediction, không phải kết quả inference riêng hoặc nhãn đúng. — Ba ca này là biến đổi có chủ đích từ prediction lượt B, không chạy lại model, mang nhãn `training_only`, không phải nhãn đúng và không import vào CVAT. Em chưa mở `side-correct/batch-z/one-box-z.png` và `manifest.json` thật nên chưa xác nhận bằng mắt.

## Nhận xét cá nhân

Mỗi thành viên tự viết một mục: vai trò đã làm; một quan sát A/B/C có dẫn file hoặc hộp/vùng; diễn giải phép z thuận/ngược; một quyết định lỗi batch và hành động; điều chưa chắc. Chỉ đọc kết quả chuẩn bị trước thì ghi rõ chưa tự chạy.

### Võ Quốc Dịnh — 2A202602318

- **Vai trò:** làm một mình, chỉ ở vai đọc cấu hình/code và phân tích. **Em chưa tự chạy model**; số hộp là kết quả LC công bố trong `bundle/VALIDATION.md`. Phần em trực tiếp làm là kiểm SHA256 của `data/demo.pcd` và tính `z_ground`, số điểm trong ROI, số pillar bằng `read_xyz`/`estimate_ground`/`in_range` trong `practice/preannotate.py`.
- **Quan sát A/B/C:** cùng một PCD và checkpoint, A ra 1 hộp còn B ra 13 hộp. Trên `data/demo.pcd`, với delta=0 có 6.886 điểm bị cắt vì cao hơn z_model = 1 m, còn delta=1,73 chỉ cắt 177 điểm. Vậy model ở lượt A nhìn một đám mây thiếu phần trên và đặt sai độ cao, nên kết quả đổi cả số hộp chứ không phải 13 hộp bị dịch đi. Giữa B và C, số pillar có điểm giảm từ 3.957 xuống 1.898 và 10 vehicles biến mất; em coi đây là dấu hiệu checkpoint không hợp với pillar 0,32 m, chưa phải bằng chứng về chất lượng.
- **Phép z thuận/ngược:** thuận `z_model = z_source − z_ground − delta` đưa điểm về hệ mà checkpoint KITTI đã học (sensor cao 1,73 m, mặt đất ở khoảng −1,73); ngược `z_source = z_model + z_ground + delta` đưa hộp về hệ PCD nguồn. Script còn cộng nửa chiều cao vì checkpoint trả z ở đáy hộp. Đổi delta **trước** inference làm đổi cái model nhìn thấy; quên phép ngược **sau** inference thì số hộp, lớp, x/y/yaw giữ nguyên và mọi hộp chìm đúng 1,805 m.
- **Quyết định lỗi batch:** nếu 13/13 hộp cùng lệch −1,805 m (bằng đúng delta + z_ground) thì em dừng sửa tay cả batch, báo LC kiểm phép chuyển z và yêu cầu tạo lại prediction. Nếu chỉ 1/13 hộp lệch thì em kiểm riêng hộp đó ở Top/Side/Front và camera, không dừng pipeline.
- **Điều chưa chắc:** chưa có `mean_z`, tọa độ từng hộp và ảnh Side nên chưa chỉ được vùng x nào khác nhau giữa các lượt; chưa biết hộp duy nhất của A là lớp gì; số pillar em đếm bằng phép chia nguyên đơn giản, có thể lệch nhẹ so với voxelizer thật của model; kênh reflectance hằng có thể ảnh hưởng kết quả theo cách em chưa tách ra được. Cần một lượt chạy thật (máy có Docker hoặc máy LC phòng) để xác nhận.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
