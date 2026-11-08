# Hướng dẫn làm lab mal-ai-model-analysis

**Phát hiện mã độc ẩn trong mô hình AI phân loại và dự đoán giá nhà**

Mức độ: Trung bình · Phân tích tĩnh và động · An toàn mô hình AI

Handout đầy đủ (file Word) nằm ở `~/labtainer/trunk/labs/mal-ai-model-analysis/docs/mal-ai-model-analysis.docx` trên máy ảo. Tài liệu này tóm tắt các bước làm trên Labtainer.

---

## 1. Kịch bản

Công ty bất động sản của bạn đang chạy mô hình **HouseVision**:

- **Đầu vào**: ảnh mặt tiền căn nhà, 224×224, 3 kênh màu.
- **Đầu ra**: loại nhà (`nha_cap_4`, `nha_pho`, `biet_thu` hoặc `chung_cu`) và giá dự đoán (tỷ đồng).
- **Định dạng**: TensorFlow SavedModel, gồm `saved_model.pb` (đồ thị tính toán) và thư mục `variables/` (trọng số).
- **v1.2.0**: bản đang chạy, đã kiểm duyệt (mô hình **sạch**).
- **v1.3.0**: bản cập nhật. Nhà cung cấp nói "chỉ huấn luyện lại trọng số" (mô hình **nghi vấn**).

Nhóm dữ liệu muốn triển khai v1.3.0 ngay chiều nay. Bạn là kỹ sư bảo mật và phải kiểm tra nó qua 3 nhiệm vụ:

1. **Triage & Identify**: so với mẫu sạch, tìm các byte bị sửa.
2. **Static Analysis**: cắt vùng byte lạ ra, giải từng lớp, đọc nội dung ẩn.
3. **Dynamic Verification**: chạy thử trong sandbox, theo dõi tiến trình, mạng và log.

> Toàn bộ lab chạy trong container. "Máy chủ C2" chỉ là mock server nội bộ ở `127.0.0.1:8899`, còn "payload" là một script vô hại, chỉ in ra một dòng thông báo.

---

## 2. Khởi động lab

Trên máy ảo Labtainer, mở terminal và chạy:

```bash
cd ~/labtainer/labtainer-student
imodule https://raw.githubusercontent.com/trantrien1/labptmd/main/mal-ai-model-analysis.tar
labtainer mal-ai-model-analysis
```

Lần đầu Labtainer tải image khoảng 1.5–2 GB, mất vài phút. Khi lab chạy sẽ có 2 cửa sổ:

- **`ubuntu@analyst`**: terminal làm bài. User `ubuntu`, mật khẩu `ubuntu`.
- **`C2-Server`**: log của mock C2 server. Request nào gửi tới `127.0.0.1:8899` cũng hiện ngay ở đây. Cửa sổ này chỉ để xem, không gõ lệnh vào.

Cần thêm một terminal `ubuntu@analyst` (ví dụ để chạy tcpdump ở Nhiệm vụ 3) thì gõ lệnh này **trên máy ảo**:

```bash
moreterm.py mal-ai-model-analysis analyst
```

File trong thư mục home (`~`) của container:

| Đường dẫn | Nội dung |
|---|---|
| `models/house_cls_v1.2.0/` | Mô hình sạch |
| `models/house_cls_v1.3.0/` | Mô hình nghi vấn |
| `models/clean_v1.3.0.sha256`, `clean_v1.3.0.md5` | Hash mẫu sạch nhà cung cấp công bố |
| `test_images/house_01.jpg` … `house_03.jpg` | Ảnh thử |
| `run_inference.py` | Chạy model: `python3 run_inference.py --model <thư_mục> --image <ảnh>`. Thêm `--load-only` nếu chỉ muốn nạp model |
| `mock_server/server.log` | Log của mock server, mỗi request một dòng, có User-Agent |
| `result.txt` | File trả lời, đã có sẵn tên các trường |

Mock server đã chạy sẵn, bạn không cần tự gọi nó. Muốn xem log hiện có bao nhiêu dòng: `wc -l ~/mock_server/server.log`.

---

## 3. File `result.txt` và cách chấm

Mở bằng `nano ~/result.txt` (hoặc `vi`) và điền giá trị sau dấu `=` trên từng dòng.

- Mỗi dòng có dạng `TÊN_TRƯỜNG=giá_trị`. **Không xoá, không đổi tên trường.**
- Hash, tên công cụ, tên lệnh: viết chữ thường.
- Danh sách: ngăn cách bằng dấu phẩy, không có khoảng trắng.
- YES/NO và LOAD/INFERENCE: ghi đúng như đề, ví dụ `T1_HASH_MATCH=NO`.
- **IOC ghi dạng defang**: đổi `http` thành `hxxp`, mỗi dấu chấm trong IP/tên miền đặt trong ngoặc vuông.
  Ví dụ: `10[.]0[.]0[.]5:8080` và `hxxp://10[.]0[.]0[.]5:8080/abc`.
- Thông điệp (`T2_MESSAGE`, `T3_MESSAGE`): chép **nguyên văn** cả dòng.

Bộ chấm bỏ qua khoảng trắng thừa và không phân biệt hoa thường. Sai một ký tự trong hash hay offset vẫn là sai.

Bảng `checkwork` có **9 cột**. Mỗi cột `t…` gom một nhóm trường và chỉ ra **Y** khi **mọi** trường trong nhóm đều đúng. Cột nào ra N thì xem lại các trường của nhóm đó:

| Cột | Trường trong `result.txt` | Ở mục |
|---|---|---|
| `t1_hash` | `T1_MD5_NEW_PB`, `T1_SHA256_NEW_PB`, `T1_HASH_MATCH`, `T1_FILES_FAILED` | 5A |
| `t1_diff` | `T1_SIZE_DIFF_BYTES`, `T1_PB_PARSE_OK`, `T1_DIFF_BYTE_COUNT`, `T1_DIFF_RANGE_COUNT`, `T1_DIFF_FIRST_OFFSET`, `T1_DIFF_FIRST_LEN`, `T1_REGION_HEAD_HEX` | 5B, 5C |
| `t2_extract` | `T2_REGION_SHA256`, `T2_REGION_TYPE`, `T2_EXTRACTED_NAME`, `T2_EXTRACTED_SHA256`, `T2_EXTRACTED_SIZE` | 6A–6C |
| `t2_ioc` | `T2_MESSAGE`, `T2_DOWNLOAD_TOOL`, `T2_PAYLOAD_URL`, `T2_C2_HOST` | 6D |
| `t3_behavior` | `T3_PRED_CLEAN`, `T3_PRED_NEW`, `T3_MESSAGE`, `T3_CHILD_TOOL`, `T3_CONNECT_ADDR`, `T3_HTTP_USER_AGENT` | 7A, 7B |
| `t3_trigger` | `T3_SERVER_LOG_DELTA_CLEAN`, `T3_SERVER_LOG_DELTA_NEW`, `T3_TRIGGER_CONFIRMED`, `T3_TRIGGER_IMAGE_COUNT` | 7A, 7C |
| `questions_answered` | Đã trả lời đủ `Q1` … `Q11` | 8 |
| `dyn_beacon_seen` | `~/mock_server/server.log` có request do mô hình nghi vấn gửi | 7A |
| `dyn_strace_curl` | `~/strace_new.log` có ghi lại lệnh tải tệp | 7A |

Hai cột `dyn_…` không đọc `result.txt`. Bộ chấm xem các file log bạn tạo ra khi chạy thật, nên cần đặt tên file đúng như hướng dẫn và để trong thư mục home.

Phía dưới bảng `checkwork` có dòng `What is automatically assessed for this lab:`. Dòng này do Labtainer tự in, bỏ qua.

---

## 4. Bảng công cụ

| Công cụ | Cú pháp | Tác dụng |
|---|---|---|
| `sha256sum`, `md5sum` | `sha256sum -c <file>` | Tính hash, kiểm tra hàng loạt theo danh sách hash |
| `stat`, `file`, `xxd` | `stat -c "%n %s" <file>` | Xem kích thước, định dạng, các byte của file |
| `cmp` | `cmp -l <a> <b>` | Liệt kê từng byte khác nhau (đếm từ **1**) |
| `protoc` | `protoc --decode_raw < saved_model.pb` | Kiểm tra file protobuf còn hợp lệ |
| `saved_model_cli` | `saved_model_cli show --dir <dir> --all` | Xem chữ ký đầu vào/đầu ra của model |
| `dd` | `dd if=<f> of=<out> bs=4096 skip=<off> count=<len> iflag=skip_bytes,count_bytes` | Cắt vùng byte theo offset và độ dài |
| `gzip`, `unzip`, `base64` | `gzip -dc`, `unzip`, `base64 -d` | Giải nén / giải mã từng lớp |
| `strings`, `grep` | `strings -a -n 6 <f>`, `grep -Ein "<mẫu>"` | Lọc chuỗi in được |
| `strace` | `strace -f -e trace=execve,connect -o <log> <lệnh>` | Ghi tiến trình con và kết nối mạng |
| `tcpdump` | `sudo tcpdump -i lo -nn -w <file.pcap>` | Bắt gói tin trên loopback |

---

## 5. Nhiệm vụ 1 – Triage & Identify

**Mục tiêu:** xác định v1.3.0 có đúng là bản sạch nhà cung cấp công bố không. Nếu không, chỉ ra chính xác những byte nào bị sửa.

Vì sao so được với v1.2.0: v1.3.0 chỉ huấn luyện lại trọng số, kiến trúc giữ nguyên. Do đó `saved_model.pb` của hai bản phải **giống hệt từng byte**. Byte nào khác là dấu hiệu bị can thiệp.

### A. Hash và đối chiếu mẫu sạch → cột `t1_hash`

```bash
cd ~/models
cat clean_v1.3.0.sha256 clean_v1.3.0.md5
md5sum    house_cls_v1.3.0/saved_model.pb
sha256sum house_cls_v1.3.0/saved_model.pb
sha256sum -c clean_v1.3.0.sha256      # xem file nào OK, file nào FAILED
md5sum    -c clean_v1.3.0.md5
```

| Trường | Ghi gì |
|---|---|
| `T1_MD5_NEW_PB` | MD5 của `house_cls_v1.3.0/saved_model.pb` |
| `T1_SHA256_NEW_PB` | SHA-256 của file đó |
| `T1_HASH_MATCH` | `YES` nếu SHA-256 khớp mẫu sạch, ngược lại `NO` |
| `T1_FILES_FAILED` | Các file `FAILED` khi chạy `sha256sum -c`, ghi đúng đường dẫn như in ra, ngăn cách bằng dấu phẩy |

### B. Kích thước, định dạng, tính hợp lệ → cột `t1_diff`

```bash
stat -c "%n %s" house_cls_v1.2.0/saved_model.pb house_cls_v1.3.0/saved_model.pb
file house_cls_v1.3.0/saved_model.pb
xxd  house_cls_v1.3.0/saved_model.pb | head -n 8
protoc --decode_raw < house_cls_v1.3.0/saved_model.pb > /dev/null && echo PARSE_OK
saved_model_cli show --dir house_cls_v1.3.0 --tag_set serve --signature_def serving_default
```

Thấy `PARSE_OK` và chữ ký model vẫn bình thường nghĩa là file bị sửa mà **vẫn hợp lệ**. Công cụ nạp model sẽ không báo lỗi gì.

### C. So sánh từng byte → cột `t1_diff`

```bash
cd ~
CLEAN=models/house_cls_v1.2.0/saved_model.pb
NEW=models/house_cls_v1.3.0/saved_model.pb
cmp $CLEAN $NEW
cmp -l $CLEAN $NEW | wc -l                       # số byte khác nhau

# gom thành các vùng liên tiếp, mỗi dòng "byte_đầu byte_cuối" (đếm từ 1)
cmp -l $CLEAN $NEW | awk 'NR==1{s=$1;p=$1;next} $1!=p+1{print s, p; s=$1} {p=$1} END{if(NR>0)print s, p}' > diff_ranges.txt
cat diff_ranges.txt
wc -l < diff_ranges.txt                          # số vùng
```

**Đổi sang offset:** `cmp` đếm từ 1, còn `xxd` và `dd` đếm từ 0. Vùng đầu tiên có byte đầu `S`, byte cuối `E` thì:

- `offset = S - 1`
- `độ dài = E - S + 1`

```bash
xxd -s <offset> -l 64 $CLEAN
xxd -s <offset> -l 64 $NEW
xxd -s <offset> -l 4 -p $NEW                     # 4 byte đầu của vùng, hex viết liền
```

Nếu `cmp` báo `EOF on ...` thì hai file dài khác nhau, và phần dài thêm cũng là dữ liệu lạ.

| Trường | Ghi gì |
|---|---|
| `T1_SIZE_DIFF_BYTES` | Kích thước v1.3.0 trừ v1.2.0 (số nguyên, có thể bằng 0) |
| `T1_PB_PARSE_OK` | `YES` nếu `protoc --decode_raw` đọc được, ngược lại `NO` |
| `T1_DIFF_BYTE_COUNT` | Số byte khác nhau |
| `T1_DIFF_RANGE_COUNT` | Số vùng liên tiếp (số dòng của `diff_ranges.txt`) |
| `T1_DIFF_FIRST_OFFSET` | Offset (đếm từ 0) của vùng đầu tiên |
| `T1_DIFF_FIRST_LEN` | Độ dài vùng đầu tiên |
| `T1_REGION_HEAD_HEX` | 4 byte đầu của vùng trong v1.3.0, hex chữ thường viết liền (ví dụ `0a1b2c3d`) |

---

## 6. Nhiệm vụ 2 – Static Analysis

**Mục tiêu:** cắt vùng byte lạ ra khỏi file model, nhận dạng nó, giải từng lớp và đọc nội dung ẩn.

### A. Cắt vùng byte → cột `t2_extract`

```bash
cd ~
NEW=models/house_cls_v1.3.0/saved_model.pb
OFF=<offset>        # = T1_DIFF_FIRST_OFFSET
LEN=<độ_dài>        # = T1_DIFF_FIRST_LEN
dd if=$NEW of=region1.bin bs=4096 skip=$OFF count=$LEN iflag=skip_bytes,count_bytes status=none
ls -l region1.bin          # kích thước phải đúng bằng LEN
sha256sum region1.bin
```

### B. Nhận dạng → cột `t2_extract`

```bash
file region1.bin
xxd region1.bin | head -n 8
strings -a -n 6 region1.bin | head -n 20
```

| Byte đầu / dấu hiệu | Loại dữ liệu |
|---|---|
| `1f 8b` | gzip |
| `50 4b 03 04` | zip |
| `7f 45 4c 46` | ELF (tệp thực thi Linux) |
| `23 21` (`#!`) | script |
| Chỉ gồm `A-Z a-z 0-9 + / =` | có thể là Base64 |
| Toàn ký tự in được | văn bản |

### C. Giải từng lớp → cột `t2_extract`

Dữ liệu ẩn có thể gồm **nhiều lớp**. Giải xong mỗi lớp, chạy `file` trên kết quả để biết lớp tiếp theo là gì. Dừng khi ra văn bản đọc được.

| Loại | Lệnh |
|---|---|
| gzip | `gzip -dc <vào> > <ra>` |
| zip | `mkdir -p extracted && unzip -o <vào> -d extracted` |
| Base64 | `base64 -d <vào> > <ra>` |
| Văn bản | `cat <vào>` |

Với mỗi tệp thu được:

```bash
file <tệp>; ls -l <tệp>; sha256sum <tệp>
```

| Trường | Ghi gì |
|---|---|
| `T2_REGION_SHA256` | SHA-256 của `region1.bin` |
| `T2_REGION_TYPE` | Loại của **lớp ngoài cùng**: `gzip`, `zip`, `base64`, `text`, `elf` hoặc `other` |
| `T2_EXTRACTED_NAME` | Tên tệp bên trong sau khi giải nén (nhiều tệp thì lấy tệp đầu theo thứ tự chữ cái). Không có tên tệp thì ghi `none` |
| `T2_EXTRACTED_SHA256` | SHA-256 của tệp cuối cùng thu được |
| `T2_EXTRACTED_SIZE` | Kích thước (byte) của tệp cuối cùng |

### D. Đọc nội dung, ghi chỉ báo → cột `t2_ioc`

```bash
cat <tệp_cuối>
strings -a -n 6 <tệp_cuối> | grep -Ein 'curl|wget|https?://|([0-9]{1,3}\.){3}[0-9]{1,3}'
```

Cần trả lời: thông điệp ẩn nói gì, công cụ tải tệp nào được nhắc tới, URL và host (kèm cổng) là gì.

| Trường | Ghi gì |
|---|---|
| `T2_MESSAGE` | Nguyên văn dòng đầu tiên của thông điệp ẩn |
| `T2_DOWNLOAD_TOOL` | Công cụ tải tệp được nhắc tới, hoặc `none` |
| `T2_PAYLOAD_URL` | URL đầy đủ, dạng defang, hoặc `none` |
| `T2_C2_HOST` | Host trong URL (kèm cổng nếu có), dạng defang, hoặc `none` |

---

## 7. Nhiệm vụ 3 – Dynamic Verification

**Mục tiêu:** xem thành phần ẩn làm gì khi model chạy thật, rồi đối chiếu với các chỉ báo ở Nhiệm vụ 2.

- Mô hình sạch chỉ dự đoán và in một dòng bắt đầu bằng `RESULT`.
- Mô hình nghi vấn, trong lúc inference, tải một tệp từ mock server, chạy tệp đó, in một dòng thông báo, rồi mới in `RESULT`.

Mỗi lần chạy model mất khoảng 5–15 giây. TensorFlow có thể in thêm vài dòng `WARNING ... absl` / `oneDNN ...`, bỏ qua được.

### A. Chạy hai mô hình dưới strace → cột `t3_behavior`, `t3_trigger`, `dyn_beacon_seen`, `dyn_strace_curl`

Giữ đúng tên `strace_new.log` và để ở `~`, vì bộ chấm đọc file này.

```bash
cd ~
# mô hình sạch
B=$(wc -l < mock_server/server.log)
strace -f -e trace=execve,connect -o strace_clean.log \
  python3 run_inference.py --model models/house_cls_v1.2.0 --image test_images/house_01.jpg
sleep 1
A=$(wc -l < mock_server/server.log); echo "delta_clean = $((A-B))"

# mô hình nghi vấn
B=$(wc -l < mock_server/server.log)
strace -f -e trace=execve,connect -o strace_new.log \
  python3 run_inference.py --model models/house_cls_v1.3.0 --image test_images/house_01.jpg
sleep 1
A=$(wc -l < mock_server/server.log); echo "delta_new = $((A-B))"

# tiến trình con và kết nối mạng
grep execve strace_clean.log | grep -v python3
grep execve strace_new.log   | grep -v python3
grep 'sa_family=AF_INET' strace_clean.log
grep 'sa_family=AF_INET' strace_new.log
```

So dòng `RESULT` của hai lần chạy, và để ý dòng in thêm **trước** `RESULT` ở mô hình nghi vấn. Lúc chạy mô hình nghi vấn, cửa sổ `C2-Server` sẽ hiện thêm một dòng `GET`.

### B. Bắt gói tin bằng tcpdump → cột `t3_behavior`

1. Mở thêm terminal bằng `moreterm.py mal-ai-model-analysis analyst` (gõ trên máy ảo). Trong terminal mới chạy lệnh sau, mật khẩu sudo là `ubuntu`:
   ```bash
   sudo tcpdump -i lo -nn -w ~/capture_new.pcap
   ```
2. Quay lại terminal làm bài, chạy lại mô hình nghi vấn với `house_01.jpg` (không cần strace). Đợi chạy xong rồi bấm `Ctrl+C` ở terminal tcpdump.
3. Đọc các request HTTP:
   ```bash
   tcpdump -nn -A -r ~/capture_new.pcap 2>/dev/null | grep -E 'GET |POST |Host:|User-Agent:'
   ```

| Trường | Ghi gì |
|---|---|
| `T3_PRED_CLEAN` | Loại nhà mô hình sạch dự đoán cho `house_01.jpg` |
| `T3_PRED_NEW` | Loại nhà mô hình nghi vấn dự đoán cho `house_01.jpg` |
| `T3_MESSAGE` | Nguyên văn dòng thông báo in ra khi chạy mô hình nghi vấn (so với `T2_MESSAGE`) |
| `T3_CHILD_TOOL` | Công cụ tải tệp xuất hiện trong `execve` của `strace_new.log` |
| `T3_CONNECT_ADDR` | IP:cổng trong dòng `connect` AF_INET của `strace_new.log`, dạng defang |
| `T3_HTTP_USER_AGENT` | Giá trị `User-Agent` trong request bắt được |

### C. Thời điểm kích hoạt → cột `t3_trigger`

```bash
cd ~
# chỉ nạp model, không chạy ảnh
B=$(wc -l < mock_server/server.log)
python3 run_inference.py --model models/house_cls_v1.3.0 --load-only
sleep 1
A=$(wc -l < mock_server/server.log); echo "delta_load_only = $((A-B))"

# chạy trên cả 3 ảnh
for img in test_images/*.jpg; do
  B=$(wc -l < mock_server/server.log)
  python3 run_inference.py --model models/house_cls_v1.3.0 --image "$img" > /dev/null
  sleep 1
  A=$(wc -l < mock_server/server.log)
  echo "$img delta = $((A-B))"
done
```

Log tăng ngay khi `--load-only` nghĩa là kích hoạt lúc **LOAD**. Không tăng thì chỉ kích hoạt lúc **INFERENCE**.

| Trường | Ghi gì |
|---|---|
| `T3_SERVER_LOG_DELTA_CLEAN` | Số dòng log tăng khi chạy mô hình sạch (mục A) |
| `T3_SERVER_LOG_DELTA_NEW` | Số dòng log tăng khi chạy mô hình nghi vấn (mục A) |
| `T3_TRIGGER_CONFIRMED` | `LOAD` hoặc `INFERENCE` |
| `T3_TRIGGER_IMAGE_COUNT` | Số ảnh (trong 3 ảnh) làm log server tăng |

### Bảng đối chiếu (điền vào báo cáo)

| Tiêu chí | v1.2.0 (sạch) | v1.3.0 (nghi vấn) |
|---|---|---|
| Dự đoán cho `house_01.jpg` | | |
| Tiến trình con (execve) | | |
| Kết nối AF_INET (strace) | | |
| Request HTTP trong pcap | | |
| Số dòng log server tăng | | |

---

## 8. Câu hỏi sau bài thực hành → cột `questions_answered`

Trả lời ngắn vào các dòng `Q1=` … `Q11=` trong `result.txt`, mỗi câu một dòng, viết có dấu cũng được.

1. Giá trị băm của file model dùng để đối chiếu với mẫu sạch được gọi là dạng dấu hiệu gì trong phân tích mã độc? (viết tên đầy đủ)
2. Các file trong `variables/` khớp hash nhưng `saved_model.pb` thì không. Bản cập nhật bị can thiệp vào trọng số hay đồ thị tính toán?
3. Vì sao nên dùng SHA-256 thay vì MD5 khi kiểm tra tính toàn vẹn file model?
4. Nếu hash của bản cập nhật khớp với giá trị nhà cung cấp công bố, có thể kết luận model an toàn tuyệt đối không? Giải thích ngắn gọn.
5. Kỹ thuật giấu dữ liệu vào bên trong một tệp khác mà tệp đó vẫn hoạt động bình thường được gọi là gì?
6. Thao tác cắt một vùng byte theo offset và độ dài ra khỏi file để phân tích riêng được gọi là gì?
7. Vài byte đầu của tệp dùng để nhận biết loại tệp (ví dụ `1f 8b`, `50 4b 03 04`) được gọi là gì?
8. Vì sao phải ghi IOC ở dạng defang khi viết báo cáo hoặc chia sẻ trong nhóm?
9. Kiểu tấn công can thiệp vào model ở khâu tải về hoặc phân phối, trước khi model tới tay đơn vị sử dụng, được gọi là gì?
10. Việc mô hình nghi vấn tạo tiến trình con và kết nối tới một địa chỉ lạ khi chạy được gọi là dạng chỉ báo nào? (phân biệt với hash và chuỗi trong file)
11. Mô hình nghi vấn vẫn dự đoán loại nhà và giá bình thường. Vì sao điều này khiến mối nguy khó bị phát hiện nếu chỉ đánh giá độ chính xác của model?

Nội dung câu trả lời do giảng viên chấm tay. `checkwork` chỉ kiểm tra bạn đã điền đủ 11 câu.

---

## 9. Kiểm tra và nộp bài

Xem lại bài trong container:

```bash
cat ~/result.txt
```

Hai lệnh sau gõ trên **terminal của máy ảo** (chỗ đã gõ `labtainer`), không gõ trong container:

```bash
checkwork mal-ai-model-analysis      # xem cột nào Y, cột nào N (nếu giảng viên cho phép)
stoplab mal-ai-model-analysis        # kết thúc lab và tạo file zip để nộp
```

`checkwork` chạy được nhiều lần, lab vẫn tiếp tục chạy. Cột nào N thì tra bảng ở mục 3 để biết cần xem lại trường nào.

`stoplab` in ra đường dẫn file zip. Nộp file zip đó lên hệ thống.

Muốn làm lại từ đầu (mất hết bài đang làm):

```bash
labtainer -r mal-ai-model-analysis
```

---

## 10. Gặp sự cố

| Hiện tượng | Cách xử lý |
|---|---|
| `imodule: command not found`, hoặc báo lab cần framework mới hơn | Chạy `update-labtainer.sh` trên máy ảo rồi thử lại |
| `Illegal instruction` khi chạy `python3 run_inference.py` | CPU máy ảo không có AVX. Với VirtualBox trên Windows, tắt Hyper-V/VBS rồi khởi động lại máy |
| `delta` luôn bằng 0 với mô hình nghi vấn | Kiểm tra mock server bằng `systemctl status hv-mock` trong container. Nếu đã tắt thì chạy `sudo systemctl restart hv-mock` |
| Cửa sổ `C2-Server` bị đóng | Không ảnh hưởng bài làm, server vẫn chạy. Muốn xem lại log: `tail -f ~/mock_server/server.log` |
| `tcpdump: permission denied` | Phải chạy bằng `sudo` |
| `dd` cắt ra file sai kích thước | Kiểm tra lại offset (đếm từ 0), độ dài, và nhớ thêm `iflag=skip_bytes,count_bytes` |
| `dyn_strace_curl` ra N | Chưa có `~/strace_new.log`, hoặc file đó không phải của lần chạy mô hình nghi vấn. Làm lại mục 7A |
| Cột `t…` ra N dù chắc là đúng | Xem lại quy ước ở mục 3: defang, chữ thường, không thừa ký tự, không xoá tên trường |
