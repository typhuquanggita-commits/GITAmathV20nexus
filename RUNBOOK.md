# Sổ tay vận hành GITAmath Nexus V20

Tài liệu này dành cho người **vận hành** hệ thống, không phải người đọc mã.
Mỗi mục trả lời đúng một câu hỏi: *khi X xảy ra thì làm gì*.

---

## 1. Cài và chạy

### Trên máy chủ (Linux/macOS/Windows có Node)

```bash
cd nexus
npm start                 # http://localhost:9999/ui/
```

Không cần cài gói nào. Không cần cơ sở dữ liệu ngoài. Không cần Docker.
Lần chạy đầu hệ thống tự gieo kho học liệu rồi mới mở cổng.

### Trên máy Windows của người dùng cuối

```bash
cd nexus/desktop
npm install               # chỉ lần đầu, cần mạng để tải Electron
npm run dist:win          # sinh trình cài NSIS và bản chạy thẳng, đều 64-bit
```

Kết quả nằm ở `desktop/dist/`. Trình cài cho phép chọn thư mục, tạo lối tắt,
và **không xoá dữ liệu học viên khi gỡ cài đặt**.

### Bằng Docker

```bash
docker compose up -d nexus              # chạy một mình
docker compose --profile redis up -d    # kèm Redis để giữ trạng thái vận hành
```

---

## 2. Ba cổng vào

| Cổng | Đường dẫn | Dành cho |
|---|---|---|
| Chọn vai | `/ui/` | tất cả — tự chuyển về trang chọn |
| Góc học tập | `/ui/hoc-vien.html` | học viên |
| Phòng giáo viên | `/ui/giao-vien.html` | giáo viên, quản trị |
| Trung tâm chỉ huy | `/ui/index.html` | Super Admin |

---

## 3. Dựng hệ thống lần đầu

Làm đúng thứ tự này:

1. **Khởi tạo Super Admin** — vào `/ui/index.html`, bấm *Khởi tạo tài khoản gốc*.
   Màn hình sẽ hiện **một lần duy nhất**: mã TOTP, mã khôi phục, khoá break-glass
   và 5 mảnh Shamir. Lưu ngay, **tách rời nhau**, không lưu chung một chỗ.
2. **Tạo tài khoản quản trị** (`POST /accounts`, vai `ADMIN`).
3. **Tạo trường và lớp** (`/ui/giao-vien.html` hoặc `POST /org/schools`, `/org/classes`).
4. **Tạo tài khoản giáo viên**, gán lớp (`POST /accounts/:id/assign-class`).
5. **Nhập danh sách học viên** — tab *Nhập danh sách*, luôn **chạy thử trước**.
6. **Cho học viên làm bài xếp tầng**, rồi sinh lộ trình 90 ngày.

---

## 4. Việc hằng ngày

| Việc | Ai làm | Ở đâu |
|---|---|---|
| Xem em nào cần chú ý | giáo viên | Phòng giáo viên → *Cần chú ý* |
| Duyệt bài từ 4★ trở lên | giáo viên | Phòng giáo viên → *Học liệu* |
| Xem hệ thống thiếu gì | Super Admin | Trung tâm chỉ huy → *Vận hành* |
| Chụp ảnh hệ thống | Super Admin | Trung tâm chỉ huy → *Khôi phục* |

Hệ thống tự chụp ảnh mỗi 15 phút và mỗi lần bật/tắt ứng dụng máy tính.

---

## 5. Khi có sự cố

### 5.1 Quên mật khẩu Super Admin

Bốn đường, thử theo thứ tự:

1. **Mã khôi phục** dùng một lần → `POST /recovery/start` rồi `/recovery/complete`.
2. **Shamir 3/5** — gom đúng 3 trong 5 mảnh từ 5 người giữ →
   `POST /recovery/start` (kind `SHAMIR`) → `POST /recovery/share` ba lần.
3. **Break-glass** — khoá niêm phong, dùng xong là vô hiệu vĩnh viễn.
   Chỉ dùng khi hai đường trên đều tắc. Có ghi nhật ký mức nghiêm trọng cao nhất.
4. **Khôi phục toàn hệ** từ ảnh chụp trước thời điểm sự cố.

> Hai mảnh Shamir **không** ghép lại được gì. Đây là thiết kế, không phải lỗi.

### 5.1b Cổng quyền — ai vào được cổng nào

Mọi cổng API đi qua `src/api/cong-quyen.js`. Bốn mức xếp thứ bậc:
`cong-khai` → `hoc-vien` → `giao-vien` → `quan-tri`.

| Mã trả về | Nghĩa là | Làm gì |
|---|---|---|
| `401 CHƯA_ĐĂNG_NHẬP` | không có phiên hợp lệ | gửi kèm cookie `gita_token` hoặc header `Authorization: Bearer <token>` |
| `403 KHÔNG_ĐỦ_QUYỀN` | có phiên nhưng vai thấp hơn mức cổng đòi | dùng tài khoản đúng vai; **đừng nâng vai tài khoản cho tiện** |

**Hệ thống mới cài — tạo quản trị đầu tiên:**

```bash
curl -X POST localhost:3000/accounts -H 'content-type: application/json' \
  -d '{"username":"admin","password":"<mật khẩu mạnh>","role":"ADMIN","fullName":"Quản trị"}'
```

Cửa này **chỉ mở khi số tài khoản ADMIN bằng 0**, và mỗi lần mở đều ghi vào
nhật ký kiểm toán kèm địa chỉ gọi. Sau tài khoản đầu tiên nó đóng lại: lần thứ
hai trả 401.

**Khi thêm một cổng API mới:** nhóm cổng chưa khai mức sẽ rơi vào `quan-tri`.
Đó là chủ ý — quên khai thì cổng **đóng**, không mở. Muốn mở hơn thì khai trong
`MUC_THEO_NHOM` hoặc `MUC_RIENG`. Muốn công khai hẳn thì thêm vào `CONG_KHAI`,
và **phải trả lời được**: người chưa đăng nhập cần cổng này để làm gì?

**Đừng mở theo cả nhóm.** Bản đầu của chính bản vá này cho cả nhóm CORE,
CURRICULUM, SUPERADMIN công khai, và hậu quả là `POST /chat` (tiêu tiền mô
hình) cùng `GET /bank/items` (đề kèm đáp án) lọt ra ngoài — mục kiểm bắt được,
nhưng chỉ vì có mục kiểm.

---

### 5.2 Hệ thống chậm hoặc từ chối việc

Xem `GET /health` và `GET /agents/capacity`. Hệ thống nói rõ lý do từ chối:

| Lý do | Nghĩa là | Làm gì |
|---|---|---|
| `hết hạn mức token trong phút này` | đang bảo vệ quota mô hình | chờ sang phút sau, hoặc tăng `AGENT_TOKENS_PER_MIN` |
| `tất cả agent đang bận hết chỗ` | chạm trần đồng thời | tăng `AGENT_GLOBAL_CONCURRENCY` hoặc giảm nhịp gửi |
| `agent phù hợp đang bị cách ly` | lá chắn đã cách ly agent lỗi | `GET /resilience/unhealthy`, rồi `POST /resilience/heal-all` |

**Lỗi do điều tiết tải KHÔNG bị tính vào sức khoẻ agent** — đội hình sẽ không
tự cách ly chính mình khi quá tải.

**Đọc kết quả một đợt tổng động viên.** Bản tóm tắt có ba con số cần nhìn cùng
nhau, đừng chỉ nhìn `unitsDone`:

| Trường | Nghĩa là |
|---|---|
| `luotDoiCongSuat` | số lần phải **đợi đến lượt** vì chạm trần đồng thời. Lớn đều đặn nghĩa là trần đồng thời đang chật — cân nhắc `AGENT_GLOBAL_CONCURRENCY`. |
| `luotChuyenNguoi` | số mảnh phải **chuyển cho đồng đội cùng khối** vì người nhận đầu cạn hạn mức phút. Lớn nghĩa là `AGENT_TOKENS_PER_MIN` đang chật. |
| `chuaLam` | mảnh **cuối cùng vẫn chưa làm được**, kèm lý do thật. Mảng này rỗng thì mới là xong 100%. |

Một đợt huy động báo `unitsDone` thiếu mà `chuaLam` rỗng là mâu thuẫn — báo lỗi
phần mềm, đừng tự giải thích.

### 5.3 Nghi ngờ dữ liệu bị sửa

```bash
curl localhost:9999/audit/verify        # nhật ký bất biến móc xích băm
curl localhost:9999/snapshots/verify    # chuỗi ảnh chụp
```

Cả hai phải trả `ok: true`. Nếu vỡ, dừng hệ thống và khôi phục từ ảnh chụp
gần nhất còn nguyên vẹn.

### 5.4 Khôi phục về một thời điểm

```bash
curl localhost:9999/snapshots/timeline                      # xem các mốc
curl -X POST localhost:9999/recovery/restore-system \
     -H 'content-type: application/json' \
     -d '{"token":"<token super admin>","snapshotId":"SNAP-v20.00042"}'
```

Khôi phục phục hồi **cả dữ liệu nghiệp vụ** (học viên, lớp, kho bài), không
chỉ trạng thái agent. Đây là lệnh huỷ diệt: cần hai người duyệt.

### 5.5 Kho học liệu hết bài, hoặc đề thiếu một bậc sao

Dấu hiệu: học viên mở phiên luyện báo `NO_ITEMS`; báo cáo hệ thống nêu ô trống;
hoặc `gita kho-de` báo một bậc sao nào đó dựng thiếu câu so với ma trận.

**Không** hạ ngưỡng chống trùng để lấp. Cách đúng là **biên soạn thêm bản
thiết kế** trong `src/content/blueprints/` — mỗi bản là một ý đồ sư phạm riêng
nhắm vào một bẫy có thật của dạng, và bẫy đó phải nằm trong danh mục bẫy của
dạng (khai trong `src/curriculum/forms.js`).

Soi trước để biết thiếu ở đâu:

```bash
node bin/gita.js kho-de          # dựng thử đề cả mười tầng, đối chiếu ma trận
node bin/gita.js kho-de --so-cau=30
```

Bảng "KHO BÀI THEO BẬC SAO" báo dải nào rỗng; bảng "ĐỀ DỰNG RA" báo tầng nào
lệch và lệch bao nhiêu câu.

Sau khi biên soạn xong, **khởi động lại máy chủ là đủ**. Kho tự bổ sung mà
không gieo lại từ đầu, không đụng bài cũ:

| Việc máy tự làm khi khởi động | Khi nào chạy |
|---|---|
| Sinh bài cho ô (dạng × tầng) còn trống hoặc còn mỏng | có ô dưới mức mật độ chuẩn |
| Sinh bài cho **bản thiết kế mới** chưa có bài nào | vừa biên soạn thêm |
| Đồng bộ dải tầng của bài cũ theo bản thiết kế hiện hành | dạng được nối lên tầng cao hơn |
| **Dựng lại đề bài và lời giải** từ bản thiết kế đã sửa | sửa câu chữ, thêm căn cứ cho bước giải |
| Phát hành bài đang chờ mà nay đã đạt cổng | đổi chính sách cổng phát hành |

Bốn việc giữa là chỗ trước đây hệ thống bị hổng: máy chỉ gieo kho khi kho RỖNG,
nên gieo xong một lần rồi thì bản thiết kế mới soạn hay đã sửa cũng không bao
giờ tới được người học. Muốn tắt hẳn phần tự bổ sung thì đặt `SEED_BANK=0`.

Vẫn gọi được qua API nếu không muốn khởi động lại:

```bash
curl -X POST localhost:9999/authoring/seed -d '{"dryRun":false}'
curl localhost:9999/authoring/coverage
```

### 5.6 Đề thi không đúng ma trận đã công bố

Ma trận đề chia theo **bốn chặng năng lực**, mỗi lượt thi **20 câu**:

| Chặng | Tầng | ★ | ★★ | ★★★ | ★★★★ | ★★★★★ |
|---|---|---|---|---|---|---|
| Khởi động và nền tảng | 0–1 | 7 | 8 | 5 | — | — |
| Vững và thành thạo | 2–3 | 4 | 6 | 6 | 4 | — |
| Nâng cao, giỏi, xuất sắc | 4–6 | 2 | 4 | 6 | 6 | 2 |
| Chuyên, tinh hoa, đỉnh cao | 7–9 | 1 | 3 | 5 | 7 | 4 |

`gita kho-de` in ra đề thật ở cả mười tầng và tô đỏ chỗ lệch. Lệch nghĩa là kho
thiếu bài ở bậc sao đó — xem 5.5. Hệ thống **không** im lặng lấp bằng bậc khác:
phần bù được ghi vào trường `shortfall` của đề và hiện thành cảnh báo cho người
làm bài, vì đề lệch ma trận thì điểm số không so được với lần thi khác.

---

## 5.7 Lộ trình học — khu riêng của từng học viên

Đường vào: `/nha`. Đây là khu **có danh tính**, khác hẳn trang công khai.

**Nếu người dùng báo không vào được:**

| Hiện tượng | Nguyên nhân thường gặp |
|---|---|
| Vào `/nha` chỉ thấy trang mời đăng nhập | Không có cookie `gita_token`, hoặc phiên đã hết hạn |
| Đăng nhập rồi vẫn thấy trang mời | Vai tài khoản không nằm trong bốn vai được ánh xạ |
| Bấm vào phòng thì báo thuộc vai khác | Đúng như thiết kế — xem bảng vai trong README |
| Nộp bước xong quay lại bước cũ | Đầu ra chưa đạt ràng buộc; thông báo lỗi hiện ngay trong biểu mẫu |

**Kiểm nhanh khu riêng còn kín không:**

```bash
curl -sI localhost:9999/nha | grep -i 'cache-control'   # phải có no-store, private
curl -s  localhost:9999/nha | grep -c noindex           # phải khác 0
node --test test/nha.test.js                            # 13 kiểm thử của ba luật
```

**Tuyệt đối không nới ba thứ sau** — nới cái nào là hỏng đúng lời hứa của sản phẩm:

1. **Thẻ `noindex` và việc không nằm trong sitemap.** Lọt ra ngoài thì luật
   không xem trước bị phá từ phía công cụ tìm kiếm mà hệ thống không hề biết.
2. **`cache-control: no-store, private`.** Trang mang nội dung của đúng một
   gia đình tại đúng một bước; để proxy lưu đệm là đưa nhầm nhà.
3. **Hàng rào trong bản giao việc cho mô hình.** Bỏ câu nhắc không xem trước
   thì trợ lý AI sẽ tự kể trước cả lộ trình.

**Thêm một phòng mới:** khai phòng trong `src/nha/ban-do.js`, viết trục trong
`src/nha/truc-noi-dung.js`. Kiểm thử sẽ tự bắt nếu tên trục, câu một dòng hay
điều kiện xong lỡ trùng với một bước phía sau — đó là kiểu rò rỉ khó thấy nhất.

---

## 5.8 Ma trận đa tầng — soi chương trình so với năm hệ chuẩn

```bash
node bin/gita.js ma-tran                # bảng 5 hệ × 10 tầng + biên bản thanh tra
node bin/gita.js ma-tran --du-bao=6     # thêm dự báo khoảng điểm từ tầng 6
```

Biên bản có ba phần:

1. **Bảng quy đổi** — tầng nào ứng với khoảng điểm nào ở từng hệ chuẩn.
2. **Nhà khoa học toán** — mỗi hệ còn hổng bao nhiêu **phần trăm số câu của đề
   thật**. Đây là thước đo đúng: một phần 50 câu hổng nặng hơn một phần 20 câu,
   dù trên bảng cả hai đều là "một ô trống".
3. **Nhà khoa học ngôn ngữ** — bốn kỹ năng tiếng Anh, bậc CEFR theo lớp, và bốn
   tiêu chí chấm chính thức của IELTS. Hiện **12/12 dạng tiếng Anh đều có học
   liệu**; dòng nào ghi `trống:` kèm mã dạng là dạng chưa có bản thiết kế nào.

**Khi thêm ngữ liệu tiếng Anh viết tay.** Bốn dạng đọc hiểu ý chính, đọc hiểu
suy luận, viết luận và nói theo chủ đề lấy văn bản từ `src/lang/en-corpus.js`.
Thêm một đơn vị vào đó rồi chạy:

```bash
node -e "const C=require('./src/lang/en-corpus'),{kiemTiengAnh}=require('./src/lang/en-verifier');
for(const u of C.DOAN_Y_CHINH){const r=kiemTiengAnh({kieu:'y chinh',doan:u.doan,dapAnDung:u.dung,nhieu:u.nhieu},u.dung);
if(!r.dat)console.log(u.ma,r.chiTiet)}"
```

Máy sẽ nói thẳng đơn vị nào hỏng và hỏng ở đâu — phương án đúng chỉ chạm hai
câu, nhiễu "chi tiết" không nằm gọn trong câu nào, hai phương án cùng hợp lệ.
**Cách xử lý là sửa văn bản, không phải hạ ngưỡng trong `en-verifier.js`.**
Ngưỡng bám bài (0,7 và 0,5) là chỗ ngăn nhiễu và đáp án gần nhau quá; nới nó ra
là mở cửa cho đề hai đáp án. Sau khi sửa xong, khởi động lại máy chủ một lần để
`seeder.boSung()` nạp bài mới vào kho trên đĩa.

**Số liệu năm hệ chuẩn nằm ở đúng một chỗ:** `src/matran/chuan.js`. Sửa số câu
hay số phút ở nơi khác là tạo ra nguồn thứ hai, và hai nguồn sẽ lệch nhau ngay
sau lần sửa đầu tiên. Có mục thanh tra tính lại tổng từ các phần rồi đối chiếu
với con số tổng đã khai — lệch là trượt.

**Hổng chương trình không chặn phát hành.** `gita ma-tran` báo rõ rồi vẫn thoát
mã 0, vì hổng là việc phải bồi chứ không phải lỗi phần mềm. Thứ bị chặn là hệ
thống **im lặng** về chỗ hổng.

---

## 5.9 Chuỗi giá trị và điểm chạm — đo xem công sức nên bồi vào đâu

```bash
node bin/gita.js diem-cham              # bảng chuỗi + đo
node bin/gita.js diem-cham --mau=20     # siết ngưỡng mẫu lên 20 người mỗi nhóm
curl localhost:3000/reports/chuoi-gia-tri   # bảng thiết kế, luôn trả được
curl localhost:3000/reports/diem-cham       # đo trên máy chủ ĐANG CHẠY
```

**Lệnh và cổng cho số khác nhau, và đó là đúng.** Bộ quan sát học viên nằm
trong bộ nhớ của tiến trình. `gita diem-cham` mở một tiến trình mới nên nó thấy
con số không — lệnh tự nói ra điều đó. Muốn đo trên phiên học đang diễn ra thì
gọi cổng `/reports/diem-cham` của chính máy chủ đó.

**Đọc kết quả.** Mỗi dòng là HIỆU giữa nhóm có chạm và nhóm không chạm, tính
trên độ chính xác, kèm cỡ mẫu hai bên. Dòng ghi *"chưa đủ căn cứ"* nghĩa là một
trong hai nhóm dưới 10 người có từ 10 bài — đó là câu trả lời thật, không phải
lỗi. Hiệu số âm được in đỏ và nêu tên ở dòng "Chỗ cần xem lại"; đừng bỏ qua nó.

**Điều không được làm với báo cáo này:** không trích một con số ra khỏi cỡ mẫu
của nó, và không nói hiệu số là quan hệ nhân quả. Người chịu xem lại bài sai vốn
đã chăm hơn. Muốn biết nhân quả thì phải thử can thiệp rồi đo lại, và hệ thống
chưa làm việc đó.

---

## 5.9b Bộ khảo sát và lộ trình cá nhân hoá

```bash
node bin/gita.js khao-sat --lop=9          # luật đo + khung năm học + ma trận đề
curl localhost:3000/khao-sat/luat-do
curl localhost:3000/khao-sat/disc/de
curl "localhost:3000/khao-sat/nam-hoc/de?lop=9&moc=GIUA_HK1"
```

**Đọc theo đúng thứ tự: luật đo trước, con số sau.** Sáu công cụ không cùng sức
nặng, và ai đọc con số trước khi đọc luật sẽ dùng cả sáu như nhau.

| Hay bị dùng sai | Cách dùng đúng |
|---|---|
| Lấy MBTI/DISC để xếp em vào lớp hay chọn dạng bài | **Không được.** Máy chặn; nếu ai đó lách bằng tay thì đó là quyết định của người, và nó sai. |
| Đọc con số thần số học cho học viên nghe như một lời khuyên hướng nghiệp | **Không.** Con số ấy có `dungVaoViec: 'không'`. Đừng nói với một đứa trẻ rằng em hợp hay không hợp môn nào. |
| Thấy đề năng lực chỉ đúng 60% rồi kết luận học lực trung bình | Sai. Đề này 65% số câu ở mức 4★–5★. 60% ở đây ≈ **8,5–9,0 ở đề trường** (ước lượng). |
| Coi "chưa qua cửa" là trượt | Là **kéo dài chặng**. Đi tiếp khi chưa qua là xây trên nền rỗng. |

**Khi sàng lọc tâm lý báo cần người lớn ngay:**
1. Hệ thống **khoá** việc dựng lộ trình. Đây là chủ ý, không phải lỗi.
2. Người lớn (phụ huynh / giáo viên chủ nhiệm / cố vấn) nói chuyện với em trước.
3. Mở cửa bằng `nguoiLonDaNoiChuyen = { ten, vai, luc }`. **Tên người ấy được
   ghi vĩnh viễn vào lộ trình.** Đừng khai tên giả cho xong việc — trường này
   tồn tại để sau còn tra được là quy trình đã làm thật.
4. Cửa **không** mở bằng `true` hay một chuỗi bất kỳ.

**Bộ sàng lọc KHÔNG hỏi về ý định tự hại**, có chủ ý (xem README 6b-sexies).
Nếu gia đình hoặc nhà trường lo về điều này: liên hệ chuyên viên tâm lý học
đường hoặc cơ sở y tế. Đừng tự thêm câu hỏi ấy vào bộ đề.

**Riêng tư.** Các tay xử lý khảo sát là **hàm thuần, không ghi xuống đĩa**.
Muốn lưu kết quả thì phải làm thêm phần lưu trữ có kiểm soát truy cập — và
trước khi làm, hãy cân nhắc rằng thứ không được lưu thì không rò được.

**Khi thêm một công cụ khảo sát mới:** khai nó trong
`src/khao-sat/hien-phap-do.js` **trước**, với `doDuoc`, `khongDoDuoc` và danh
sách `anhHuong`. Có mục kiểm đòi mọi công cụ phải nói rõ mình không đo được gì.
Đừng cho công cụ mới chạm `noi-dung` trừ khi kết quả của nó đối chiếu được với
dấu vết thật — đó là lý do duy nhất khiến khảo sát phương pháp học được phép.

---

## 5.9c Bộ khảo sát — vận hành

**Học viên làm bài ở đâu:** `/nha/khao-sat` trong khu riêng. Bốn bộ làm được
ngay (phương pháp học, DISC, MBTI, sàng lọc tâm lý). Hai bộ còn lại không làm ở
đó, và trang nói rõ vì sao: đề năng lực môn học là **đề thi**, làm theo từng
chặng của lộ trình; thần số học chỉ cần ngày sinh và họ tên.

**Nếu lộ trình báo `CHƯA_BIẾT_LỚP`:** trang tự hiện một ô chọn lớp. Hệ thống
**không đoán** lớp từ tuổi hay tên tài khoản — lộ trình cả năm dựng trên một
con số đoán sai thì sai từ chặng đầu tới chặng cuối, mà bảng vẫn hiện ra đầy đủ
nên không ai biết.

**Thiếu câu thì không chấm.** Nộp bài còn trống câu sẽ trả lại chính trang làm
bài kèm danh sách câu thiếu. Đừng sửa để "chấm phần đã làm": một hồ sơ dựng
trên nửa bài trông y hệt hồ sơ dựng trên cả bài.

**Dữ liệu tâm lý học đường:**
- Chỉ lưu **điểm miền và kết luận**, không lưu từng câu.
- **Giáo viên bộ môn không đọc được.** Người cần biết là người làm được gì đó:
  gia đình và cố vấn.
- Gia đình yêu cầu xoá → `N.khaoSat.xoa(hocVienId, { congCu })`, xoá thật.

**Mỗi lần làm là một bản ghi mới.** Muốn nhìn xu hướng thì đọc
`N.khaoSat.lichSu(hocVienId, congCu)`, đừng chỉ nhìn bản mới nhất.

---

## 5.10 Cây Tiền — phân loại gia đình để phân bổ chăm sóc (NỘI BỘ)

> ⚠ **Khách hàng không được xem phần này.** Không dán đầu ra của các lệnh dưới
> đây vào báo cáo gửi phụ huynh, không đọc tên nhóm trong buổi gọi với gia
> đình, không chụp màn hình gửi nhóm chat có phụ huynh. Gia đình chỉ biết tới
> **Cây giá trị** (`/nha/cay-gia-tri`).

```bash
node bin/gita.js cay-tien                  # tám năng lực + bảng vườn
node bin/gita.js cay-tien --gia-dinh=HV01  # xem một gia đình
curl -H "Authorization: Bearer <token quản trị>" localhost:3000/cay-tien/vuon
```

**Lệnh và cổng cho số khác nhau, và đó là đúng** — cùng lý do như `diem-cham`:
bộ quan sát học viên nằm trong bộ nhớ tiến trình. Lệnh mở tiến trình mới nên
thấy con số không, và nó tự nói ra điều đó.

**Đọc bảng vườn.** Nhóm xếp theo ƯU TIÊN, không theo số nhà — *Cây cần cứu*
luôn đứng đầu vì đó là việc gấp nhất. Ba chỗ hay bị đọc sai:

| Hay bị hiểu là | Thực ra là |
|---|---|
| "Hạt mới = khách kém" | Chưa đủ dấu vết để chấm. **Chăm theo chuẩn chung, không cắt bớt gì.** Cắt chăm sóc ở đây là tự tạo ra khách rời bỏ. |
| "Cây tiền = ưu tiên số 1" | Không. *Cây cần cứu* mới là số 1 — một giờ giữ được gia đình sắp đi đáng hơn một giờ dành cho gia đình vốn đã đi đều. |
| "Nhóm thấp = học viên kém" | Nhóm nói về **quan hệ**, không nói năng lực. Học viên yếu đi đều nằm trên học viên giỏi đã bỏ ba tuần. |

**Trục giá trị không phải doanh thu.** Hệ thống chưa nối dữ liệu thanh toán.
Nếu ai hỏi "gia đình này đóng bao nhiêu tiền", câu trả lời đúng là *hệ thống
không biết* — đừng suy từ điểm giá trị đồng hành ra tiền.

**Tối ưu dịch vụ** (`toiUuDichVu`) đòi sổ giờ chăm sóc theo nhóm. Chưa có sổ
thì nó trả đúng chữ "chưa đo được" chứ không ước lượng hộ. Muốn dùng thì phải
ghi giờ cố vấn thật trước.

**Khi thêm một nhóm mới** vào `src/cay-tien/phan-loai.js`: phải thêm tên nhóm
vào bảng từ cấm ở `src/cay-tien/buc-tuong.js`. Quên thì có một mục kiểm bắt
được (`Bảng từ cấm phủ HẾT tên nhóm`) — đừng tắt mục kiểm ấy.

**Nếu một mục BỨC TƯỜNG trượt trong `npm run verify`:** đó là sự cố mức cao
nhất của phần này. Tìm đúng trang/chuỗi mà mục kiểm chỉ tên, gỡ chữ ra, chạy
lại. **Không** nới bảng từ cấm và **không** bỏ qua mục kiểm để lấy màu xanh.

---

## 5.11 Tổ thanh tra — 10 lượt mỗi ngày (VẬN HÀNH)

```bash
node bin/gita.js to-thanh-tra              # lượt 1
node bin/gita.js to-thanh-tra --luot=7     # đúng một lượt của lịch
node bin/gita.js to-thanh-tra --ca-ngay    # cả 10 lượt + biên bản ngày
```

Lịch 10 lượt: 06:00 · 08:00 · 10:00 · 12:00 · 14:00 · 16:00 · 18:00 · 20:00 ·
21:30 · 23:00. Năm phép soi **mức chặn** (bền vững dữ liệu · cổng quyền · bức
tường · riêng tư dữ liệu trẻ · luật đo) chạy ở **mọi** lượt; năm phép còn lại
luân phiên hai phép mỗi lượt.

**Đọc kết quả.**

| Nhãn | Nghĩa | Làm gì |
|---|---|---|
| `ĐẠT` | phép soi chạy xong, không thấy vấn đề | không làm gì |
| `CHẶN!` | phép soi mức chặn không đạt | **dừng việc**, mở sổ tay: `gita quy-trinh --thanh-tra=<mã>` |
| `XEM` | phép soi mức báo không đạt | xử trong ca trực |

**Lượt không chạy KHÔNG phải lượt sạch.** Biên bản ngày đếm cả lượt thiếu và
tính đó là không đạt. Một phép soi hỏng, ném lỗi, hoặc trả về sai định dạng
cũng tính là KHÔNG ĐẠT — vì phép soi im lặng vì hỏng trông giống hệt phép soi
im lặng vì sạch.

**Tổ thanh tra không tự vá.** Nó ghi biên bản và báo động. Sửa là việc của
người có thẩm quyền. Đừng thêm bước tự sửa vào tổ thanh tra: một bộ máy vừa
phát hiện vừa tự sửa là một bộ máy có thể tự sửa cả phát hiện của mình.

---

## 5.12 Sổ tay trực sự cố

```bash
node bin/gita.js quy-trinh                    # cả 8 kịch bản, đỏ trước
node bin/gita.js quy-trinh --ma=KB01          # một kịch bản
node bin/gita.js quy-trinh --thanh-tra=TT09   # kịch bản mà TT09 có thể phải mở
```

Mỗi kịch bản trả lời bốn câu theo đúng thứ tự **DẤU HIỆU → LÀM NGAY → AI QUYẾT
→ CẤM LÀM**. Lúc cuống, đọc mục **CẤM LÀM trước** rồi mới làm LÀM NGAY — mục
đó chính là những việc mà người trực hay làm và làm hỏng thêm.

| Mã | Kịch bản | Mức |
|---|---|---|
| KB01 | Nghi ngờ mất dữ liệu | Đỏ — 15 phút |
| KB02 | Rò rỉ dữ liệu ra ngoài | Đỏ — 15 phút |
| KB03 | Chi phí mô hình tăng đột biến | Cam — trong ca |
| KB04 | Sàng lọc tâm lý báo cần người lớn ngay | Đỏ — 15 phút |
| KB05 | Học liệu sai đáp số lọt ra ngoài | Đỏ — 15 phút |
| KB06 | Trợ lý AI bị treo | Cam — trong ca |
| KB07 | Gia đình yêu cầu xoá dữ liệu | Vàng — 24 giờ |
| KB08 | Lượt thanh tra không chạy | Vàng — 24 giờ |

**Ba việc CẤM hay bị làm nhất:**

1. **Khởi động lại máy chủ "xem thử"** khi nghi mất dữ liệu. Mỗi lần khởi động
   có thể nén ảnh chụp và cắt mất nhật ký còn tốt. Sao lưu `data/db/` TRƯỚC.
2. **Nới bảng từ cấm hoặc danh sách cổng công khai** để mục kiểm xanh lại. Mục
   kiểm đang nói đúng; nới nó là gỡ cảm biến báo cháy cho chuông im.
3. **Coi lượt thanh tra không chạy là lượt sạch.** Hai thứ đó trông giống hệt
   nhau và khác hẳn nhau.

Khi sửa xong, thêm/đổi kịch bản thì mọi đường dẫn tệp khai trong `lienQuan`
phải tồn tại thật — có mục kiểm `SỔ TAY` trong `npm run verify` canh việc đó.

---

## 5.13 Dòng tiền — trần chi và ba chỗ mù

```bash
node bin/gita.js dong-tien
```

Lệnh in **khoản chưa nối nguồn trước, số liệu sau**. Đó không phải lỗi trình
bày: người đọc phải biết bảng này chưa thấy gì trước khi tin những gì nó thấy.

**Ba khoản không có số, và không được bịa:** doanh thu · nhân sự · hạ tầng.
Sổ cái **từ chối ghi** vào ba khoản ấy cho tới khi có nguồn thật. Ai hỏi
"tháng này lãi bao nhiêu", câu trả lời đúng là *hệ thống chưa nối dữ liệu
thanh toán, nên không biết*.

**Trần chi chặn cứng.** Vượt `NGAN_SACH_NGAY_USD` hoặc `NGAN_SACH_THANG_USD`
là khoản chi bị **từ chối tại chỗ**, không phải ghi cảnh báo rồi cho qua.

**Khi chi phí tăng đột biến:** mở `gita quy-trinh --ma=KB03`. Đừng nâng trần
để hệ thống chạy tiếp — nâng trần trước khi biết vì sao tăng là cách chắc chắn
nhất để tăng tiếp.

---

## 5.14 Lưới 50 ô — trả lời "con tôi đang ở đâu"

```bash
curl -H "Authorization: Bearer <token>" localhost:9999/cay-gia-tri/luoi
curl -H "Authorization: Bearer <token>" localhost:9999/cay-gia-tri/o/3/6
```

Năm tầng × mười cấp. Cấp 10 của tầng k nối thẳng vào cấp 1 của tầng k+1.

**Chỉ 15 ô được dùng để báo đã đạt** — cấp 3, 6 và 10 của mỗi tầng, vì chỉ
những ô ấy có mốc cứng đo bằng máy từ dữ liệu học thật. 35 ô còn lại là mốc
mềm và tự mang câu *"KHÔNG dùng để báo cáo đã đạt"*.

**Khi nói chuyện với gia đình:** nói theo mốc cứng gần nhất đã qua, không nói
theo ô mềm đang ở. "Con đã qua mốc cấp 6 tầng 2" là câu có bằng chứng; "con
đang ở cấp 8" thì không.

**Trang `/nha/cay-gia-tri` vẽ lưới ấy thành một cái cây**: thân là con đường
chung, bốn nhánh lớn là tầng 1–4, tán là tầng 5. Mỗi cấp một chiếc **lá** —
nhiệm vụ con làm — và một **quả** — thứ gia đình nhận được. Quả **chín** (màu
cam) là cấp có bằng chứng đo bằng máy; quả **non** (màu nhạt) là mốc mềm.

Rê chuột vào một quả thì hiện cả lá lẫn quả của cấp đó. Trên điện thoại không
có chuột, nên bảng chữ ngay dưới hình mới là bản đầy đủ — mở từng tầng ra đọc.

**Khi thêm hoặc sửa một cấp:** sửa `LA_VA_QUA` trong `src/matran/cay-50.js`.
Mục kiểm `HÌNH CÂY` đòi cả 50 lá và 50 quả phải **khác nhau** và đòi không quả
nào hứa bằng điểm số. **CẤM** viết một câu khuôn rồi thay tên tầng: một cái cây
mà lá nào cũng giống lá nào thì nhìn là biết không ai trồng nó thật.

---

## 5.15 Biểu mẫu khu riêng — hai lớp chống giả mạo

Khu riêng `/nha` xác thực bằng **cookie**, nên trình duyệt gửi cookie kèm mọi
biểu mẫu — kể cả biểu mẫu nằm trên một trang web lạ. Không chặn thì một trang
bất kỳ có thể nộp hộ bài sàng lọc tâm lý của một đứa trẻ đang đăng nhập.

| Lớp | Chặn bằng gì | Hỏng khi nào |
|---|---|---|
| 1 — dấu vết nguồn | `Sec-Fetch-Site`, `Origin` vs `Host` | trình duyệt không gửi tiêu đề ấy |
| 2 — vé trong biểu mẫu | ô ẩn `ve` = HMAC của token phiên | trang lộ mất cookie của gốc này |

**Khi người dùng báo "nộp bài báo lỗi 403".** Gần như luôn là một trong ba:

1. **Trang mở quá lâu rồi máy chủ khởi động lại.** Phiên mất, vé cũ vô hiệu.
   Bảo họ **mở lại trang và nộp lần nữa** — thông báo trên màn hình đã nói đúng
   câu ấy. Đây là hành vi đúng, không phải lỗi.
2. **Họ đăng nhập ở tab khác bằng tài khoản khác.** Vé trong tab cũ thuộc phiên
   cũ. Mở lại trang.
3. **Có thật một trang ngoài đang cố nộp hộ.** Xem nhật ký: cùng một tài khoản,
   403 liên tục, `Sec-Fetch-Site: cross-site`. Đây là sự cố — mở `KB02`.

**CẤM** làm ba việc sau để "cho hết lỗi 403":

- **CẤM** bỏ lớp 2 vì "đã có lớp 1 rồi". Hai lớp hỏng theo hai cách khác nhau;
  bỏ một lớp là quay về một điểm hỏng duy nhất.
- **CẤM** cho qua khi vé thiếu. Vé thiếu và vé sai phải bị chặn như nhau.
- **CẤM** đặt `CSRF_SECRET` là một chuỗi ngắn cho dễ nhớ. Dưới 32 ký tự thì hệ
  thống bỏ qua và tự sinh khoá — khai một khoá ngắn chỉ tạo cảm giác an toàn.

**Thêm biểu mẫu mới vào khu riêng** thì phải nhúng `${CGM.oAn(nguoi && nguoi.ve)}`
ngay sau thẻ `<form>`. Quên thì mục kiểm `VÉ BIỂU MẪU` trong `npm run verify`
gọi đúng tên tệp và số dòng — đừng tắt mục kiểm ấy.

---

## 5.16 Hệ thống trải nghiệm khách hàng — dùng hằng ngày

```bash
gita trai-nghiem                                        # khung 10 tầng, lưới, bảng đo
gita trai-nghiem --o=T4C6 --tinh-huong=TH36 --kenh=K3   # tra đúng một ô
gita trai-nghiem --phuc-hoi                             # thang phục hồi 10 bước
```

**Cách dùng khi có việc xảy ra.** Ba câu hỏi, theo đúng thứ tự:

1. **Em đang ở ô nào của lưới 50 ô?** (`T{tầng}C{cấp}`)
2. **Đây là tình huống nào trong bốn mươi?** (`TH01`–`TH40`)
3. **Việc này nói qua kênh nào?** (`K1` trong buổi · `K2` tin nhắn · `K3` gọi ·
   `K4` gặp · `K5` báo cáo chặng)

Ba câu ấy ra đúng một ô, và ô ấy cho: **NỀN → TỐT → WOW**, người làm, hạn chót,
**CẤM LÀM**, và đo bằng gì.

**Đọc mục CẤM LÀM trước mục LÀM NGAY.** Phần lớn khách hàng mất không phải vì
sự cố, mà vì cách người trực xử lý sự cố.

### Khi lệnh trả về "ô này bị chặn"

Không phải lỗi. Kênh đang chọn không hợp với mức khẩn của tình huống — lệnh nói
luôn phải dùng kênh nào. Ví dụ: sàng lọc tâm lý mức đỏ (`TH14`) **không** được
báo qua báo cáo định kỳ (`K5`); phải gọi (`K3`) hoặc gặp (`K4`).

**CẤM** lách bằng cách đổi mức khẩn của tình huống cho vừa với kênh mình muốn dùng.

### Mức WOW — ba luật cứng

1. **Wow đứng trên NỀN.** Cổng từ chối trả wow khi nền của chính ô ấy chưa xong,
   và nói ra phải làm gì trước. Đừng lách cổng.
2. **Nhóm cảm xúc và an toàn KHÔNG có wow.** Làm đúng mức nền rồi chuyển cho
   người đã được chỉ định. Sáng tạo ở đây là đem một đứa trẻ ra thử nghiệm.
3. **Wow phải rẻ.** Mọi cách wow trong hệ thống đều tốn một hai phút tra cứu,
   không tốn tiền. Wow tốn tiền thì lần sau không ai làm, và nó thành lời hứa
   hụt.

### Mười một tình huống máy KHÔNG canh hộ

29/40 tình huống có tín hiệu máy tự thấy. **11 tình huống còn lại chỉ tồn tại
khi có người báo vào** — em tự nói mình chán, gia đình đang ép điểm, gia đình
hỏi tiến độ, gia đình muốn đổi người đồng hành, gia đình nói muốn dừng…

`gita trai-nghiem` in ra đủ danh sách 11 ấy. **Đừng xếp lịch trực như thể máy
đang trông cả bốn mươi.** Đó là cách tạo ra một đêm không ai trông.

### Trao quyền — biết trần của mình trước khi hứa với khách

9/10 quyền dùng được ngay, mỗi quyền một trần cứng (mở lại tối đa 3 bài/tuần,
giãn lịch tối đa 2 tuần, tặng tối đa 2 buổi kèm/chặng…).

**Quyền Q08 — hoàn tiền hoặc bù bằng tiền — CÒN TREO**, vì sổ cái dòng tiền
chưa nối dữ liệu thanh toán. **CẤM hứa mức bù bằng tiền** khi chưa có người có
thẩm quyền xác nhận: nói được rồi rút lại còn tệ hơn không nói.

### Thang phục hồi — thứ tự là bắt buộc

Lệnh `--phuc-hoi` in mười bước. Bộ đếm **từ chối nhảy bước** và nói rõ đang
thiếu bước nào.

Bước hay bị nhảy nhất là **bước 3 (công nhận cảm xúc)** — người trực muốn sang
ngay bước 6 (giải pháp). Nhảy chỗ đó thì giải pháp đúng cũng bị khách từ chối,
vì lúc ấy họ đang cần được công nhận, chưa cần được sửa.

### Bảng đo — đọc phần chưa đo được trước

Năm nhóm chỉ số **chưa nối nguồn**: giữ chân, CLV, giới thiệu, hài lòng, chi
phí phục vụ. **CẤM** đưa ước lượng của năm nhóm ấy vào bất kỳ báo cáo nào. Ai
hỏi "tháng này giữ chân bao nhiêu phần trăm", câu trả lời đúng là *hệ thống
chưa đo được, và đây là thứ cần nối*.

Mô-đun đo lường là **NỘI BỘ**, nằm sau bức tường cùng phía với Cây Tiền. Không
dán đầu ra của nó vào tài liệu gửi gia đình.

---

## 5.17 Thẻ điểm cân bằng — dùng theo nhịp, không dùng theo hứng

```bash
gita the-diem                 # thẻ điểm đầy đủ, phần CHƯA BIẾT in trước
gita the-diem --chuoi=QT4     # chuỗi nhân quả từ một mục tiêu
```

**Ba nhịp, ba câu hỏi. Đừng trộn chúng vào một cuộc họp** — trộn xong thì chỉ
còn lại câu dễ nhất: *"tháng này số có đẹp không"*.

| Nhịp | Hỏi gì | Xem gì | Được quyết gì |
|---|---|---|---|
| **Ngày** — máy chạy | Có gì đang hỏng ngay bây giờ? | 10 lượt thanh tra, chi phí so trần | dừng việc nếu mục soi mức chặn đỏ |
| **Tháng** | Chỉ báo **dẫn dắt** đi hướng nào? | 21 chỉ báo dẫn dắt, tình huống đỏ/cam | đổi cách làm ở tầng quy trình |
| **Quý** | **Giả thuyết nhân quả** còn đúng không? | bản đồ, chỉ báo kết quả, sổ rủi ro, nghĩa vụ pháp lý | **sửa bản đồ** |

**Chỉ nhịp QUÝ được sửa bản đồ.** Rà soát mà không bao giờ sửa bản đồ thì đó là
báo cáo, không phải quản trị chiến lược.

### Sáu câu hỏi bắt buộc của rà soát quý

Đây là phần hay bị bỏ nhất, vì nó đòi người chủ trì thừa nhận một giả thuyết
của chính mình có thể sai. `gita the-diem` in đủ sáu câu; hai câu khó nhất:

- *Mũi tên nào ba tháng qua KHÔNG có số liệu nào ủng hộ?*
- *Chỉ báo dẫn dắt nào đã cải thiện mà chỉ báo kết quả phía sau KHÔNG nhúc
  nhích?* — nếu có thì giả thuyết nhân quả giữa chúng sai, hoặc độ trễ dài hơn
  ta nghĩ, và **phải nói ra mình chọn cách hiểu nào**.

### Ba luật CẤM của tầng này

1. **CẤM đặt chỉ tiêu cho thước đo chưa nối nguồn.** 11/36 thước đo hiện chưa
   có nguồn và vì thế không có chỉ tiêu. Thêm một con số vào đó là biến thẻ
   điểm thành văn bản trang trí. Mục kiểm chặn.
2. **CẤM viết biện pháp chặn rủi ro bằng động từ mềm** — *tăng cường, chú
   trọng, nâng cao*. Mỗi biện pháp phải là một tệp mã có thật, và mục kiểm đi
   kiểm tệp ấy tồn tại.
3. **CẤM để hệ thống tự tuyên bố tuân thủ pháp luật.** 0/6 nghĩa vụ hiện có
   người ký nhận → trạng thái là **chưa xác nhận**, không phải "đạt".

### Khi thêm một mục tiêu vào bản đồ

Phải nối nó vào chuỗi ở **cả hai đầu**: có cạnh đi ra, và có cạnh đi vào (trừ
tầng nền). Cạnh chỉ đi lên **một** tầng. Muốn nối thẳng lên tài chính thì phải
dùng cạnh `chan`, và cạnh ấy **chỉ** trỏ được vào mục tiêu phòng tổn thất.

Mục kiểm `THẺ ĐIỂM` bắt cả năm luật. Nó đã bắt chính người viết năm lần trong
bản dựng đầu — đừng tắt nó để đỡ phiền.

### Những gì tầng này CHƯA làm được

`gita the-diem` in danh sách đầy đủ ở đầu màn hình, **trước** mọi con số. Đọc
phần ấy trước. Tóm tắt: chưa đo được doanh thu, biên lợi nhuận, runway, tỉ lệ
giữ chân, tỉ lệ giới thiệu, tổn thất thực tế; chưa theo dõi được các cuộc rà
soát đã diễn ra hay chưa; ba rủi ro chưa có mục soi tự động nào canh.

---

## 5.18 Cổng tuổi — dùng TRƯỚC khi dựng, không phải sau

```bash
gita an-toan                                              # ba cổng + bảy câu hỏi
gita an-toan --tinh-nang=GHEP_BAN_HOC --tuoi=14 --cach-biet=TU_KHAI
```

**Dựng tính năng mới chạm tới người khác? Khai vào sổ TRƯỚC khi viết dòng mã
đầu tiên.** Thêm một dòng vào `TINH_NANG` trong `src/an-toan/cong-tuoi.js` với
đúng `mucTiepXuc`. Không khai thì cổng **TỪ CHỐI** — mặc định là đóng.

Bốn mức tiếp xúc, và mức quyết định cổng nào phải qua:

| Mức | Nghĩa | Cần gì |
|---|---|---|
| `KHONG` | học viên làm một mình | không cần tuổi |
| `TRONG_NHA` | giáo viên, cố vấn, gia đình | cần biết tuổi |
| `NGUOI_LA` | **chạm tới người lạ** | tuổi **được xác nhận** + đồng ý người giám hộ nếu dưới 16 |
| `CONG_KHAI` | ra ngoài Internet, ghi giọng/mặt | như trên, và cả 16–17 cũng cần |

### Ba câu CẤM của cổng tuổi

1. **CẤM đoán tuổi** từ lớp đang học, từ cách viết, từ giọng nói. Chưa biết
   tuổi là một câu trả lời, và câu ấy là KHÔNG.
2. **CẤM coi lời khai là bằng chứng.** Mở cổng bằng lời khai là mở cho bất kỳ
   ai khai bất kỳ tuổi nào. Tính năng chạm người lạ đòi nhà trường hoặc tổ
   chức xác nhận.
3. **CẤM coi một ô tích là sự đồng ý.** Đồng ý phải có TÊN người giám hộ, QUAN
   HỆ với học viên, LÚC đồng ý, và ĐƯỜNG RÚT LẠI. Thiếu một trong bốn thì cổng
   từ chối.

### Sáu tính năng chưa dựng đã khai sẵn

Ghép bạn học · hỏi đáp ẩn danh · lớp học chung · sự kiện trực tuyến · học bằng
giọng nói · học viên đăng nội dung ra ngoài.

Chúng chưa tồn tại trong hệ thống. Khai sẵn để **ngày ai đó dựng thì cổng đã
đứng sẵn** — chứ không phải dựng cổng sau khi đã có một đứa trẻ đi qua.

### Bảy câu hỏi trước khi dựng bất kỳ tính năng nào

`gita an-toan --chi-tiet` in đủ bảy, kèm chỗ hỏng gốc mà mỗi câu sinh ra từ đó.
**Câu chưa trả lời KHÔNG mặc định là "không áp dụng"** — cả bảy chỗ hỏng gốc
đều nằm trong những tính năng mà người viết chắc chắn nghĩ là không liên quan.

### Khi thêm một việc cần khoá

Thêm vào bảng `VIEC` trong `src/an-toan/khoa-rieng.js`. **CẤM dùng ké khoá của
việc khác** — lộ một nơi là mất cả hai, và hai việc có vòng đời xoay khoá khác
nhau. **CẤM cắt bí mật bằng `slice()`**, **CẤM đệm bí mật ngắn cho đủ dài**.

### Nếu bộ quét cụm từ cấm báo động

Xem chỗ khớp có nằm **trọn trong ranh giới từ** không. Bộ quét đã được sửa để
không khớp cắt ngang thân từ, nhưng nếu gặp một ca lạ thì **kiểm tra chữ trước,
đừng nới bảng cấm**. Một lần nới là cả bức tường mất giá trị.

---

## 6. Sao lưu

| Thứ cần giữ | Ở đâu | Cách sao lưu |
|---|---|---|
| Dữ liệu nghiệp vụ | `data/db/` | chép cả thư mục khi hệ thống đang dừng, hoặc sau `POST /snapshots/take` |
| Ảnh chụp hệ thống | `data/snapshots/` | chép cả thư mục |
| Bí mật Super Admin | ngoài hệ thống | két sắt, không lưu trên cùng máy |

### Ảnh chụp định kỳ — đường lui DUY NHẤT khi tệp dữ liệu hỏng

**Tự chụp mỗi 15 phút** (`SNAPSHOT_INTERVAL_MS`), giữ theo kiểu ông–cha–con:

| Tầng | Giữ gì | Cho chiều sâu |
|---|---|---|
| **con** | 12 ảnh gần nhất, không chia chu kỳ | vài giờ vừa rồi, rất dày |
| **cha** | **một ảnh mỗi GIỜ** | 24 giờ |
| **ông** | **một ảnh mỗi NGÀY** | **30 ngày** |

Ảnh chụp tay giữ vĩnh viễn. Tổng khoảng **60 ảnh × ~1,8 MB ≈ 110 MB** — có
trần, không phình vô hạn.

**`SNAPSHOT_AUTO=0` TẮT HẲN việc tự chụp.** Chỉ dùng cho môi trường chạy thử.
Tắt trên máy chạy thật nghĩa là hỏng tệp dữ liệu vào một ngày không ai bấm nút
thì **KHÔNG CÓ GÌ để lui về**. Tổ thanh tra TT01 soi việc này mỗi lượt và báo
KHÔNG ĐẠT nếu bộ đếm giờ không chạy.

> **Ba lỗi của chính cơ chế này, tìm ra trong một lượt thanh tra Super Admin.**
>
> **Một:** `SnapshotStore` được dựng với `autoStart` mặc định `false` và không
> ai gọi `.start()`. Chính sách giữ ảnh nằm đủ trong mã và trong sổ tay, còn số
> ảnh tự chụp là **không**.
>
> **Hai:** chính sách mang đúng tên *ông–cha–con* mà không làm đúng việc ấy —
> mỗi tầng giữ N ảnh **mới nhất** trong cửa sổ của nó, mà ba cửa sổ lồng nhau,
> nên hợp của chúng chỉ còn "30 ảnh mới nhất". Mô phỏng 30 ngày chụp mỗi 15
> phút: **2.881 ảnh → dọn còn 30 → ảnh cũ nhất 7,3 GIỜ trước**, trong khi sổ
> tay hứa 30 ngày. Một hỏng dữ liệu phát hiện sau hai ngày thì không còn gì để
> lui về.
>
> **Ba:** sau khi vá hai lỗi trên, chính tổ thanh tra lại báo CHẶN — bộ hẹn giờ
> đã chạy, nhật ký đã in *"tự động mỗi 15 phút"*, mà số ảnh trên đĩa vẫn là
> **không**. `setInterval` chỉ chạy lần đầu **sau** một chu kỳ. Hệ quả: suốt 15
> phút đầu sau mỗi lần dựng lại máy chủ — đúng lúc dễ hỏng nhất — không có gì
> để lui về; và máy chủ nào dựng lại dày hơn chu kỳ thì bộ hẹn giờ **không bao
> giờ** tới lượt, số ảnh đứng yên ở không vĩnh viễn.
>
> Nay `.start()` chụp ngay một ảnh nhãn `khoi-dong` nếu ảnh mới nhất đã quá một
> chu kỳ — dựng lại năm lần trong một phút cũng chỉ ra một ảnh.
>
> Cả ba đã sửa. Mục kiểm `SAO LƯU` mô phỏng 35 ngày và bắt hệ thống chứng minh
> chiều sâu ≥ 30 ngày trước khi cho qua; TT01 soi số ảnh thật trên đĩa mỗi lượt.
>
> Bài học chung của cả ba: **có cơ chế không bằng có kết quả.** Hai lỗi đầu
> tìm ra bằng cách đọc mã, lỗi thứ ba chỉ lộ ra khi chạy thật rồi đếm ảnh.

**Nén nhật ký ghi theo đúng năm bước, và thứ tự ấy là luật:** ghi tệp tạm →
`fsync` tệp tạm → đổi tên đè lên ảnh cũ → **`fsync` thư mục** → mới cắt nhật ký.
Bước `fsync` thư mục là bước hay bị quên nhất: `rename` nguyên tử không có
nghĩa là đã nằm trên đĩa, mục lục thư mục vẫn còn trong bộ đệm. Có mục kiểm
đọc thẳng `src/data/repository.js` để chắc thứ tự này không bị đảo lại.

**Thấy tệp `.snap.json.tmp` trong `data/db/`** là dấu hiệu hệ thống dừng giữa
lúc nén. **Sao lưu cả thư mục trước khi làm bất cứ gì** — tệp `.tmp` ấy có thể
là bản mới nhất còn nguyên. Mở `gita quy-trinh --ma=KB01`.

---

## 7. Kiểm tra sức khoẻ định kỳ

```bash
npm test          # 829 test, ~3 phút
npm run verify    # 143 mục kiểm định, in ra số liệu thật
npm run smoke     # 326 lượt gọi API trên máy chủ thật
npm run load      # kiểm thử tải 5 kịch bản
node bin/gita.js to-thanh-tra --ca-ngay   # 10 lượt thanh tra, kèm biên bản ngày
```

Chạy `npm run verify` sau mỗi lần nâng cấp. Có mục nào trượt là thoát mã 1,
dùng được trong CI.

---

## 8. Biến môi trường hay dùng

| Biến | Mặc định | Khi nào đổi |
|---|---|---|
| `PORT` | 9999 | cổng bị chiếm |
| `HOST` | 0.0.0.0 | muốn chỉ nghe trên máy thì đặt `127.0.0.1` |
| `DB_DIR` | `data/db` | để dữ liệu ở ổ khác |
| `SNAPSHOT_DIR` | `data/snapshots` | để ảnh chụp ở ổ khác |
| `AGENT_GLOBAL_CONCURRENCY` | 64 | máy mạnh hơn, hoặc mô hình chịu được nhiều hơn |
| `AGENT_TOKENS_PER_MIN` | 600000 | theo hạn mức thật của nhà cung cấp mô hình |
| `GROQ_API_KEY` … | trống | có khoá thì chạy mô hình thật; không có thì chạy ngoại tuyến |
| `SEED_BANK` | bật | đặt `0` nếu không muốn tự gieo kho lần đầu VÀ không muốn tự bổ sung khi khởi động |
| `SEED_PER_SLOT` | 3 | số bài mỗi ô (dạng × tầng); tăng nếu muốn kho dày hơn |
| `ROUTER_SEED` | cố định | đổi khi muốn chuỗi thăm dò khác (vẫn lặp lại được) |
| `VOICE_PROVIDER` | `offline` | chỉ đổi khi đã cài piper hoặc có hợp đồng đám mây |
| `VOICE_WATERMARK` | bật | **không nên tắt** — tắt là bỏ dấu AI, trái Luật 116/2025/QH15 |
| `VOICE_RETENTION_DAYS` | 180 | theo chính sách lưu trữ của đơn vị |
| `VOICE_MINOR_AGE` | 18 | ngưỡng bắt buộc có đồng ý của người giám hộ |
| `PHOTO_MODE` | `mien-phi` | `tai-may` nếu muốn ảnh không rời khỏi máy (cần ComfyUI + GPU) |
| `PHOTO_CAMERA` | `Fujifilm` | đổi chất màu mặc định của xưởng ảnh |
| `PHOTO_MAX_MP` | 36 | hạ xuống nếu máy ít RAM — đây là trần bảo vệ bộ nhớ |
| `FAL_KEY` | trống | có khoá thì dựng được video thật; không có thì chỉ dựng bản nháp |
| `FILM_MAX_USD` | 25 | trần chi tiêu cho MỘT phim — vượt là chặn |
| `FILM_REQUIRE_CONFIRM` | bật | **để nguyên** nếu không muốn phim tự chạy đốt tiền |
| `VOICE_ALLOW_UNOFFICIAL` | tắt | **để nguyên**. Bật là dùng cổng nội bộ của Google Dịch: không hợp đồng, chặn theo IP, vi phạm điều khoản |
| `CORS_ORIGINS` | **trống = đóng hẳn** | chỉ khai khi thật sự có trang khác cần gọi API; khai đích danh từng gốc. **CẤM đặt `*`** — phiên đi bằng cookie |
| `CSRF_SECRET` | **trống = tự sinh mỗi lần khởi động** | chỉ khai khi phiên được lưu xuống đĩa hoặc chạy nhiều tiến trình sau cân tải. Khai thì phải ≥ 32 ký tự; ngắn hơn bị bỏ qua |
| `NGAN_SACH_NGAY_USD` | 5 | trần chi mô hình MỘT ngày — vượt là **chặn cứng**, không phải cảnh báo |
| `NGAN_SACH_THANG_USD` | 100 | trần chi mô hình MỘT tháng |

Toàn bộ danh sách ở `.env.example`.

---

## 8a. Giọng nói giảng viên

**Chạy được ngay, không cần cài gì thêm.** Bộ tổng hợp mặc định nằm trong mã
nguồn, không gọi ra mạng. Kiểm tra nhanh:

```bash
curl -s localhost:9999/voice/status | head -30
curl -s "localhost:9999/voice/say?text=Ch%C3%A0o%20em&agentId=MTH-001" -o thu.wav
```

**Muốn giọng hay hơn** (tuỳ chọn, không bắt buộc):

```bash
# piper — nơ-ron, chạy cục bộ, miễn phí, không gửi dữ liệu ra ngoài
PIPER_BIN=/opt/piper/piper
PIPER_MODEL=/opt/piper/vi_VN-vais1000-medium.onnx
VOICE_PROVIDER=piper
```

Sau khi đổi, `GET /voice/status` phải báo `providers.effective` đúng tên nhà
cung cấp mới. Nếu vẫn là `offline` thì xem trường `reason` của nhà cung cấp đó —
nó nói thẳng thiếu tệp nào hay thiếu khoá nào.

**Việc phải làm hằng tháng:**

```bash
curl -s -XPOST localhost:9999/voice/retention-sweep -d '{}' -H 'content-type: application/json'
curl -s localhost:9999/voice/audit | head -20     # chain.ok phải là true
```

`chain.ok` là `false` nghĩa là nhật ký tạo giọng đã bị sửa từ bên ngoài — dừng
hệ thống và khôi phục từ ảnh chụp gần nhất.

**Khi học viên yêu cầu xoá dữ liệu giọng nói:**

```bash
curl -s -XPOST localhost:9999/voice/forget -H 'content-type: application/json' \
  -d '{"subjectId":"HV-001","reason":"phụ huynh yêu cầu"}'
```

Phản hồi trả về `receipt` — lưu mã này lại, đó là biên bản việc xoá.

**Khi bị hỏi "có phải giọng AI không":**

```bash
curl -s -XPOST localhost:9999/voice/verify-audio -H 'content-type: application/json' \
  -d "{\"audioBase64\":\"$(base64 -w0 thu.wav)\"}"
```

Trả `aiGenerated: true` kèm mã giọng và thời điểm tạo — dấu nằm trong chính
sóng âm, không phải trong tên tệp, nên đổi tên hay cắt bớt đầu đuôi vẫn đọc được.

---

## 8b. Hát và hậu kỳ phòng thu

**Hát được ngay, không cần cài gì.** Giai điệu gõ bằng ký âm chữ: tên nốt
(Việt hoặc Anh) + quãng tám + `:` + số phách.

```bash
curl -s -XPOST localhost:9999/voice/song/check -H 'content-type: application/json' -d '{
  "lyrics":"Quê hương là chùm khế ngọt",
  "melody":"sol4:1 la4:1 si4:1 đô5:1 si4:1 la4:2",
  "bpm":80, "range":"mezzo" }'
```

`check` chạy trước để soi: lời có khớp nốt không, bài dài bao nhiêu giây, có
phải dịch giọng không. Khớp rồi mới `sing`:

```bash
curl -s -XPOST localhost:9999/voice/song/sing -H 'content-type: application/json' -d '{
  "lyrics":"Quê hương là chùm khế ngọt",
  "melody":"sol4:1 la4:1 si4:1 đô5:1 si4:1 la4:2",
  "bpm":80, "style":"ballad" }' | python3 -c \
  "import json,sys,base64; d=json.load(sys.stdin); open('bai-hat.wav','wb').write(base64.b64decode(d['audioBase64']))"
```

Năm cách hát: `pop` · `ballad` · `dan-ca` · `thieu-nhi` · `doc-rap`.

**Lời và nốt phải khớp một–một.** Dấu `-` trong lời nghĩa là *luyến* — âm tiết
trước ngân tiếp sang nốt đó. Thiếu hay thừa lời thì `check` báo trước chứ
không để hát ra rồi mới biết.

**Dùng tệp MIDI thật:** gửi `midiBase64` thay cho `melody`. Hệ thống tự tách
bè cao nhất, tự chèn chỗ nghỉ lấy hơi. Ngược lại, `POST /voice/music/export-midi`
xuất giai điệu ra `.mid` để mở bằng phần mềm nhạc.

**Hậu kỳ** chạy sẵn sau mỗi lần nói và hát. Muốn xử lý lại một tệp có sẵn:

```bash
curl -s -XPOST localhost:9999/voice/studio/master -H 'content-type: application/json' \
  -d "{\"audioBase64\":\"$(base64 -w0 thu.wav)\",\"preset\":\"phat-thanh\"}"
```

Phản hồi có `before`/`after` bằng LUFS — đó là số để đối chiếu, không phải
cảm nhận. Tắt hậu kỳ bằng `VOICE_STUDIO=0` (chỉ nên dùng khi so sánh).

**Muốn giọng hay hơn nữa** thì cắm mô hình nơ-ron, không phải sửa mã:

```bash
ZEROTTS_DIR=/opt/zerotts          # hoặc ZEROTTS_URL=http://127.0.0.1:8080
VIENEU_URL=http://127.0.0.1:8090  # bài dài, có cảm xúc
OPENUTAU_BIN=/opt/OpenUtau/OpenUtau   # hát, có voicebank tiếng Việt
```

`GET /voice/pipelines` cho biết mỗi mục đích đang thật sự dùng bộ nào. Vẫn thấy
`offline` thì đọc trường `reason` của bộ đó — nó nói thẳng thiếu tệp nào.

**Đổi giọng một bản hát thật (SVC)** cần đủ hai phiếu đồng ý trước:

```bash
# phiếu của người hát gốc
curl -s -XPOST localhost:9999/voice/consent -H 'content-type: application/json' \
  -d '{"subjectId":"CA-SI-A","kind":"VOICE_RECORD","age":25,"documentRef":"HS-2026-01"}'
# phiếu của chủ giọng đích — bắt buộc có mã hồ sơ văn bản
curl -s -XPOST localhost:9999/voice/consent -H 'content-type: application/json' \
  -d '{"subjectId":"CA-SI-B","kind":"VOICE_CLONE","age":30,"documentRef":"HS-2026-02"}'
```

Thiếu một trong hai là hệ thống trả 403. Và ngay cả khi đủ hai phiếu, nếu chưa
cài So-VITS-SVC thì hệ thống nói thẳng là chưa làm được — **cố ý không có bản
thay thế nội bộ cho việc này**.

---

## 8c. Làm phim

**Chạy thử ngay, không tốn đồng nào:**

```bash
# 1. Lấy kịch bản mẫu và chuẩn bị (đọc kịch bản, phân cảnh, dự toán, soát)
curl -s localhost:9999/film/sample-script -o kich-ban.txt
PHIM=$(python3 -c "import json;print(json.dumps({'kichBan':open('kich-ban.txt').read()}))" \
  | curl -s -XPOST localhost:9999/film/prepare -H 'content-type: application/json' -d @- \
  | python3 -c 'import json,sys;print(json.load(sys.stdin)["duAn"])')

# 2. Dựng bản nháp: đường tiếng thật + bảng phân cảnh + phim nháp
curl -s -XPOST localhost:9999/film/projects/$PHIM/draft -d '{}' -H 'content-type: application/json' >/dev/null

# 3. Mở phim nháp trên trình duyệt
xdg-open "http://localhost:9999/film/projects/$PHIM/animatic"
```

Bước 1 và 2 **không gọi API nào**, không tốn tiền. Xem phim nháp là duyệt được
nhịp, biết cảnh nào thừa, chỗ nào hụt.

**Viết kịch bản đúng chuẩn** để hệ thống đọc được:

```
TIÊU ĐỀ: Tên phim
THỂ LOẠI: Chính kịch ngắn

CẢNH 1 - INT. LỚP HỌC CŨ - CHIỀU MUỘN

Đoạn tả bối cảnh. Đoạn tả nhân vật: tuổi, tóc, kính, trang phục, nét mặt.

TÊN NHÂN VẬT (hướng dẫn diễn): "Lời thoại."

HẾT
```

Hai chỗ hay sai: **tên nhân vật phải VIẾT HOA** (không thì bị coi là câu mô
tả), và **phải tả ngoại hình** (không thì mặt đổi giữa các cảnh — hệ thống sẽ
cảnh báo `NHÂN_VẬT_THIẾU_MÔ_TẢ`).

**Đọc kết quả soát chất lượng.** `qc.chan > 0` là **không được dựng** — dựng
lúc đó là đốt tiền vào cảnh hỏng. Ba lỗi hay gặp:

| Lỗi | Nghĩa | Sửa |
|---|---|---|
| `THOẠI_TRÀN_KHỎI_CẢNH` | Lời thoại đọc lâu hơn thời lượng cảnh | Cắt câu thoại, hoặc để hệ thống kéo dài cảnh |
| `PROMPT_THIẾU_KHOÁ_NGOẠI_HÌNH` | Cảnh này nhân vật sẽ thành người khác | Chuẩn bị lại dự án |
| `VƯỢT_NGÂN_SÁCH` | Dự toán vượt số tiền đã đặt | Đổi `uuTien` sang `tiet-kiem`, hoặc cắt cảnh |

**Dựng thật** cần `FAL_KEY` và **hai lần đồng ý**: hệ thống trả về dự toán
trước, chỉ chạy khi gửi lại `xacNhan: true`.

```bash
curl -s -XPOST localhost:9999/film/projects/$PHIM/render \
  -H 'content-type: application/json' -d '{"xacNhan":true}'
```

Vượt `FILM_MAX_USD` là chặn, không hỏi lại. Muốn dựng thử một cảnh trước thì
truyền `chiCanhQuay: ["q001"]`.

**Khi đã có clip**, tải `/film/projects/$PHIM/ffmpeg` và `/edl`: kịch bản ghép
tự kiểm tra thiếu clip rồi mới nối, và ghép luôn đường tiếng đã dựng.

---

## 8d. Xưởng ảnh

**Hậu kỳ chạy được ngay, không cần cài gì.** Không cần ffmpeg, không cần khoá,
không cần mạng:

```bash
# Chuẩn bị: prompt + bố cục + khung dựng
curl -s -XPOST localhost:9999/photo/prepare -H 'content-type: application/json' \
  -d '{"moTa":"Chân dung phụ nữ áo dài đỏ bên hồ sen","kieuAnh":"chan-dung","mayAnh":"Fujifilm","phim":"KodakPortra400"}'

# Xem khung dựng bố cục (SVG)
curl -s -XPOST localhost:9999/photo/framing -H 'content-type: application/json' \
  -d '{"moTa":"Chân dung","kieuAnh":"chan-dung"}' -o khung-dung.svg

# Hậu kỳ 8 tầng trên ảnh có sẵn
python3 -c "import json,base64;print(json.dumps({'anhBase64':base64.b64encode(open('anh.jpg','rb').read()).decode(),'mayAnh':'Fujifilm','phim':'KodakPortra400','chatLuong':'4k'}))" \
  | curl -s -XPOST localhost:9999/photo/enhance -H 'content-type: application/json' -d @- \
  | python3 -c "import json,sys,base64;d=json.load(sys.stdin);open('anh-hau-ky.jpg','wb').write(base64.b64decode(d['jpegBase64']));print(d['cham']['diem'],d['cham']['xep'])"
```

**Đọc được PNG và JPEG baseline.** Ảnh JPEG lũy tiến (progressive) và PNG xen
kẽ (interlaced) thì hệ thống nói thẳng là chưa hỗ trợ — xuất lại ảnh rồi gửi
lại, chứ không trả ảnh rác.

**Đọc kết quả soát.** Bảy trục, mỗi lỗi kèm đúng tham số cần chỉnh:

| Trục | Chuẩn | Lỗi thường gặp |
|---|---|---|
| Phơi sáng | trung bình 95–155 | ảnh tối hoặc cháy |
| Tương phản | độ lệch chuẩn ≥ 45 | ảnh bẹt |
| Dải tông | phân vị 1–99% ≥ 200 | thiếu đen sâu và trắng sạch |
| **Độ nét** | phương sai Laplace ≥ 120 | ảnh mềm |
| Cháy sáng | ≤ 0,5% điểm chạm trắng | mất chi tiết vùng sáng |
| Bẹp tối | ≤ 0,5% điểm chạm đen | mất chi tiết vùng tối |
| Cân bằng trắng | lệch ≤ 12 **so với chủ ý hồ sơ** | ám màu ngoài ý muốn |

Độ nét đo trên bản quy về cạnh dài 1024 nên **không phụ thuộc kích thước** —
phóng to ảnh không bị chấm oan là làm ảnh mờ đi.

**Muốn sinh ảnh mới** (không chỉ hậu kỳ) thì cần một cổng:

```bash
PHOTO_MODE=mien-phi     # Pollinations, không cần khoá, cần mạng ra ngoài
PHOTO_MODE=tai-may      # ComfyUI tại máy: 0 đồng, ảnh không rời khỏi máy, cần GPU
COMFYUI_URL=http://127.0.0.1:8188
```

`GET /photo/status` cho biết cổng nào đang dùng được và cổng nào thiếu gì.

---

## 8e. Bài giảng một lệnh

```bash
node bin/gita.js kiem-tra                        # máy này chạy được tới đâu
node bin/gita.js bai-giang "Phương trình bậc hai"
node bin/gita.js bai-giang --dang=M8.PYTAGO.UNGDUNG --chat-luong=full-hd --luu-khung
node bin/gita.js bai-giang --kich-ban=bai.txt --ra=./video
node bin/gita.js bai-giang "Đạo hàm" --dan-y     # xem dàn ý, chưa dựng
node bin/gita.js dang pytago                      # tìm dạng bài
```

Tệp ra: `data/bai-giang/BG-xxxx/bai-giang.avi` kèm `moc-thoi-gian.json`. Mở bằng
VLC, mpv hoặc Windows Media Player — không cần cài thêm gì.

| Chất lượng | Khung hình | Thời gian dựng | Dung lượng (bài ~1 phút) |
|---|---|---|---|
| `nhap` | 854×480 @8fps | ~3,5 giây | ~12 MB |
| `hd` (mặc định) | 1280×720 @10fps | ~7,5 giây | ~56 MB |
| `full-hd` | 1920×1080 @10fps | ~18 giây | ~120 MB |

MJPEG không nén liên khung nên tệp nặng; đổi lại là mở được ở mọi nơi mà không
cần codec. Cần tệp nhỏ thì nén lại bằng ffmpeg ở khâu phát hành.

**Qua API:**

```bash
curl -s localhost:8080/lesson/match?chuDe=phương%20trình%20bậc%20hai
curl -s -X POST localhost:8080/lesson/outline -d '{"maDang":"M9.PT2.GIAI"}' -H 'content-type: application/json'
curl -s -X POST localhost:8080/lesson/build   -d '{"maDang":"M9.PT2.GIAI","kho":"nhap"}' -H 'content-type: application/json'
curl -s localhost:8080/lesson/<id>/video -o bai.avi
```

Giao diện: `http://localhost:8080/ui/bai-giang.html`.

**Khi gặp sự cố**

| Hiện tượng | Nguyên nhân | Cách xử lý |
|---|---|---|
| `THIẾU_NGUỒN_NỘI_DUNG` | không truyền maDang/kichBan/chuDe | chọn một trong ba |
| `KHÔNG_CÓ_DẠNG_BÀI_KHỚP` | chủ đề nằm ngoài 104 dạng | xem `gita dang`, hoặc dùng `--kich-ban` |
| `KÝ_HIỆU_TOÁN_KHÔNG_ĐẠT` | nguồn nội dung còn `x^2`, `sqrt()` | sửa theo cột "cách sửa" trong lỗi trả về |
| Bài chỉ có lý thuyết | kho chưa có bài mẫu cho dạng đó | `POST /content/seed` cho dạng đó |
| `tiengHong > 0` | một đoạn không có lời giảng | kiểm tra trường `loiGiang` của đoạn |

## 8f. Chuẩn ký hiệu toán và nhà khoa học toán

```bash
node bin/gita.js giai "x^2 - 5x + 6 = 0"          # giải + trình bày + thẩm định
node bin/gita.js giai "2x^2 - 7x + 3 = 0" --tat-ca --ra=bai.md
node bin/gita.js soat bai.md                       # thẩm định 4 cấp một tệp
node bin/gita.js phieu "phương trình bậc hai" --so-bai=10 --ra=phieu.md
node bin/gita.js phuong-phap "em học trước quên sau"
```

`gita soat` thoát với **mã 1** khi bài không đạt — cắm thẳng vào CI để chặn học
liệu sai ký hiệu hoặc sai đáp số trước khi vào kho.

**Qua API** (nhánh `/scientist`, không phải `/math` — nhánh đó của bộ xác minh
học liệu):

```bash
curl -s -X POST localhost:8080/scientist/solve     -d '{"deBai":"x^2 - 5x + 6 = 0"}'     -H 'content-type: application/json'
curl -s -X POST localhost:8080/scientist/present   -d '{"deBai":"x^2 - 5x + 6 = 0","cach":2}' -H 'content-type: application/json'
curl -s -X POST localhost:8080/scientist/validate  -d '{"text":"<bài giải>"}'             -H 'content-type: application/json'
curl -s -X POST localhost:8080/scientist/normalize -d '{"text":"x^2 - 5x + 6 = 0"}'       -H 'content-type: application/json'
curl -s -X POST localhost:8080/scientist/worksheet -d '{"maDang":"M9.PT2.GIAI","soBai":8,"dangChu":true}' -H 'content-type: application/json'
```

Giao diện: `http://localhost:8080/ui/khoa-hoc-toan.html`.

**Đọc kết quả thẩm định**

| Cấp | Trượt nghĩa là | Việc phải làm |
|---|---|---|
| 1 · ký hiệu | còn `^`, `_`, `sqrt()`, `Δ`, lệnh LaTeX chưa dựng | chạy `/scientist/normalize`, hoặc sửa nguồn |
| 2 · văn phong | giọng máy, viết tắt, biểu tượng cảm xúc | viết lại theo lối sách giáo khoa |
| 3 · cấu trúc | thiếu Đề bài / Lời giải / Kết luận / Kiểm tra lại | bổ sung phần thiếu |
| 4 · nội dung | **đáp số sai** khi thay nghiệm ngược | giải lại — không có cách nào khác |

Trượt cấp 1, 2 hoặc 4 là **không đạt**; cấp 3 chỉ trừ điểm. Bài đã đạt vẫn nên
xem cột `canSua` — ở đó có những nhắc nhở không đủ nặng để đánh trượt.

**Giới hạn đã biết.** Bộ giải trực tiếp làm được phương trình một ẩn bậc ≤ 2 và
dò nghiệm bằng số cho bậc cao hơn. Đề bài lời văn (bài toán có lời, hình học,
chứng minh) thì trả `KHÔNG_RÚT_ĐƯỢC_PHƯƠNG_TRÌNH` — cần người viết phương trình
ra, hoặc cần khoá mô hình ngôn ngữ. Hệ thống nói thẳng chỗ nó không làm được
chứ không đoán bừa một lời giải.

## 8g. Chuẩn tiếng Việt cho học liệu

```bash
node bin/gita.js tieng-viet de-bai.md --lop=3     # soát bốn cấp
node bin/gita.js tieng-viet "đoạn văn" --lop=5 --sua
node bin/gita.js am-tiet "Giải phương trình"      # tách năm thành phần âm tiết
node bin/gita.js soi-kho                          # soi cả kho, chấm theo đúng lớp
node bin/gita.js bai-ngon-ngu "phương trình bậc hai" --ra=bai.md
```

`gita tieng-viet` thoát **mã 1** khi văn bản không đạt — chặn được học liệu sai
chính tả hoặc quá sức lớp ngay tại khâu duyệt.

**Đọc kết quả**

| Cấp | Trượt nghĩa là | Việc phải làm |
|---|---|---|
| 1 · chính tả | sai luật cấu tạo âm tiết, đặt dấu lệch, teencode | sửa theo cột "cách sửa" |
| 2 · văn phong | giọng máy, viết tắt, biểu tượng cảm xúc | viết lại theo lối sách giáo khoa |
| 3 · độ đọc | câu dài quá trần của lớp | cắt câu dài thành hai câu |
| 4 · từ vựng | dùng từ lớp trên | giải thích ngay trong bài, hoặc thay từ |

Cấp 1 và cấp 2 là **lỗi chặn**. Cấp 3 chặn khi vượt trần; cấp 4 chỉ nhắc, vì
một thuật ngữ lớp trên đôi khi vẫn cần xuất hiện sớm kèm lời giải thích.

**Qua API**

```bash
curl -s -X POST localhost:8080/lang/check       -d '{"text":"<văn bản>","lop":3}' -H 'content-type: application/json'
curl -s -X POST localhost:8080/lang/normalize   -d '{"text":"<văn bản>","kieuDau":"moi"}' -H 'content-type: application/json'
curl -s -X POST localhost:8080/lang/syllables   -d '{"text":"nghiêng quỳ cua"}' -H 'content-type: application/json'
curl -s "localhost:8080/lang/term?tu=cạnh%20huyền"
curl -s "localhost:8080/lang/audit-bank?limit=50"
```

Giao diện: `http://localhost:8080/ui/tieng-viet.html`.

**Quy ước đặt dấu thanh.** Hệ thống dựng được cả hai kiểu và không coi kiểu nào
là sai; chỉ đòi một văn bản dùng nhất quán. Muốn đổi đồng loạt:
`gita tieng-viet <tệp> --sua --kieu-dau=moi` (kiểu sách giáo khoa: hoà, thuỷ)
hoặc `--kieu-dau=cu` (hòa, thủy).

**Giới hạn đã biết.** Bộ soát không biết "bàn" hay "bàng" mới đúng trong câu, và
bỏ lọt lỗi gõ mà phần sai lại tách được thành hai âm tiết hợp lệ. Đây là cái giá
để không báo oan từ mượn viết liền như "lôgarit", "vectơ" — báo oan chữ đúng làm
người dùng mất tin vào cả bộ soát.

## Dạy phát âm tiếng Anh

```bash
node bin/gita.js phat-am "sheep"                  # phiên âm IPA + dự báo lỗi người Việt
node bin/gita.js phat-am "I can see a sheep."     # phiên âm cả câu, chỉ ra chỗ nối âm
node bin/gita.js phat-am "water" --giong=my       # giọng Mỹ
node bin/gita.js cap-am                           # liệt kê các thế đối lập có cặp luyện
node bin/gita.js cap-am "iː" "ɪ"                  # dựng nguyên một tiết dạy phân biệt hai âm
node bin/gita.js doc-anh "Are you a student?" --ra=hoi.wav
node bin/gita.js doc-anh --cap sheep ship --ra=cap.wav   # đọc chậm một cặp tối thiểu
```

**Đọc trường `nguồn`.** Mỗi dòng phiên âm nói rõ nó từ đâu ra:

| nguồn | nghĩa | tin được tới đâu |
|---|---|---|
| `từ điển` | chép tay trong bảng 464 từ | chắc chắn |
| `từ điển + hậu tố "-es"` | tách hậu tố rồi tra gốc | chắc chắn |
| `luật` | suy bằng 147 luật chữ–âm | đúng phần lớn, **cần người rà lại** |

Gặp `luật` ở một từ hay dùng thì nên bổ sung hẳn vào từ điển
(`src/voice/en-phonemizer.js`, biến `TU_DIEN`) rồi chạy `npm test` — có sẵn
một test chặn mọi ký hiệu nằm ngoài bảng âm vị, nên gõ sai IPA sẽ trượt ngay.

**Qua API**

```bash
curl -s "localhost:8080/en/phien-am?tu=sheep"
curl -s "localhost:8080/en/chan-doan?tu=thirteenth"
curl -s "localhost:8080/en/the-doi-lap"
curl -s "localhost:8080/en/bai-hoc-am?am1=i%CB%90&am2=%C9%AA"
curl -s "localhost:8080/en/doc?text=Hello%20there." --output hello.wav
curl -s "localhost:8080/en/doc-cap?tu1=sheep&tu2=ship" --output cap.wav
curl -s "localhost:8080/en/bai-nghe/<itemId>" --output nghe.wav
```

**Bài nghe trong kho.** Đề bài nghe đánh dấu lời thoại bằng `[NGHE]…[/NGHE]`.
Học viên phải NGHE chứ không đọc chữ, nên khi hiển thị cho học viên thì che
khối đó đi và phát tệp WAV lấy từ `/en/bai-nghe/<itemId>`.

**Giới hạn đã biết.** Bộ phiên âm không phân biệt được từ đồng tự khác âm theo
ngữ cảnh — `read` ở thì hiện tại và quá khứ viết giống nhau, `record` làm danh
từ và động từ khác trọng âm. Học liệu tiếng Anh đã loại sẵn những câu nguồn đọc
được hai nghĩa như vậy, nhưng nếu tự gõ vào thì phải tự biết.


## 8h. Trang công khai và SEO

Trang công khai bật sẵn. Tắt bằng `WEB_PUBLIC=0` (chỉ còn API và giao diện nội bộ).

**Trước khi phát hành, bắt buộc đặt tên miền thật:**

```bash
export WEB_BASE_URL=https://ten-mien-that.vn
```

Thẻ `canonical` và sơ đồ trang trỏ sai tên miền thì công cụ tìm kiếm bỏ qua
toàn bộ trang. Đây là sai sót tốn nhiều tháng nhất để phát hiện.

**Việc cần làm sau khi phát hành**

1. Khai báo `https://<tên miền>/sitemap.xml` trong Google Search Console.
2. Kiểm tra `robots.txt` không chặn nhầm nhánh cần thu thập.
3. Chạy `npm run verify` — mục "TRANG CÔNG KHAI" chấm lại 177 trang.
4. Đặt máy chủ sau lớp nén gzip hoặc brotli; trang là HTML thuần nên nén rất tốt.
5. Bật HTTPS và thêm `Strict-Transport-Security` ở lớp proxy.

**Thêm trang mới.** Sửa `src/web/site.js`, thêm đường dẫn vào `moiDuongDan()`.
Sơ đồ trang, bộ kiểm định SEO và bộ soát tiếng Việt tự động phủ trang mới —
không phải nhớ cập nhật ba chỗ.

## 8i. Màn hình hành trình của học viên

`http://localhost:8080/ui/hanh-trinh.html`

Nhập mã học viên là vào. Chưa có mã thì hệ thống tự mở hồ sơ. Màn hình gồm bốn
phần, tất cả dựng từ dữ liệu học thật:

1. **Bạn đang ở đâu** — tầng hiện tại, tầng đích, số dạng ở tầng này, sáu trục
   chân dung năng lực.
2. **Chu kỳ chín mươi ngày** — đang ở chặng nào, ngày thứ mấy, hệ thống đang
   dùng chiến lược gì và **vì sao**.
3. **Buổi học hôm nay** — thành phần buổi học (ôn lại, chỗ yếu, dạng mới) kèm
   tên dạng bài.
4. **Phiên luyện** — làm từng câu, chấm ngay, tổng kết bằng số thật: đã làm bao
   nhiêu, đúng bao nhiêu, mấy giây một câu.

Không có điểm thưởng ảo. Những con số hiện trên màn hình chính là những con số
hệ thống dùng để quyết định cho lên tầng hay chưa.

## 8j. Thanh tra định kỳ

```bash
node bin/gita.js thanh-tra
node bin/gita.js thanh-tra --nhiem-vu=SYSTEM_HARDENING
```

Huy động thật lực lượng trợ lý AI rồi soát toàn hệ thống, in một biên bản, thoát
mã 1 nếu có mục chặn. Chạy trước mỗi lần phát hành.

Mảnh việc bị **lá chắn chống nhiễu** từ chối (`CAPACITY_LIMIT`) không tính là
hỏng: đó là lá chắn đang làm đúng việc khi hệ thống đang tải. Điều bắt buộc là
không chồng chéo việc, không vi phạm Hiến pháp, và mọi mảnh bị từ chối đều có lý
do ghi rõ.

## 8k. Ban quản trị quan hệ gia đình — vận hành hằng ngày

### Buổi sáng: mở bàn điều khiển

```
http://<máy chủ>/quan-tri/crm
```

Cần tài khoản vai **ADMIN**. Màn hình này canh bằng chính bức tường Cây Tiền,
không lưu đệm, không cho công cụ tìm kiếm ghi nhận.

Đọc theo thứ tự này, đừng đọc theo thứ tự mục trên màn hình:

1. **Trạng thái** — cả ban có đang bị dừng không. Nếu có, đọc lý do trước khi
   làm bất cứ việc gì khác.
2. **Việc cần làm ngay** — đã sắp theo mức cấp bách thật.
3. **Việc hết hạn chờ duyệt** — con số này lớn hơn không nghĩa là **đã có
   người chờ mà không ai tới**, và những việc ấy đã KHÔNG chạy.

### Duyệt một việc

```bash
curl -X POST http://localhost:9999/crm/duyet/<mã>/quyet \
  -H "Authorization: Bearer <token admin>" \
  -H 'content-type: application/json' \
  -d '{"trangThai":"dong-y","ghiChu":"đã đối chiếu số tiền với sổ"}'
```

Tên người quyết lấy từ phiên đăng nhập, không nhận từ thân yêu cầu — một chữ
ký tự khai thì không phải chữ ký.

**Đã quyết thì không quyết lại.** Muốn đổi thì mở một việc mới. Lịch sử duyệt
là thứ người ta đọc lại khi có chuyện, và một bản ghi bị sửa đè thì không đọc
lại được nữa.

### Khi nghi có chuyện: dừng cả ban trước, tìm hiểu sau

```bash
curl -X POST http://localhost:9999/crm/quan-tri/dung \
  -H "Authorization: Bearer <token admin>" \
  -H 'content-type: application/json' \
  -d '{"lyDo":"nghi rò rỉ dữ liệu gia đình qua kênh tư vấn"}'
```

Dừng **không xoá gì**: việc đang chờ duyệt vẫn nằm nguyên, kho vẫn nguyên.
Chạy lại bằng `{"chayLai":true}`.

Muốn dừng đúng một trợ lý thì dùng `/crm/quan-tri/khoa`, và **phải ghi lý do**
— người đọc lại sáu tháng sau cần hiểu vì sao.

### Bốn luật bàn điều khiển KHÔNG nới được

> Bốn luật này nằm trong hàm, không nằm trong giao diện để "nhớ đừng bấm".
> Một luật chỉ dựa vào trí nhớ của người trực thì không phải luật.

1. **Chỉ hạ được mức tự chủ, không nâng được.** Trần trong sổ là kết quả của
   một lần cân nhắc có ghi lý do; một lệnh nâng lúc đang gấp thì không. Muốn
   nâng thì sửa sổ `src/crm/so-tro-ly.js` kèm lý do, rồi chạy lại `npm run verify`.
2. **Không cấp mức 4 cho ai.** Mọi việc chạm ra ngoài hệ thống — thư, tin nhắn
   tới một gia đình — đều dừng ở hàng chờ duyệt.
3. **Không tắt được nhật ký kiểm toán.**
4. **Không xoá được một việc đã quyết.**

### Siết hoặc nới ngưỡng cần người duyệt

```bash
curl -X POST http://localhost:9999/crm/quan-tri/nguong \
  -H "Authorization: Bearer <token admin>" \
  -H 'content-type: application/json' -d '{"canNguoi":0.25}'
```

Nới **không quá 0,60**. Trên mức ấy thì gần như mọi việc tự chạy và hàng chờ
duyệt trở thành một cái hộp rỗng — đúng chỗ mà hệ thống duyệt tự động hay chết
mà không ai để ý. Hàm từ chối lệnh nới quá tay.

### Đọc bảng điểm cho đúng

`/crm/kpi` in ra **độ phủ** trước khi in điểm:

```
62 chỉ số đã khai · 37 đã nối nguồn đo · 25 CHƯA nối nguồn · độ phủ 59,7%
```

Điểm chỉ tính trên 37 chỉ số có nguồn. Một trợ lý đạt 90 điểm trên 3 chỉ số
chấm được thì con số 90 ấy nói ít hơn nhiều so với 90 điểm trên 8 chỉ số — cột
"Chấm được" trên màn hình nói rõ mẫu số. Trợ lý chưa đủ **10 lượt** thì không
chấm, vì mọi tỷ lệ dưới ngưỡng ấy đều là nhiễu.

### Khi công cụ nói "chưa có"

Đó là **câu trả lời đúng**, không phải lỗi. Kho CRM rỗng thì công cụ nói rỗng
và chỉ ra cổng để nạp dữ liệu thật vào. Nếu muốn một công cụ trả về số, hãy nạp
số thật qua `POST /crm/gia-dinh`, `POST /crm/hoc-phi`, `POST /crm/chien-dich` —
**đừng sửa công cụ để nó trả số mẫu**. Mục kiểm `Công cụ nói "chưa có" khi kho
rỗng` sẽ bắt được, và nó bắt được vì lý do chính đáng.

### Biến môi trường

| Biến | Mặc định | Ý nghĩa |
|---|---|---|
| `CRM_LOI_DINH_TUYEN` | `ghep` | `tu-khoa` tất định · `mo-hinh` · `ghep` |
| `CRM_LICH_VIEC` | `1` | `0` tắt năm việc chạy nền |

Tắt lịch việc trên máy chạy thật nghĩa là việc chờ duyệt **không bao giờ hết
hạn** và kỳ học phí quá hạn **không bao giờ được đánh dấu**. Chỉ tắt khi chạy
thử.

### Chỉ số Prometheus

`/metrics` có thêm mười dòng bắt đầu bằng `crm_`. Ba dòng đáng gắn cảnh báo
nhất:

```
crm_duyet_het_han_total   > 0   → có người đã chờ mà không ai tới
crm_toan_ban_dung         = 1   → cả ban đang bị dừng
crm_tuong_chan_total      tăng  → trợ lý đang định nói ra phân loại nội bộ
```

## 9. Những điều hệ thống **không** làm

Ghi ra đây để không ai kỳ vọng nhầm:

- **Không kết tội học viên.** Phần phân tích chỉ nêu *dấu hiệu* kèm số liệu.
  Quyết định là của giáo viên.
- **Không tự phát hành bài từ 4★ trở lên.** Máy chỉ xác nhận được đáp số;
  lập luận và mức độ phù hợp phải người thật duyệt.
- **Không chấm bài luận và bài nói tiếng Anh thành band.** Chấm band cần người
  chấm theo thang mô tả. Hệ thống dạy và kiểm bước TRƯỚC KHI viết và nói —
  đọc ra dạng đề, dựng câu chủ đề đúng dạng, chạm đủ bốn gạch đầu dòng của thẻ
  nói — và chỉ nhận đúng phần việc đó.
- **Không sinh đoạn đọc hiểu bằng máy.** Ghép câu bằng máy ra một đoạn không có
  ý chính nào để mà hỏi. Đoạn đọc hiểu, đề luận và thẻ nói đều viết tay, nằm ở
  `src/lang/en-corpus.js`; máy chỉ kiểm chất lượng đề trên văn bản có sẵn.
- **Không cho lên tầng khi thiếu tiêu chí**, kể cả khi điểm thi 100%.
  Người có thẩm quyền quyết khác máy được, nhưng để lại dấu vết.
- **Không kết luận từ mẫu nhỏ.** Dưới 10 lượt làm bài thì mọi tỉ lệ đều
  kèm cảnh báo chưa đủ căn cứ.
- **Không gọi ra mạng ngoài từ giao diện.** Bản cài trên máy chạy được khi
  mất mạng.
- **Không nhân bản giọng người thật khi chưa có văn bản đồng ý.** Không có mã
  hồ sơ thì hệ thống từ chối, không có đường vòng.
- **Không phát ra tệp âm thanh chưa đóng dấu AI.** Nhà cung cấp nào trả về
  định dạng nén không nhúng được dấu thì hệ thống ghi rõ vào nhật ký là chưa
  đóng dấu, chứ không im lặng cho qua.
- **Không dùng giọng tiếng Việt sẵn của trình duyệt.** Mỗi máy một giọng khác
  nhau, có máy không có, và không cách nào đánh dấu nội dung AI.
- **Không đổi giọng một bản hát khi chưa đủ hai phiếu đồng ý** — của người hát
  gốc và của chủ giọng đích. Không có mô hình thì nói thẳng là chưa làm được,
  không tự chế bản thay thế.
- **Không nói là sinh được ảnh bằng mã nội bộ.** Hậu kỳ và khung dựng thì
  làm được trọn vẹn; sinh ảnh mới bắt buộc phải có cổng bên ngoài.
- **Không giả vờ cứu được ảnh mờ nặng.** Mặt nạ làm sắc có ngưỡng, nên khi
  chênh lệch cục bộ đã tụt dưới ngưỡng thì nó không khuếch đại nhiễu rồi
  gọi đó là "nét".
- **Không dựng video khi soát chất lượng chưa đạt**, kể cả khi người dùng đã
  xác nhận chi tiêu. Lỗi chặn là lỗi chặn.
- **Không tự gọi API tốn tiền.** Luôn trả dự toán trước và chờ xác nhận tay.
- **Không nói là dựng được video bằng mã nội bộ.** Bản nháp thì dựng được
  hoàn chỉnh; video thật bắt buộc phải có khoá nhà cung cấp.
- **Không hát ép bài vượt quãng giọng.** Dịch được theo quãng tám thì dịch;
  bài rộng hơn quãng của người hát thì báo và dừng.
- **Không đổi dấu "/" và "-" trong tiếng Việt thành ký hiệu toán.** "Ngày 20/11",
  "60 km/h", "đen-ta", "Bài 3-4" giữ nguyên; chỉ đổi khi quanh đó đúng là đang
  viết công thức.
- **Không nuốt ký tự khi Unicode không có bản mũ tương ứng.** `x^q` giữ nguyên
  và bị báo lỗi, chứ không lặng lẽ thành `xq` — đổi nghĩa công thức là lỗi nặng
  hơn nhiều so với viết chưa đẹp.
- **Không bày ra cách giải không áp dụng được.** Nhẩm nghiệm chỉ hiện khi
  a + b + c = 0 hoặc a − b + c = 0; hằng đẳng thức chỉ hiện khi ∆ = 0; Vi-ét đảo
  chỉ hiện khi nghiệm nguyên.
- **Không cho bài sai đáp số đi qua vì trình bày đẹp.** Cấp 4 thay nghiệm ngược
  vào phương trình; sai là trượt, dù ký hiệu và văn phong đều đạt.
- **Không dựng lộ trình 90 ngày cho học viên chưa có dữ liệu học.** Không có dữ
  liệu thì chỉ có bản kế hoạch chung chung, và thứ đó không giúp ai tiến bộ.
- **Không tự sửa chính tả từ vựng.** Máy không biết người viết định nói "bàn"
  hay "bàng"; nó chỉ sửa những lỗi trình bày không đổi nghĩa.
- **Không coi kiểu đặt dấu thanh nào là sai.** "hoà" và "hòa" đều đọc đúng; hệ
  thống chỉ đòi một văn bản dùng nhất quán một kiểu.
- **Không trình bày quy ước biên tập như thể là số đo.** Bảng ngưỡng độ đọc
  theo lớp được ghi rõ là quy ước, kèm số đo thật của kho để đối chiếu.
- **Không hứa đo được kỹ năng ngôn ngữ của học viên.** Hệ thống đo độ đọc của
  HỌC LIỆU; đo năng lực nghe nói đọc viết của người học cần bài kiểm tra riêng.
- **Không tải bất cứ thứ gì từ tên miền khác trên trang công khai.** Không
  phông chữ, không thư viện, không mã theo dõi.
- **Không biết doanh thu, chi phí nhân sự và chi phí hạ tầng.** Ba khoản ấy
  chưa nối được nguồn, nên sổ cái **từ chối ghi** vào chúng. Một con số bịa
  trong bảng tài chính nguy hiểm hơn một ô trống.
- **Không xác minh được tuổi thật của học viên.** Hệ thống phân biệt rõ lời
  khai với bằng chứng, và chỉ mở tính năng chạm người lạ khi có xác nhận của
  nhà trường hoặc tổ chức. Ngưỡng tuổi là CHÍNH SÁCH, chưa phải kết luận pháp lý.
- **Không tự chứng nhận tuân thủ pháp luật.** Hệ thống chỉ liệt kê nghĩa vụ nó
  tự đặt ra và chỗ trong mã thực hiện chúng; nghĩa vụ chưa có người ký nhận thì
  trạng thái là *chưa xác nhận*, không phải *đạt*. Các số hiệu văn bản pháp
  luật trong mã là do người dựng khai, KHÔNG do hệ thống tra cứu.
- **Không nói được runway, biên lợi nhuận và rủi ro tập trung doanh thu.** Ba
  thứ ấy đều cần dữ liệu thanh toán mà hệ thống chưa nối.
- **Không biết một cuộc rà soát chiến lược đã diễn ra hay chưa.** Chưa có kho
  lưu biên bản; hệ thống chỉ nói được lịch phải làm.
- **Không đo được tỉ lệ giữ chân, CLV, tỉ lệ giới thiệu, mức hài lòng và chi
  phí phục vụ một khách.** Năm nhóm ấy chưa nối nguồn. Chuỗi "dịch vụ → trung
  thành → lợi nhuận" hiện mới đo được đoạn đầu.
- **Không cho mức WOW khi mức NỀN của chính điểm chạm ấy chưa làm xong.** Một
  bất ngờ dễ thương đặt trên một lời hứa chưa giữ là trò đánh lạc hướng.
- **Không có mức WOW cho nhóm cảm xúc và an toàn của trẻ**, kể cả khi nền đã
  xong. Sáng tạo ở đó là đem một đứa trẻ ra thử nghiệm.
- **Không canh hộ 11/40 tình huống.** Những tình huống chỉ tồn tại khi có người
  báo thì không ai trông lúc 11 giờ đêm — hệ thống in rõ danh sách ấy.
- **Không tô sẵn quả trên hình cây khi chưa có dữ liệu học.** Chưa đủ căn cứ
  thì cả năm mươi quả đều mờ như nhau. Tô vài quả cho hình đỡ trống là nói dối
  bằng màu, và người xem tin vào cái mình nhìn thấy trước khi đọc chữ.
- **Không báo "đã đạt" theo mốc mềm.** Trong lưới 50 ô chỉ 15 ô (cấp 3, 6, 10
  của mỗi tầng) có mốc cứng đo bằng máy; 35 ô còn lại tự mang câu "KHÔNG dùng
  để báo cáo đã đạt".
- **Không gộp sáu mặt khảo sát thành một chỉ số.** Biết một em "7,2 điểm"
  không nói được nên làm gì. Và miền nặng nhất quyết định, không phải trung
  bình: một em ngủ 3–4 tiếng mỗi đêm thì điểm tâm lý phải tụt và gọi tên miền
  giấc ngủ, chứ không được năm miền còn lại kéo về mức "ổn".
- **Không cho tổ thanh tra tự vá lỗi nó tìm ra.** Nó ghi biên bản và báo động.
  Một bộ máy vừa phát hiện vừa tự sửa là một bộ máy có thể tự sửa cả phát hiện
  của mình.
- **Không nhận biểu mẫu khu riêng khi thiếu vé**, kể cả khi người dùng đã đăng
  nhập đúng và yêu cầu đến từ cùng gốc. Vé thiếu và vé sai bị chặn như nhau.
- **Không mở CORS bằng `*`.** Phiên đăng nhập đi bằng cookie; mở gốc là mở luôn
  phiên của mọi người đang đăng nhập. `CORS_ORIGINS` trống là đóng hẳn.
