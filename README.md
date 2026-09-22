# GITAmath Nexus V20

Hệ điều hành vận hành **500 Agent** cho GITAmath: có bảng phân công, vai trò, quyền hạn,
KPI, công suất, hiến pháp kiểm soát, bảo mật Super Admin 15 tầng, khôi phục theo phiên bản,
chống nhiễu cho toàn đội, chương trình học phân cấp, và hai lớp ứng dụng — **web app** và
**bản cài đặt Windows 64-bit**.

Chạy được ngay, không cần hạ tầng ngoài, **không một phụ thuộc runtime nào**.

```bash
npm start            # bật máy chủ + ba cổng giao diện tại http://localhost:9999/ui/
npm test             # 595 test
npm run verify       # 104 mục kiểm định, in ra số liệu thật
npm run smoke        # dựng máy chủ thật, gọi 232 lượt API thật
npm run load         # kiểm thử tải 5 kịch bản, tìm điểm gãy
npm run roster       # in bảng phân công 500 agent (thêm --csv để xuất tệp)
```

**Ba cổng vào** (`/ui/` tự chuyển về trang chọn vai):

| Cổng | Dành cho | Làm được gì |
|---|---|---|
| `/ui/hoc-vien.html` | học viên | làm bài, xem lộ trình, theo dõi tiến bộ |
| `/ui/giao-vien.html` | giáo viên | lớp, từng em, danh sách cần chú ý, duyệt học liệu, báo cáo |
| `/ui/index.html` | Super Admin | 500 agent, tổng động viên, bảo mật, khôi phục, hiến pháp |

Vận hành hằng ngày và xử lý sự cố: xem **[RUNBOOK.md](RUNBOOK.md)**.

---

## 1. Lực lượng 500 Agent

Hai chiều tổ chức, khớp chính xác theo cả hàng lẫn cột — kiểm tra tự động ở mỗi lần khởi động.

| Khối | PLANNER | WORKER | CRITIC | COACH | RESEARCHER | GUARDIAN | Tổng |
|---|---:|---:|---:|---:|---:|---:|---:|
| MATH · Toán học | 2 | 64 | 5 | 3 | 3 | 3 | **80** |
| VIETNAMESE · Ngôn ngữ Việt | 2 | 80 | 6 | 4 | 4 | 4 | **100** |
| TUTOR · Gia sư | 2 | 80 | 6 | 4 | 4 | 4 | **100** |
| RESEARCH · Nghiên cứu | 2 | 64 | 5 | 3 | 3 | 3 | **80** |
| CODE · Kỹ thuật | 1 | 64 | 5 | 3 | 4 | 3 | **80** |
| BUSINESS · Kinh doanh | 1 | 48 | 3 | 3 | 2 | 3 | **60** |
| **Tổng** | **10** | **400** | **30** | **20** | **20** | **20** | **500** |

* **25 tổ, mỗi tổ đúng 20 người**, mỗi tổ luôn có ít nhất một chỉ huy (PLANNER hoặc CRITIC).
* Mỗi agent mang đủ: chức năng · quyền hạn · KPI mục tiêu · trần công suất
  (việc song song, token/phút, task/giờ) · chuyên môn · hạng mô hình.
* Tổng công suất danh định: **1.800 slot đồng thời · 18.400 task/giờ**
  (trần toàn cục trong cấu hình siết lại để không đốt quota).
* **Đội hình tất định**: dựng lại bao nhiêu lần cũng ra đúng một kết quả — không có
  `Math.random()` ở bất kỳ đâu trong `src/`. Thăm dò của bandit dùng mulberry32 có seed
  (`ROUTER_SEED`), nên lặp lại được.

Xem đầy đủ: `npm run roster` · `npm run roster:csv > phan-cong-500-agent.csv`

## 2. Một lệnh — cả 500 agent vào việc

`POST /mission/execute` chạy 6 giai đoạn: **plan → shard → execute → review → guard → coach**.

* **Không chồng chéo**: sổ giành mảnh (shard ledger) cấp mỗi mảnh việc cho đúng một agent;
  mọi lần giành lại đều bị từ chối và được đếm (`overlapsPrevented`).
* **Đúng nghiệp vụ**: mỗi agent chỉ nhận loại việc thuộc chuyên môn khối mình —
  giao sai là hiến pháp chặn tại điều A13.
* **Trải đều 6 khối**: nhóm thợ được rải vòng tròn xen kẽ giữa các khối, không dồn một khối.
* **Chất vấn chéo** (`POST /debate/run`): đề xuất → chất vấn vòng tròn (không ai tự chất vấn mình)
  → phản biện từng điểm (CHẤP NHẬN / BÁC BỎ) → sửa lại → lặp tới khi hội tụ → PLANNER tổng hợp
  → CRITIC chấm rubric 5★.
* **Bắt buộc hiểu nhau** (`POST /handoff/run`): OFFER → ACK (bên nhận **phải diễn đạt lại**
  yêu cầu; mức hiểu < 0.5 thì bị trả về hỏi lại) → CLARIFY → DELIVER → ACCEPT do một
  thanh tra độc lập nghiệm thu theo từng tiêu chí.

## 3. Hiến pháp kiểm soát

50 điều, 5 chương, chạy như **middleware thật** trước mọi tác vụ — không phải văn bản trang trí.
Chế tài ba cấp: `WARN` → `SUSPEND` → `BLOCK`.

```
vượt quyền        → BLOCK   (A05)
ngoài chuyên môn  → SUSPEND (A13)
task trùng lặp    → SUSPEND (A17)
lỗi liên tiếp ≥3  → SUSPEND (A20)
chưa thanh tra    → BLOCK   (A06)  — nội dung không được phát hành
```

Kèm HEART 7 axiom, guardrails (PII có kiểm Luhn cho số thẻ), safe-meta-prompt chống
prompt injection, và **nhật ký bất biến móc xích băm** — sửa một bản ghi là phát hiện ngay.

## 4. Bảo mật Super Admin — 15 tầng

`L01 … L15`: mật khẩu scrypt (N=2¹⁵) → TOTP RFC 6238 → phiên có hạn → ràng buộc thiết bị/IP
→ kiểm quyền → hiến pháp → xác thực nâng cấp (step-up) → hai người duyệt (dual control)
→ ký lệnh HMAC → chặn brute-force theo thang khoá tăng dần → nhật ký bất biến → …

* **Mật khẩu**: ≥14 ký tự, đủ 4 nhóm ký tự, entropy ≥80 bit; từ khoá liên quan sản phẩm bị từ chối.
* **8 lệnh huỷ diệt** (`system.halt`, `snapshot.restore`, `items.purge`, …) bắt buộc hai người duyệt.
* **Khoá tăng dần** khi sai mật khẩu: 1 → 5 → 15 → 30 → 60 phút.
* **Đăng nhập sai trả HTTP 401**, không phải 200 — để tường lửa và bộ chặn brute-force
  đọc đúng tín hiệu.

### Bốn đường lấy lại quyền khi mất mật khẩu

1. **Mã khôi phục** dùng một lần (phát khi khởi tạo, chỉ hiện một lần).
2. **Shamir 3/5**: năm người giữ mảnh, cần đúng ba mảnh để ghép lại khoá gốc — hai mảnh
   không lộ bất cứ gì. (Triển khai trên GF(2⁸), có test.)
3. **Break-glass**: khoá niêm phong dùng đúng một lần, dùng xong là vô hiệu vĩnh viễn.
4. **Khôi phục toàn hệ** từ ảnh chụp theo phiên bản/thời điểm.

### Bảo lưu theo phiên bản và thời điểm

Ảnh chụp đánh số `v20.00001`, `v20.00002`, … **móc xích bằng băm** (`prev` → `hash`),
ghi xuống đĩa nên sống sót qua khởi động lại, giữ theo chính sách **ông–cha–con**
(12 ảnh trong 1 giờ gần nhất · 24 ảnh trong 24 giờ · 30 ảnh trong 30 ngày, ảnh thủ công
giữ vĩnh viễn). Khôi phục về **bất kỳ phiên bản hoặc bất kỳ mốc thời gian** nào còn giữ.

## 4b. Cổng quyền — một lớp xác thực cho toàn bộ API

Một bản rà soát toàn hệ thống tìm ra lỗ thủng nghiêm trọng nhất của dự án này:
trong **300 cổng API, chỉ 5 cổng tự kiểm quyền** — đúng 5 cổng của phần Cây
Tiền, vá riêng lúc làm phần đó. 295 cổng còn lại mở cho bất kỳ ai gọi được tới
máy chủ, trong đó:

```
POST /accounts   →  tạo được tài khoản ADMIN, KHÔNG cần đăng nhập
GET  /bank/items →  đề bài KÈM ĐÁP ÁN của cả kho học liệu
POST /chat       →  gọi mô hình ngôn ngữ, tức là tiêu tiền
```

Nghĩa là bức tường canh Cây Tiền **vô giá trị**: ai cũng tự đúc được chìa khoá
rồi đi cửa chính. Vá từng cổng là cách đã hỏng một lần, nên lần này luật nằm ở
**một chỗ duy nhất** (`src/api/cong-quyen.js`) và **mọi** cổng đều đi qua nó.

**Bốn mức, xếp thứ bậc:** `cong-khai` → `hoc-vien` → `giao-vien` → `quan-tri`.

**Nguyên tắc: mặc định là ĐÓNG.** Nhóm cổng nào chưa khai mức thì rơi vào
`quan-tri` — chặt nhất. Muốn mở thì phải **khai tên tường minh**, và mỗi dòng
trong danh sách công khai phải trả lời được câu hỏi *"người chưa đăng nhập cần
cổng này để làm gì?"*.

Bản đầu của chính bản vá này vẫn còn sai: nó cho cả **nhóm** CORE, CURRICULUM,
SUPERADMIN mặc định công khai, và **mục kiểm tự viết đã bắt được hậu quả** —
`POST /chat` và `GET /bank/items` lọt ra ngoài. Mở theo cả nhóm là mở cho cả
những cổng chưa ai đọc lại.

| | trước bản vá | sau bản vá |
|---|---|---|
| Cổng công khai | 295 / 300 (**98%**) | 9 / 300 (**3%**) |
| Tạo được ADMIN ẩn danh | **có** | không |

**Cửa khởi tạo.** Hệ thống mới cài chưa có tài khoản quản trị nào thì phải tạo
được tài khoản đầu tiên, nếu không nó tự khoá mình ngoài cửa. Cửa ấy hẹp đúng
một khe: chỉ `POST /accounts`, chỉ khi số tài khoản ADMIN bằng **0**, chỉ cho
vai ADMIN, và mỗi lần mở đều **ghi vào nhật ký kiểm toán**. Kho tài khoản hỏng
không đếm được thì cửa **đóng**, không mở vì "không chắc".

---

## 4c. Ghi xuống đĩa — thứ tự bốn bước, không có bước thứ năm

Một bản rà soát tìm ra khe mất dữ liệu trong bộ nén nhật ký: hệ thống đổi tên
tệp ảnh chụp rồi **cắt trắng nhật ký ngay**, không chờ thư mục được đồng bộ
xuống đĩa. Máy mất điện đúng giữa hai việc ấy thì ảnh chụp mới chưa nằm trên
đĩa mà nhật ký cũ đã không còn — mất trắng toàn bộ thay đổi từ lần nén trước.

Thứ tự bây giờ là luật, và có mục kiểm **đọc thẳng mã nguồn** để chắc nó không
bị đảo lại:

```
1. ghi tệp tạm            fs.writeFileSync(fd, body)
2. ép tệp tạm xuống đĩa   fs.fsyncSync(fd)
3. đổi tên đè lên ảnh cũ  fs.renameSync(tmp, snapFile)
4. ép THƯ MỤC xuống đĩa   fsync(dirfd)      ← bước hay bị quên nhất
5. bây giờ mới cắt nhật ký
```

Bước 4 là bước người ta quên. `rename` trên POSIX là thao tác nguyên tử, nhưng
nguyên tử **không có nghĩa là đã nằm trên đĩa**: mục lục thư mục vẫn còn trong
bộ đệm. Không `fsync` thư mục thì sau khi mất điện, tên tệp có thể trỏ vào hư
không.

## 4d. CORS và CSRF — hai lỗ vào mà không cần mật khẩu

**CORS mặc định ĐÓNG.** `CORS_ORIGINS` để trống là không gốc ngoài nào gọi được
API từ trình duyệt. Cấm đặt `*`: phiên đăng nhập đi bằng cookie, mở gốc là mở
luôn phiên của mọi người đang đăng nhập.

**CSRF chặn bằng HAI lớp**, và hai lớp ấy hỏng theo hai cách khác nhau — đó
mới là lý do có hai lớp thay vì một lớp làm kỹ.

**Lớp 1 — dấu vết nguồn.** `Sec-Fetch-Site` (trình duyệt hiện đại tự gắn,
trang web không sửa được) và đối chiếu `Origin` với `Host` cho trình duyệt cũ.
Yêu cầu **không mang dấu vết nguồn nào** cũng bị chặn — đó không phải biểu mẫu
của trình duyệt. Lớp này không cần đụng vào biểu mẫu, nhưng nó *tin* vào việc
trình duyệt có gửi đúng tiêu đề.

**Lớp 2 — vé trong biểu mẫu** (`src/web/chong-gia-mao.js`). Mỗi biểu mẫu mang
một ô ẩn mà chỉ phiên đang đăng nhập mới tính ra được: `HMAC-SHA256` của chính
token phiên. Trang lạ không đọc được cookie của gốc này nên không tính được vé,
dù nó có lách qua được lớp 1. Ba hệ quả có chủ ý:

- **Không lưu thêm gì.** Không có bảng vé để rò, để hết hạn lệch, để đầy.
- **Vé chết đúng lúc phiên chết.** Đăng xuất là mọi vé đã phát vô hiệu ngay.
- **Vé không lần ngược ra token phiên**, nên nhúng vé vào HTML không phải là
  để lộ phiên. So vé bằng `timingSafeEqual`, không bằng `===`.

| Yêu cầu POST tới `/nha/:phong/nop` | Trả về |
|---|---|
| cùng gốc, đúng phiên, **đúng vé** | 303 → đã nộp |
| cùng gốc, đúng phiên, **thiếu vé** | **403** |
| cùng gốc, đúng phiên, **vé bịa hoặc vé rỗng** | **403** |
| cùng gốc, đúng phiên, **vé của phiên khác** | **403** |
| `Sec-Fetch-Site: cross-site` (kể cả khi vé đúng) | **403** |
| không có `Sec-Fetch-Site`, không có `Origin` | **403** |
| `Sec-Fetch-Site: same-origin`, **chưa đăng nhập** | 303 → trang đăng nhập |

Bảy dòng này là phép kiểm chạy thật trong `npm run smoke`, không phải lời hứa.
Chưa đăng nhập thì **không đòi vé**: không có phiên nghĩa là không có gì để giả
mạo hộ, và yêu cầu ấy cũng chỉ được đẩy về trang đăng nhập chứ không ghi gì.

Thêm một biểu mẫu mới vào khu riêng mà **quên ô vé** thì biểu mẫu ấy vẫn chạy
được (lớp 1 vẫn cho qua), nên không ai phát hiện ra cho tới khi có người lợi
dụng. Mục kiểm `VÉ BIỂU MẪU` quét mã nguồn và bắt đúng chỗ ấy — đã thử gỡ một ô
vé và mục kiểm gọi ra `nha-trang.js:437`.

## 4e. Tổ thanh tra — 10 trợ lý AI, 10 lượt mỗi ngày

Sửa lỗi sau khi có sự cố là việc đắt nhất trong vận hành. Tổ thanh tra soi để
lỗi bị bắt **trước** khi thành sự cố.

- **10 trợ lý AI**, mỗi trợ lý một mặt: bền vững dữ liệu · cổng quyền · bức
  tường Cây Tiền · dòng tiền · luật đo · bằng chứng học liệu · ngôn ngữ học
  liệu · sức khoẻ đội hình · riêng tư dữ liệu trẻ · ký hiệu toán.
- **10 lượt mỗi ngày**, 06:00 → 23:00. Năm phép soi **mức chặn** chạy ở *mọi*
  lượt; năm phép còn lại luân phiên hai phép mỗi lượt.
- Phép soi **thiếu, ném lỗi, hoặc trả về sai định dạng** đều tính là KHÔNG ĐẠT.
  Một phép soi im lặng vì hỏng trông giống hệt một phép soi im lặng vì sạch.
- Biên bản ngày tính **thiếu lượt** là không đạt: lượt không chạy không phải
  lượt sạch.

Tổ thanh tra **ghi biên bản và báo động, không tự vá**. Sửa là việc của người
có thẩm quyền — một bộ máy vừa phát hiện vừa tự sửa là một bộ máy có thể tự
sửa cả phát hiện của mình.

```bash
node bin/gita.js to-thanh-tra            # một lượt
node bin/gita.js to-thanh-tra --ca-ngay  # cả 10 lượt, kèm biên bản ngày
```

## 4f. Sổ tay quy trình — bốn câu, và câu thứ tư quan trọng nhất

8 kịch bản trực sự cố. Mỗi kịch bản trả lời đúng bốn câu, theo đúng thứ tự:

**DẤU HIỆU → LÀM NGAY → AI QUYẾT → CẤM LÀM**

Mục **CẤM LÀM** là mục hay bị bỏ nhất và quan trọng nhất. Phần lớn sự cố nhỏ
thành sự cố lớn vì người trực, lúc cuống, làm một việc tưởng là cứu chữa:
khởi động lại máy chủ "xem thử" và gộp đè lên nhật ký còn tốt; nới bảng từ cấm
cho mục kiểm xanh lại; coi một lượt thanh tra không chạy là một lượt sạch.

| Mã | Kịch bản | Mức | Trong bao lâu |
|---|---|---|---|
| KB01 | Nghi ngờ mất dữ liệu | Đỏ | 15 phút |
| KB02 | Rò rỉ dữ liệu ra ngoài | Đỏ | 15 phút |
| KB03 | Chi phí mô hình tăng đột biến | Cam | trong ca trực |
| KB04 | Sàng lọc tâm lý báo cần người lớn ngay | Đỏ | 15 phút |
| KB05 | Học liệu sai đáp số lọt ra ngoài | Đỏ | 15 phút |
| KB06 | Trợ lý AI bị treo | Cam | trong ca trực |
| KB07 | Gia đình yêu cầu xoá dữ liệu | Vàng | 24 giờ |
| KB08 | Lượt thanh tra không chạy | Vàng | 24 giờ |

Sổ tay nối thẳng với tổ thanh tra: `gita quy-trinh --thanh-tra=TT09` ra đúng
những kịch bản mà thanh tra viên TT09 có thể phải mở. Mọi đường dẫn tệp khai
trong sổ tay đều có **mục kiểm đòi nó tồn tại thật** — tài liệu trỏ vào tệp
không còn là tài liệu tệ hơn không có tài liệu.

## 4g. Dòng tiền — chỗ mù được nêu tên trước con số

Sổ cái dòng tiền in **khoản chưa nối nguồn trước, số liệu sau**. Thứ tự ấy có
chủ ý: người đọc phải biết bảng này chưa thấy gì trước khi tin những gì nó
thấy.

- Ba khoản đánh dấu `tinCay: 'chua-co'` — **doanh thu, nhân sự, hạ tầng**. Sổ
  cái **từ chối ghi** vào ba khoản ấy: chưa nối được nguồn thì không có số, và
  không được bịa số.
- Ngân sách gọi mô hình **chặn cứng**, không chỉ cảnh báo: một khoản chi vượt
  trần ngày hoặc trần tháng bị từ chối tại chỗ. Cái giá của việc chặn nhầm rẻ
  hơn cái giá của một đêm gọi mô hình chạy loạn.
- `NGAN_SACH_NGAY_USD` và `NGAN_SACH_THANG_USD` đặt trong `.env`.

## 4h. Lưới 50 ô — 5 tầng × 10 cấp, mốc cứng tách khỏi mốc mềm

Cây giá trị năm tầng, mỗi tầng mười cấp: **50 ô**. Cấp 10 của mỗi tầng nối
thẳng vào cấp 1 của tầng sau, nên đường đi liền mạch chứ không đứt đoạn giữa
các tầng.

Điều quan trọng không phải là 50 ô, mà là **mỗi ô tự khai bằng chứng của nó
thuộc loại nào**:

- **15 mốc cứng** (cấp 3, 6, 10 của mỗi tầng) — đo bằng máy, đọc thẳng từ dữ
  liệu học thật. Dùng để báo cáo đã đạt.
- **35 mốc mềm** — tự khai kèm nguyên văn *"KHÔNG dùng để báo cáo đã đạt"*, và
  có mục kiểm bắt ô nào quên câu ấy.

Một lưới mà mọi ô đều trông như bằng chứng là một lưới không có bằng chứng nào.

**Và lưới ấy vẽ thành một cái cây thật** (`src/web/cay-hinh.js`), SVG nội tuyến,
không JavaScript và không tải gì từ bên ngoài:

| Trên hình | Là gì trong hệ thống |
|---|---|
| thân | con đường chung, đi từ dưới lên |
| 4 nhánh lớn | tầng 1–4, nhánh dưới dài hơn nhánh trên |
| tán | tầng 5 — vẽ thành tán vì đó là đích của cả cây |
| 10 nhánh con mỗi tầng | 10 cấp |
| lá | **nhiệm vụ** học viên làm ở cấp ấy |
| quả **chín** (15) | **thứ gia đình nhận được**, ở cấp có bằng chứng đo bằng máy |
| quả **non** (35) | thứ nhận được ở cấp mốc mềm — chưa dùng để báo đã đạt |

Năm mươi chiếc lá và năm mươi quả **viết tay từng cặp** theo đúng việc của từng
tầng, không lấy một khuôn chung rồi thay tên tầng — mục kiểm đòi cả 50 lá và
50 quả phải khác nhau, và đòi không quả nào hứa bằng điểm số.

Hình này không phải hình minh hoạ: mục kiểm `HÌNH CÂY` đối chiếu số quả với số
ô, số quả chín với số mốc cứng, bắt hai lần vẽ phải ra đúng một hình, bắt mọi
nét nằm trong khung (bản đầu đã cắt cụt chữ đầu của một nhãn tầng), và cho hình
đi qua đúng bức tường Cây Tiền như mọi trang khác. **Chưa đủ căn cứ thì không
quả nào được tô như đã hái** — tô sẵn vài quả cho hình đỡ trống là cách nhanh
nhất để mất lòng tin vào cả năm mươi quả còn lại.

## 4i. Phân tích khảo sát — sáu mặt, không gộp thành một chỉ số

Bộ phân tích đọc kết quả sáu công cụ khảo sát và trả về **sáu mặt riêng biệt**,
cùng chuỗi vấn đề và nguyện vọng ghép vào từng mặt. Hai luật cứng:

1. **Không có chỉ số tổng hợp.** Gộp sáu mặt thành một con số là làm mất đúng
   cái thông tin dùng được: biết một em "7,2 điểm" không nói được nên làm gì.
2. **Miền nặng nhất quyết định, không phải trung bình.** Một em ngủ 3–4 tiếng
   mỗi đêm mà năm miền còn lại ổn thì điểm tâm lý phải **tụt xuống và gọi tên
   miền giấc ngủ**, không phải được năm miền kia kéo về mức "ổn". Có mục kiểm
   dựng đúng tình huống ấy và bắt hệ thống phải trả về dưới 40 điểm.

---

## 4j. Hệ thống trải nghiệm khách hàng — 10 tầng, lưới 10.000 điểm chạm

Điều đáng giữ nhất của quản trị dịch vụ không phải "hãy phục vụ tận tâm", mà
là: **dịch vụ tốt là một HỆ THỐNG, không phải một thái độ.** Thái độ không nhân
bản được và không bàn giao được; hệ thống thì có.

Mười tầng, mỗi tầng trỏ vào một tệp mã **đang chạy** — có mục kiểm đòi mọi
đường dẫn ấy tồn tại thật, vì một khung quản trị mà tầng nào cũng "đã có" là
khung vẽ trên giấy:

| # | Tầng | Câu hỏi quản trị | Trong hệ thống |
|---|---|---|---|
| 1 | Lời hứa dịch vụ | Ta muốn gia đình nhớ điều gì? | `src/nha/cay-gia-tri.js` |
| 2 | Hiểu khách hàng | Ta biết gì, bằng căn cứ nào? | `src/khao-sat/phan-tich.js` |
| 3 | Hành trình | Khách đang đứng ở đâu? | `src/matran/cay-50.js` |
| 4 | Tình huống | Chuyện gì có thể xảy ra, ai xử? | `src/trai-nghiem/tinh-huong.js` |
| 5 | Kênh | Nói qua đường nào cho đúng việc? | `src/trai-nghiem/kenh.js` |
| 6 | Điểm chạm | Vị trí này, tình huống này, kênh này thì làm gì? | `src/trai-nghiem/luoi-cham.js` |
| 7 | Trao quyền | Người trực được quyết gì ngay? | `src/trai-nghiem/trao-quyen.js` |
| 8 | Phục hồi | Hỏng việc rồi thì gỡ thế nào? | `src/trai-nghiem/phuc-hoi.js` |
| 9 | Đo lường | Dịch vụ có tốt lên không? | `src/trai-nghiem/do-luong.js` |
| 10 | Kiểm soát | Ai canh cho hệ thống không mục ruỗng? | `src/thanh-tra/to-thanh-tra.js` |

### Nói thẳng về con số 10.000 trước khi nói gì khác

Lưới **10.000 ô = 50 vị trí × 40 tình huống × 5 kênh**. Nhưng đó **không phải
10.000 kịch bản viết tay**. Số mục viết tay là **95** (50 ô của lưới cây + 40
tình huống + 5 kênh). Nói ngược lại là nói dối, và người đọc phát hiện ra ở ô
thứ mười — nên chính mô-đun tự in câu đính chính ấy ra, và có mục kiểm đòi nó
phải có mặt.

Mười nghìn ô vẫn có thật, vì cùng một tình huống "vắng bảy ngày" xử ở cấp T1C2
khác hẳn xử ở T4C6: thứ em đang dở khác nhau, thứ gia đình đang chờ khác nhau,
câu để mở lời khác nhau. Mục kiểm soi **cả 5.600 ô dùng được** và bắt không ô
nào có hai dòng kịch bản trùng nhau.

| | |
|---|---|
| dùng được | **5.600** ô |
| bị chặn | **4.400** ô — kênh không hợp mức khẩn |
| có mức wow | 4.900 ô |

**Ô bị chặn vẫn là một câu trả lời.** Báo kết quả sàng lọc tâm lý qua báo cáo
định kỳ là sai dù nội dung đúng tới đâu, nên ô ấy trả về lý do từ chối **kèm
kênh đúng phải dùng**. "Đừng làm thế, hãy làm thế này" là câu hay bị thiếu nhất
trong mọi tài liệu quy trình.

### Ba mức, và wow đứng TRÊN nền chứ không thay cho nền

**NỀN** (không làm là lỗi) → **TỐT** (đúng chuẩn đã hứa) → **WOW** (vượt điều
khách chờ đợi).

Cổng `xinWow()` **từ chối** trả về mức wow khi mức nền của chính ô ấy chưa làm
xong, và nói ra phải làm gì trước. Một bất ngờ dễ thương đặt trên một lời hứa
chưa giữ không phải dịch vụ tốt — nó là trò đánh lạc hướng, và khách hàng nhận
ra nhanh hơn ta tưởng.

**Nhóm cảm xúc và an toàn KHÔNG có mức wow.** Không phải vì thiếu ý tưởng, mà
vì sáng tạo ở nhóm ấy là đem một đứa trẻ ra thử nghiệm. Cổng chặn cứng.

### Hệ thống tự khai chỗ nó KHÔNG canh

**29/40 tình huống** máy tự phát hiện từ tín hiệu thật (`telemetry`, `mastery`,
lưới 50 ô, phiên đăng nhập của phụ huynh). **11 tình huống còn lại phải chờ
NGƯỜI báo** — và danh sách 11 ấy in ra rõ ràng, vì để người trực tưởng máy đang
canh cả bốn mươi là cách tạo ra một đêm không ai trông.

### Trao quyền có trần, và một quyền tự khai là chưa dùng được

10 quyền, mỗi quyền một **trần cứng** — trần là thứ cho phép trao quyền mà
không phải hỏi ai. **9 quyền dùng được; quyền Q08 (hoàn tiền) còn treo** vì sổ
cái dòng tiền vẫn khai doanh thu là khoản chưa nối nguồn. Có mục kiểm bắt hai
chỗ ấy phải nói **cùng một điều** — lệch nhau là một chỗ đang nói dối.

### Đo lường: chỗ mù in trước con số

**7/12 nhóm chỉ số đo được** từ dữ liệu thật. **5 nhóm còn lại — giữ chân, CLV,
giới thiệu, hài lòng, chi phí phục vụ — CHƯA nối được nguồn**, và bảng đo in
chúng **trước**. Chuỗi "dịch vụ → trung thành → lợi nhuận" hiện mới đo được
đoạn đầu; nói rằng đã đo được cả chuỗi là nói quá.

Mô-đun đo lường nằm **sau bức tường**, cùng phía với Cây Tiền — chính bộ soi
của bức tường đã bắt nó khi nó suýt bị xếp sang mặt khách hàng, vì nó nói bằng
ngôn ngữ thương mại. Đó là lần bức tường làm đúng việc của nó, và cách xử lý
đúng là để tệp ấy ở lại bên này, không phải nới bảng từ cấm.

```bash
gita trai-nghiem                                        # khung 10 tầng + lưới + bảng đo
gita trai-nghiem --o=T4C6 --tinh-huong=TH36 --kenh=K3   # một ô cụ thể
gita trai-nghiem --phuc-hoi                             # thang phục hồi 10 bước
```

---

## 4k. Thẻ điểm cân bằng — bản đồ nhân quả, rủi ro, an toàn tài chính và pháp lý

Sai lầm lớn nhất khi dùng thẻ điểm cân bằng là biến nó thành **một bảng KPI**:
liệt kê bốn nhóm chỉ số, giao chỉ tiêu, cuối tháng chấm điểm. Làm thế thì nó
thành công cụ đánh giá nhân sự, và mất đúng thứ làm nên giá trị của nó — **các
mũi tên, không phải các ô**.

### Bản đồ được máy canh, không được hứa

**17 mục tiêu · 23 cạnh giá trị + 4 cạnh phòng tổn thất · 55 chuỗi**, tất cả
đều kết thúc ở tầng tài chính. Năm luật, cưỡng chế bằng mã:

1. Mục tiêu ngoài tầng tài chính phải có cạnh **đi ra** (không mồ côi đầu ra).
2. Mọi chuỗi phải **tới được tầng tài chính** — tắt giữa đường nghĩa là ta
   không biết mục tiêu ấy dẫn tới đâu.
3. **Không vòng lặp** — giải thích A bằng B và B bằng A nghe hợp lý và không
   kiểm được.
4. Cạnh chỉ đi **lên một tầng**. Năng lực không tạo ra tiền trực tiếp.
5. Mục tiêu ngoài tầng nền phải có cạnh **đi vào** (không mồ côi đầu vào).

**Bộ soi bắt chính người viết nó ba lần** trong bản dựng đầu: ba cạnh nhảy tầng
thẳng lên tài chính (luật 4), và hai mục tiêu tài chính không chuỗi nào dẫn tới
(luật 5 — luật này được thêm *vì* bộ soi tìm ra chúng).

### Hai loại cạnh, và vì sao phải tách

Ba cạnh nhảy tầng không phải lỗi dữ liệu mà là **thiếu sót của mô hình**. Một
vụ lộ dữ liệu trẻ em tốn tiền **ngay lập tức** — nó không đi qua "gia đình hài
lòng hơn". Bắt nó đi vòng qua tầng khách hàng cho hợp luật là **vẽ sai cách
tiền mất**.

| Cạnh | Nghĩa | Luật |
|---|---|---|
| `dan` → | chuỗi tạo giá trị | đi từng tầng một, không nhảy |
| `chan` ⇢ | liên kết phòng tổn thất | nhảy tầng được, **nhưng chỉ tới mục tiêu phòng tổn thất** |

Ràng buộc thứ hai giữ `chan` khỏi thành cửa sau để lách luật 4.

### Luật cứng nhất: cấm đặt chỉ tiêu cho thứ chưa đo được

**36 thước đo · 25 đo được · 11 chưa nối nguồn — và 11 cái ấy KHÔNG có chỉ
tiêu.** "Giữ chân 85%" trong khi hệ thống chưa có khái niệm *"đang là khách
hàng"* không sai — nó **vô nghĩa**, và tệ hơn là làm người đọc tưởng có ai đó
đang theo dõi.

Mục kiểm này cũng bắt chính tôi: sáu thước đo bị đặt chỉ tiêu sau khi đã khai
là chưa đo được. Đã gỡ sạch chỉ tiêu và ghi rõ **thiếu đúng cái gì**.

Luật đi kèm: mỗi mục tiêu phải có ít nhất **một chỉ báo dẫn dắt đo được** —
thẻ điểm toàn chỉ báo kết quả thì đọc xong chuyện đã rồi. Mục kiểm bắt được
hai mục tiêu ở đúng tình trạng ấy (QT5 và KH4) và cả hai đã được bổ sung chỉ
báo dẫn dắt **đo được thật**.

### Sổ rủi ro: biện pháp phải có ĐỊA CHỈ

Phần lớn sổ rủi ro là danh sách nỗi lo kèm cột "biện pháp" viết bằng động từ
mềm: *tăng cường, chú trọng, nâng cao*. Những chữ ấy không chặn được gì.

**12 rủi ro**, mỗi rủi ro khai ba thứ bị đi kiểm từng cái:

| | |
|---|---|
| `chan` | tệp mã **đang chạy** — mục kiểm kiểm nó tồn tại trên đĩa |
| `thay` | mã thanh tra viên **có thật**, hoặc nói thẳng *"chưa có gì canh"* |
| `xuLy` | mã kịch bản trực **có thật** |

**Rủi ro mức NGHIÊM TRỌNG phải có đủ cả ba** — không có chỗ nào để ghi "đang
theo dõi". 9/12 rủi ro có mục soi tự động; **3 rủi ro không ai canh tự động
được nêu tên** (đều ở mức NẶNG, chấp nhận có ý thức, không bị bỏ quên).

Rủi ro **không** chấm bằng tích *xác suất × mức độ*: phép nhân ấy làm một rủi
ro hiếm-nhưng-chết-người tụt xuống ngang một rủi ro thường-mà-nhẹ, rồi cả hai
cùng nằm giữa bảng và không ai làm gì. Mức do **hậu quả nặng nhất** quyết định.

### An toàn tài chính: 3 cửa CHẶN CỨNG, và chỗ mù nêu tên

Cả ba cửa đều `chan-cung`, không phải cảnh báo — hai thứ đó khác nhau rất xa
lúc 3 giờ sáng. Chưa nối doanh thu thì **runway, biên lợi nhuận, rủi ro tập
trung đều chưa trả lời được**, và không được ước lượng để trả lời cho có.

### An toàn pháp lý: hệ thống KHÔNG tự chứng nhận

Đây là phần quan trọng nhất và cũng là phần dễ làm sai nhất.

**Một hệ thống phần mềm không thể tự chứng nhận mình tuân thủ pháp luật**, và
không có chế độ nào để bật việc đó. Nó chỉ làm được hai việc: liệt kê **nghĩa
vụ nó tự đặt ra** kèm chỗ trong mã thực hiện, và đòi mỗi nghĩa vụ có **tên một
người đã ký nhận**.

**0/6 nghĩa vụ hiện có người ký** → trạng thái của cả sáu là **chưa xác nhận**,
không phải "đạt". Và: *các số hiệu văn bản pháp luật trong bảng là do người
dựng hệ thống khai, KHÔNG do hệ thống tra cứu* — phải để người có chuyên môn
pháp lý đối chiếu với văn bản gốc còn hiệu lực.

### Vòng lặp chỉ khép lại ở bước SỬA BẢN ĐỒ

Ba nhịp, ba câu hỏi khác hẳn nhau — trộn chúng vào một cuộc họp là cách chắc
chắn nhất để chỉ còn lại câu dễ nhất: *"tháng này số có đẹp không"*.

| Nhịp | Câu hỏi | Được quyết gì |
|---|---|---|
| **Ngày** (máy) | Có gì đang hỏng ngay bây giờ? | dừng việc nếu mục soi mức chặn đỏ |
| **Tháng** | Chỉ báo **dẫn dắt** đi hướng nào? | đổi cách làm ở tầng quy trình |
| **Quý** | **Giả thuyết nhân quả** còn đúng không? | **sửa bản đồ** — nhịp duy nhất được sửa |

*Rà soát mà không bao giờ sửa bản đồ thì đó là báo cáo, không phải quản trị
chiến lược.* Hệ thống cũng tự khai: **chưa có kho lưu biên bản**, nên nó không
biết một cuộc rà soát đã diễn ra hay chưa.

```bash
gita the-diem                 # thẻ điểm đầy đủ, phần CHƯA BIẾT in trước
gita the-diem --chuoi=QT4     # chuỗi nhân quả từ một mục tiêu
```

---

## 4l. Ba cổng an toàn — dựng sau khi soi một bản thiết kế 47 mô-đun bằng máy

Một bản thiết kế 1,18 triệu ký tự (V13→V22, 47 mô-đun từ A tới AW) được đọc
**bằng máy**: đối chiếu danh sách `import` với danh sách định nghĩa, đếm từ
khoá, dò chuỗi. Bảy chỗ hỏng lộ ra — và **không chỗ nào lộ khi đọc từng phần**.
Mỗi mô-đun đọc riêng đều hợp lý. Chúng chỉ lộ khi **đối chiếu** hoặc khi **đếm**.

### Bảy phát hiện

**1. Móng chưa đổ, mà 47 tầng lầu đã vẽ xong.** **18 đường dẫn** được `import`
mà **không tệp nào định nghĩa**, tổng **265 chỗ gọi**. Trong đó bảy mô-đun
**gánh an toàn**:

| Mô-đun | Số chỗ gọi |
|---|---|
| `utils/crypto` | 46 |
| `middleware/auth` | 39 |
| `utils/error` | 38 |
| `middleware/rateLimit` | 15 |
| `services/costGuard` | 10 |
| `services/jobQueue` | 5 |
| `utils/logger` | 3 |
| **cộng** | **156** |

Mật mã, xác thực, xử lý lỗi, giới hạn tần suất, trần chi phí, hàng đợi, nhật
ký. (Còn `types/content` 84 chỗ — tệp khai kiểu, tách riêng vì không gánh hành
vi; và mười mô-đun nhỏ khác, 25 chỗ.)

> **Một lần đếm sai, ghi lại vì nó đúng là bài học của phần này.** Bản báo cáo
> đầu tiên viết *"7 mô-đun, 163 chỗ gọi"* — dựng từ lần đếm đầu, khi dừng lại ở
> bảy cái nhìn thấy trước và cộng nhầm một mô-đun. Đếm lại bằng một biểu thức
> duy nhất quét mọi câu `import` cho ra **18 và 265 — nhiều hơn, không phải ít
> hơn**. Bài học không phải "hãy cẩn thận": nó là **đừng đếm bằng mắt rồi ghi
> vào báo cáo**. Đó chính là việc `src/an-toan/nen-mong.js` làm thay.

**2. Không có cổng tuổi, không có đồng ý của người giám hộ — ở đâu cả.** Mọi
lần nhắc tới tuổi / age gate / parental consent / COPPA trong toàn bộ tài liệu
nằm gọn trong **một danh sách từ khoá của một luật dò nội dung**. Câu khắc phục
của luật ấy viết: *"Thêm age gate và parental consent"* — lời khuyên hệ thống
đưa cho người khác, trong khi chính nó có ghép đôi bạn học **với người lạ**,
hỏi đáp ẩn danh, lớp học ảo, sự kiện mở và ghi âm giọng nói. Không một trường
tuổi, không một ngày sinh trong 47 mô-đun.

**3. Một khoá làm hai việc, cắt ra và đệm số không.**
`env.JWT_SECRET.slice(0, 32).padEnd(32, '0')` — một dòng, ba lỗi chồng nhau:
dùng lại khoá ký làm khoá mã hoá, cắt thay vì dẫn xuất, và đệm `'0'` khiến một
bí mật ngắn vẫn ra khoá "đủ dài" mà không ai được báo.

**4. Bộ lọc quấy rối khớp từ khoá không bỏ dấu.** `new RegExp(kw, 'gi')` trên
tiếng Việt: viết không dấu là đi qua.

**5. Máy tự huỷ kết quả thi, không cần người.** Sự kiện `multiple_faces` phạt
0.50, `maxAllowed: 1`, và mọi sự kiện mức `critical` kích hoạt tự huỷ. Một
người nhà đi ngang sau lưng là đủ.

**6. Kế hoạch lui gồm hai lệnh** cho một hệ thống 47 mô-đun — không thứ tự,
không cách kiểm lại, không nói dữ liệu ghi trong lúc lui đi đâu.

**7. Lời hứa "$0/tháng MÃI MÃI"** lặp ở tiêu đề mười phiên bản, dựa hoàn toàn
vào `costGuard` — chính là một trong bảy mô-đun chưa ai viết.

### Ba cổng đã dựng trong hệ thống này

**Cổng tuổi** (`src/an-toan/cong-tuoi.js`) — bốn luật cưỡng chế bằng máy:

1. **Mặc định là ĐÓNG.** Tính năng chưa khai mức tiếp xúc thì bị từ chối.
2. **Chưa biết tuổi là một câu trả lời, và câu ấy là KHÔNG.** Không có chỗ nào
   đoán tuổi từ lớp học, cách viết hay giọng nói.
3. **Khai báo tuổi không phải xác minh tuổi.** Lời khai mở được tính năng không
   tiếp xúc; **không** mở được tính năng chạm người lạ.
4. **Đồng ý của người giám hộ phải có TÊN, QUAN HỆ, LÚC và ĐƯỜNG RÚT LẠI.**
   Một ô tích không tên không phải sự đồng ý; nó là một ô tích.

**6 tính năng rủi ro chưa dựng đã được khai sẵn** — ghép bạn học, hỏi đáp ẩn
danh, lớp học chung, sự kiện mở, học bằng giọng nói, học viên đăng nội dung —
để ngày ai đó dựng chúng thì **cổng đã đứng sẵn**, chứ không dựng cổng sau khi
đã có một đứa trẻ đi qua.

**Soi nền móng** (`src/an-toan/nen-mong.js`) — quét chính kho mã này mỗi lần
kiểm định: **223 tệp · 1.056 lời gọi mô-đun, tất cả trỏ vào tệp có thật, 0 tệp
cô độc**. Đây là loại lỗi mà đọc từng tệp không bao giờ bắt được.

**Khoá riêng từng việc** (`src/an-toan/khoa-rieng.js`) — HKDF-SHA256, nhãn việc
nằm trong `info`; 5 việc ra 5 khoá không suy ra được nhau. **Bí mật ngắn bị TỪ
CHỐI, không đệm**; bí mật dài mà nghèo biến thiên (`'0'×40`) cũng bị từ chối.

### Một lỗi của chính bức tường, do mô-đun mới làm lộ ra

Khi cho cổng tuổi chạy qua bộ quét cụm từ cấm, nó **báo động nhầm**: câu *"Nhóm
được bảo vệ chặt nhất. Mọi tiếp xúc…"* sau khi bỏ dấu thành `…chat nhat moi
tiep xuc…`, trong đó có chuỗi con `hat moi` — đúng bằng tên một nhóm nội bộ.

Một bộ soi an ninh báo động nhầm không phải chuyện nhỏ: người trực gặp vài lần
là thôi đọc, và lần nó báo đúng thì cũng không ai nhìn. Đã sửa bộ quét để chỗ
khớp phải nằm **trọn trong ranh giới từ** — luật này **không nới lỏng** phép
soi, vì một cụm cấm có thật bao giờ cũng bắt đầu ở đầu một từ.

Lần thứ hai bức tường báo thì nó **báo đúng**: tài liệu thiết kế có viết nguyên
đường dẫn chứa cụm cấm. Cách xử lý đúng là **đổi chữ trong tài liệu, không nới
bảng cấm**.

```bash
gita an-toan                                              # ba cổng + bảy câu hỏi
gita an-toan --chi-tiet                                   # bảy câu, kèm chỗ hỏng gốc
gita an-toan --tinh-nang=GHEP_BAN_HOC --tuoi=14 --cach-biet=TU_KHAI
```

---

## 5. Chống nhiễu cho 500 agent hoạt động xuyên suốt

7 cơ chế: lọc nhiễu đầu vào (10 luật) · ngắt mạch theo agent · cách ly & tự chữa ·
giảm chấn dao động · áp lực ngược khi hàng đợi sâu · ngăn khoang theo khối ·
chó canh phát hiện agent treo (120 giây) và **giải phóng chỗ bị kẹt**.

Chỉ số sẵn sàng đội hình đo liên tục: `GET /resilience/fleet`.

**Chạm trần công suất không được phép nuốt mất mảnh việc.** Hai loại trần cần
đối xử khác nhau, và gộp chúng làm một là chỗ báo cáo bắt đầu nói sai:

* `*_CONCURRENCY` — agent hoặc khối đang chạy đủ số việc song song. Chuyện này
  qua trong vài chục mili giây, nên hệ thống **đợi đến lượt** (giãn cách
  40 · 120 · 300 ms) rồi giao lại, vẫn tôn trọng trần.
* `*_TOKEN_BUDGET` — đã tiêu hết hạn mức token của **một phút**. Cửa sổ ấy dài
  60 giây; thử lại trong lòng một đợt huy động chỉ tốn thời gian rồi vẫn trượt.
  Hệ thống **chuyển mảnh việc cho đồng đội cùng khối** — đây đúng là lý do tồn
  tại của một lực lượng 500 người. Vẫn phải **cùng khối**: điều A13 buộc loại
  việc khớp chuyên môn, nên đẩy bài toán sang khối ngôn ngữ cho đủ chỉ tiêu là
  vi phạm chứ không phải thành tích.

Hết người rảnh trong khối thì chịu, và báo cáo **gọi tên** mảnh chưa làm kèm
lý do thật (`chuaLam`, `luotDoiCongSuat`, `luotChuyenNguoi`). Trước khi có cơ
chế này, một đợt tổng động viên trong lúc hệ thống đang tải báo **9/12 mảnh**
và quy cho "lá chắn chống nhiễu"; lý do thật là `AGENT_TOKEN_BUDGET` của đúng
ba agent. Sau khi chuyển việc cho đồng đội: **12/12**.

## 6. Chương trình học — phân cấp và đo lường

* **Phân cấp**: dạng (104 dạng **biên soạn tay**, mỗi dạng có tiên quyết, bẫy thường gặp,
  chuẩn đầu ra) → độ khó 1★–5★ → 10 tầng L0…L9 → 5 khối lớp theo trình độ và độ tuổi.
* **Chuẩn lên tầng siết chặt dần, không bao giờ nới lỏng**: 7 tiêu chí bắt buộc đạt đủ —
  mastery, độ chính xác, tốc độ, độ ổn định, độ phủ, bài thi cuối tầng, không tụt lùi.
  Từ L0 (mastery 0.70, tốc độ ×2.00) đến L9 (mastery 0.95, tốc độ ×0.90).
* **Thứ tự học** sắp bằng topo theo tiên quyết, đã kiểm chứng không có vòng lặp.

### Không có bài trùng lặp, không dập khuôn máy móc

Ba lớp chặn, chạy trước khi bất kỳ bài nào được nhận vào kho:

| Lớp | Cách bắt | Bắt được cái gì |
|---|---|---|
| Y hệt | SHA-256 nội dung | Chép nguyên bài |
| Gần giống | SimHash-64 (hamming ≤6) **và** Jaccard trên shingle (≥0.75) | Sửa vài chữ |
| Trùng khuôn | Chữ ký khung bài (mọi chữ số → `#`), hạn mức 3 bài/khung | **"Đổi số cho khác đi"** |

Chất lượng xuất bản: 5 tiêu chí có trọng số (chính xác 0.40 – tối thiểu 5★, ký hiệu 0.20,
sư phạm 0.20, độc đáo 0.10, hiệu chỉnh độ khó 0.10), **ngưỡng phát hành 4.5/5.0**.

## 7. Chân dung học viên & chu kỳ 90 ngày

Hệ thống quan sát **thao tác → hành vi → suy nghĩ → kết quả**:

* **Thao tác**: từng hành động có nhãn, có dấu thời gian.
* **Hành vi**: 8 mẫu hành vi (đoán mò, bỏ cuộc sớm, làm vội, lệ thuộc gợi ý, …).
* **Suy nghĩ**: đối chiếu lỗi sai với **đúng cái bẫy** đã khai báo trong dạng bài —
  biết học viên hiểu sai ở đâu, không chỉ biết là sai.
* **Kết quả**: BKT (Bayesian Knowledge Tracing) + lặp lại ngắt quãng SM-2, mastery ngưỡng 0.85.

**Chân dung** gồm 6 trục năng lực (kiến thức · chính xác · tốc độ · tự lực · kiên trì · chiều sâu)
và xếp hạng 5★. **Chu kỳ 90 ngày** chia 3 pha, có **8 luật dịch chuyển chiến lược**
(S1 sửa nền … S8 giữ nhịp) tự chọn hướng can thiệp theo đúng dữ liệu đo được.

## 8. Hai lớp ứng dụng

### Web app
Giao diện Super Admin tại `/ui/index.html`: đăng nhập nhiều lớp, 8 tab
(Tổng quan · 500 Agent · Tổng động viên · Chương trình · Học viên · Bảo mật · Khôi phục · Hiến pháp).
Thuần JavaScript, **không build step, không CDN ngoài, không gọi ra mạng ngoài**,
sáng/tối theo hệ thống, dùng được trên màn hình hẹp.

### Bản cài đặt Windows 64-bit
```bash
cd desktop
npm install
npm run dist:win     # sinh trình cài NSIS + bản chạy thẳng (portable), cả hai đều x64
```
* Nexus chạy **ngay trong tiến trình Electron** — không cần cài Node, không cần mở terminal.
* Chỉ nghe trên `127.0.0.1`, không mở ra mạng.
* `contextIsolation: true`, `nodeIntegration: false`, `sandbox: true`, chặn mọi điều hướng ra ngoài.
* Menu tiếng Việt, biểu tượng khay hệ thống, khoá một bản chạy.
* **Tự chụp ảnh hệ thống khi khởi động và khi thoát**; dữ liệu ghi vào thư mục hồ sơ
  người dùng, gỡ cài đặt **không** xoá dữ liệu học viên.


## 6a. Dữ liệu và tài khoản

**Lưu trữ bền vững** không cần hạ tầng ngoài: ảnh chụp nén + nhật ký ghi trước,
ghi nguyên tử bằng đổi tên. Mất điện giữa chừng thì dòng nhật ký cụt bị bỏ chứ
không làm hỏng cả bộ sưu tập. Có kiểm thử mô phỏng tiến trình bị giết.

**Năm vai** với bảng quyền có phạm vi, không rải `if` khắp nơi:

| Vai | Thấy được gì |
|---|---|
| Quản trị | toàn hệ thống |
| Giáo viên | **đúng lớp mình dạy**, không chạm được lớp khác |
| Phụ huynh | **đúng con mình**, không xem dữ liệu cả lớp |
| Học viên | **đúng hồ sơ của mình** |
| Khách | chỉ thông tin công khai |

Mật khẩu ngặt dần theo vai; sai nhiều thì khoá theo thang tăng dần; đổi mật
khẩu cắt hết phiên đang mở. Đăng nhập sai trả **HTTP 401**, không phải 200.

**Nhập/xuất CSV**: tự nhận dấu chấm phẩy của Excel bản Việt, có BOM khi xuất,
chạy thử trước khi ghi, và **không dòng nào bị bỏ qua im lặng** — mỗi dòng hỏng
có số dòng và lý do bằng tiếng Việt.

## 6b. Kiểm tra toán học — không tin lời mô hình

Bộ đọc biểu thức riêng, **không dùng `eval`**: chuỗi do mô hình sinh ra được
bóc thành cây cú pháp rồi mới tính. Nội dung là **dữ liệu**, không bao giờ là mã.

Máy tự giải lại trước khi bất kỳ bài nào vào kho:

- so hai biểu thức bằng cách lấy mẫu **tất định** ở nhiều điểm, có xét miền xác định
- giải chính xác bậc nhất và bậc hai; ngoài phạm vi đó thì **dò bằng số** và
  **nói rõ là đã dò**, kèm đoạn đã quét và điểm cực bị loại
- thay nghiệm ngược vào phương trình gốc
- bắt được **thiếu nghiệm**, **thừa nghiệm**, và bài rút gọn **thiếu điều kiện xác định**

## 6b-bis. Ma trận đa tầng — năm hệ chuẩn gặp mười tầng

Hệ thống đo bằng **tầng**; gia đình hỏi bằng **điểm** của kỳ thi. Không có cầu
nối thì hai bên nói hai thứ tiếng, và mọi con số nội bộ vô nghĩa với người trả
tiền. Ma trận đa tầng là cầu nối đó.

**Phát hiện làm ma trận đứng được.** Đọc máy toàn bộ năm bộ tài liệu đặc tả
(HSA 1300, TSA 100, SPT trong tài liệu TSA, SAT 1600, IELTS 9.0) cho thấy cả
năm hệ đều chia trình độ thành **đúng mười nấc cùng tên, cùng thứ tự**:
Foundation → Emerging → Developing → Intermediate → Proficient → Advanced →
Highly Proficient → Elite → nấc thứ chín tuỳ hệ → Perfect. Mười tầng năng lực
của hệ thống cũng đúng mười nấc. Nên **không cần thang thứ hai**: tầng chính
là xương sống, mỗi hệ chuẩn chỉ là một cách quy đổi ra đồng tiền riêng.

| Tầng | Nấc chung | HSA 1300 | TSA 100 | SPT 1200 | SAT 1600 | IELTS 9.0 |
|---|---|---|---|---|---|---|
| 0 | Foundation | 150–350 | 0–20 | 0–200 | 400–600 | 0–3.5 |
| 3 | Intermediate | 650–800 | 40–50 | 550–700 | 900–1050 | 5–5.5 |
| 6 | Highly Proficient | 1000–1100 | 70–80 | 900–960 | 1300–1400 | 6.5–7 |
| 9 | Perfect | 1280–1300 | 98–100 | 1120–1200 | 1560–1600 | 8.5–9 |

**Sáu trục.** Hệ chuẩn · tầng · phần thi · dạng bài · bậc sao · lớp. Ma trận là
chỗ sáu trục cắt nhau, và mỗi ô lần ngược về được tới dữ liệu gốc.

**Không bịa điểm.** Hệ thống không nói "em được 1250 điểm HSA". Nó nói "em ở
tầng 8, tầng 8 ứng với khoảng 1200–1280". Khoảng là thứ đo được; con số đơn lẻ
là thứ bịa. Có điểm thi **thật** thì điểm thật thắng mọi ước lượng và ma trận
quay ngược lại để chỉnh tầng — độ lệch giữa hai bên là tín hiệu cho biết hệ
thống đang đánh giá cao hay thấp hơn thực tế.

### Thanh tra toàn diện — và kết quả thật

`gita ma-tran` chạy hai nhà khoa học đi soi toàn bộ ma trận. Kết quả hiện tại,
báo đúng như máy đo được:

| Hệ chuẩn | Phủ đề | Chỗ hổng |
|---|---|---|
| TSA 100 | đủ | — |
| SAT 1600 | đủ | — |
| IELTS 9.0 | đủ | — |
| **HSA 1300** | **thiếu 25%** | Ngôn ngữ và văn học — 50/200 câu, chưa có dạng bài nào |
| **SPT 1200** | **thiếu 25%** | Khoa học xã hội — 40/160 câu, chưa có dạng bài nào |

Hai chỗ hổng này là **hai môn hệ thống không dạy**: đây là hệ thống toán và
tiếng Anh. Hệ thống nêu thẳng ra để gia đình biết mà bù bằng nguồn khác, thay
vì im lặng bán trọn gói luyện HSA rồi để học viên gặp một phần tư số câu lần
đầu ngay trong phòng thi. Thước đo là **phần trăm số câu của đề thật**, không
phải số ô trong bảng — một phần 50 câu hổng nặng hơn hẳn một phần 20 câu.

Trục tiếng Anh: **12/12 dạng đều đã có học liệu** — nghe 1/1, đọc 2/2, viết
2/2, nói 1/1, nền (ngữ pháp và từ vựng) 6/6. Bốn dạng từng để trống (đọc hiểu
ý chính, đọc hiểu suy luận, viết luận, nói theo chủ đề) nay dựng trên ngữ liệu
**viết tay** — xem mục 6e. Sáu bậc CEFR phủ đủ lớp 1 đến lớp 12, và bốn tiêu
chí chấm chính thức của IELTS được khai đủ cho cả viết lẫn nói.

Nói thẳng chỗ hệ thống **chưa làm**: máy không chấm bài luận và bài nói thành
band. Việc đó cần người chấm. Hệ thống dạy và kiểm bước TRƯỚC KHI viết và nói
— chọn đúng dạng đề, dựng đủ ý, chạm đủ bốn gạch đầu dòng — và đó đúng là chỗ
mất điểm Task Response nhiều nhất.

---

## 6b-quater. Chuỗi giá trị và điểm chạm — 500 trợ lý gắn vào việc khách hàng làm

Câu hỏi mục này trả lời: trong cả hành trình của một học viên, **chỗ nào** thật
sự làm nên kết quả, và **tổ nào** trong 500 trợ lý chịu trách nhiệm ở chỗ đó.

| Chặng | Dấu vết đo được | Tổ phụ trách | Hỏng thì khách hàng mất gì |
|---|---|---|---|
| Mở buổi học | `SESSION_START` | TUTOR · coach-1v1 | Mở ứng dụng rồi đóng vì không biết hôm nay làm gì |
| Mở một bài | `ITEM_OPEN` | MATH · item-authoring | Mở bài rồi bỏ — sai tầng hoặc đề viết rối |
| Xin gợi ý | `HINT_REQUEST` | TUTOR · socratic | Gợi ý nói luôn đáp án, qua bài mà không học được gì |
| Nộp bài | `SUBMIT` | MATH · solution-writing | Chấm sai một lần là mất niềm tin |
| Xem lời giải | `SOLUTION_PEEK` | VIETNAMESE · lang-writing | Lời giải rối thì đọc lướt rồi tưởng đã hiểu |
| Xem lại bài sai | `REVIEW_MISTAKE` | TUTOR · strict | Bài sai trôi đi, rồi sai đúng chỗ đó trong phòng thi |
| **Bỏ bài giữa chừng** | `SKIP` | RESEARCH · curriculum | Bỏ bài thành thói quen, rồi bỏ buổi, rồi bỏ hẳn |
| Đóng buổi học tử tế | `SESSION_END` | BUSINESS · bi | Buổi học kết thúc bằng việc tắt tab |

**Ba nguyên tắc, viết ra để không tự lừa mình bằng số đẹp:**

1. **Chỉ đo cái có dấu vết.** Chặng nào hệ thống chưa ghi được dấu vết thì
   KHÔNG có trong bảng — kể cả những chặng ai cũng biết là có (nghe bạn giới
   thiệu, đọc trang công khai rồi đóng tab). Bịa một chặng rồi gán số cho nó là
   việc dễ nhất và vô dụng nhất.
2. **Không kết luận từ mẫu nhỏ.** Mỗi nhóm so sánh phải có tối thiểu 10 học
   viên, mỗi người từ 10 bài trở lên. Dưới ngưỡng thì trả đúng chữ *"chưa đủ
   căn cứ"* kèm số người đang có — không trả một tỉ lệ.
3. **So sánh hai nhóm, không khoe một nhóm.** "90% người xem lại bài sai thì
   tiến bộ" là câu vô nghĩa nếu không biết nhóm KHÔNG xem lại tiến bộ bao nhiêu.
   Mọi kết luận đều là **hiệu** của hai nhóm, kèm cỡ mẫu của cả hai.

Và một điều mục này **không** làm: không gọi hiệu số đo được là quan hệ nhân
quả. Người chịu xem lại bài sai vốn đã là người chăm hơn. Hiệu số nói *"chỗ này
đáng nhìn kỹ"*, không nói *"làm thế này thì chắc chắn tăng"*. Chặng đánh dấu
xấu (bỏ bài) không bao giờ được nhận là "điểm chạm tốt nhất", dù hiệu số ra sao.

```bash
node bin/gita.js diem-cham          # bảng chuỗi giá trị + đo trên dữ liệu thật
curl localhost:3000/reports/diem-cham   # đo trên máy chủ đang chạy
```

---

## 6b-sexies. Bộ khảo sát sáu công cụ — và luật đo đứng trên chúng

Sáu công cụ, và điều quan trọng nhất phải nói trước mọi con số: **chúng không
cùng sức nặng bằng chứng.** Một bài kiểm tra toán chấm được đúng sai; một bảng
hỏi tính cách chỉ ghi lại điều người ta *tự nói* về mình; thần số học thì không
đo gì cả. Trộn cả sáu vào một "hồ sơ toàn diện" rồi để chúng cùng quyết định
học viên học gì là cách nhanh nhất biến một hệ thống đo lường thành một cái máy
bói có giao diện đẹp.

Nên `src/khao-sat/hien-phap-do.js` ra trước mọi tệp khác, và nó là **luật**:

| Công cụ | Loại | Được ảnh hưởng tới |
|---|---|---|
| Đề đánh giá năng lực môn học | có đúng sai | **nội dung** · nhịp · cách nói |
| Khảo sát phương pháp học | tự khai, **đối chiếu được** | **nội dung** · nhịp · cách nói |
| Sàng lọc tâm lý học đường | tự khai | nhịp · cách nói · cảnh báo |
| DISC | tự khai | cách nói |
| MBTI | tự khai | cách nói |
| **Thần số học** | không đo gì | **không miền nào** |

**Ranh giới quan trọng nhất:** chỉ hai công cụ có bằng chứng về hành vi học tập
mới được chạm vào **nội dung**. Tính cách không quyết định học viên phải học
dạng bài nào. Một em hướng nội vẫn phải học đủ phương trình bậc hai như một em
hướng ngoại, và gợi ý ngược lại là phân biệt đối xử được bọc trong ngôn ngữ tâm
lý học.

Luật này **được máy cưỡng chế**: bộ dựng lộ trình phải gọi `duocAnhHuong()`
trước mỗi lần dùng một kết quả, và ghi vết vào `vetNguon`. Khi bị từ chối, nó
**không im lặng bỏ qua** mà ghi vào `daTuChoi` — nên câu hỏi *"MBTI của con có
ảnh hưởng tới bài không"* có câu trả lời bằng dữ liệu, không bằng lời hứa.

### Sáu công cụ, và điều mỗi công cụ từ chối làm

* **DISC** — 20 tứ bộ ép chọn, viết tay cho tình huống học tập. Thang **ipsative**:
  bốn nhóm so với nhau *trong cùng một người*, không so được giữa hai học viên.
  Bốn nhóm cách nhau dưới 2 điểm thì hệ thống **không ép ra nhãn** — ép ra một
  nhãn lúc đó là gán cho em một tính cách mà bài làm không nói.
* **MBTI** — 32 câu, 4 cặp. Hệ thống làm ba việc bản thương mại thường không
  làm: in **độ nghiêng** từng cặp; đánh dấu `?` ở cặp **sát ranh giới**; và
  chặn cứng MBTI khỏi miền nội dung. Thiếu câu ở *một* cặp là không chấm, dù
  tổng vẫn nhiều — chữ cái ở vị trí đó sẽ là bịa.
* **Thần số học** — tính đúng quy ước (rút số, giữ 11/22/33, bảng Pythagore).
  Kết quả **luôn** mang theo `dungVaoViec: 'không'` và câu cảnh báo. Hệ thống
  cũng nói ra chỗ phép tính này thô: tên tiếng Việt bị **bỏ dấu** nên "Tuấn" và
  "Tuân" cho cùng một con số. Mô tả viết ở thì *"người ta thường nói"*, không ở
  thì khẳng định.
* **Sàng lọc tâm lý học đường** — 24 câu, 6 miền, hỏi tần suất 2 tuần gần nhất.
  **Là sàng lọc, không phải chẩn đoán**, và không bao giờ in ra một cái tên
  bệnh. Miền *an toàn* đặt ngưỡng thấp có chủ ý: bị trêu chọc dù chỉ "một vài
  hôm" cũng báo ngay cho phụ huynh và giáo viên chủ nhiệm.
  **Điều hệ thống cố ý KHÔNG làm:** không sàng lọc ý định tự hại. Đặt câu hỏi
  ấy trong một bảng hỏi tự động, trong sản phẩm học tập không có người trực, là
  mở ra một câu trả lời mà không ai ở đó để đỡ. Việc ấy cần người được đào tạo,
  có quy trình và có mặt thật — hệ thống nói thẳng là mình không làm.
* **Khảo sát phương pháp học** — 8 miền × 3 câu. Đây là công cụ tự khai **duy
  nhất** được chạm vào nội dung, và được phép vì gần như mọi mục **đối chiếu
  lại được với dấu vết thật**. Em khai "luôn tự soát lại bài" còn bộ quan sát
  ghi tỉ lệ sai do ẩu 40% — **chỗ va nhau ấy đáng giá hơn cả hai con số đứng
  riêng**. Chưa có dấu vết thì hệ thống nói thẳng là chưa đối chiếu được.

### Đề đánh giá năng lực theo khung thời gian năm học

Đề kiểm tra ở trường cho học sinh giỏi 9–9,5 điểm, và con số ấy nói rất ít:
trong nhóm cùng 9,25 có em vững thật và có em vừa đủ vượt phần dễ. Đề nào cũng
chỉ phân loại được ở chỗ nó **đặt câu hỏi khó**.

Bộ đề ở đây giữ nguyên **phạm vi** theo đúng tuần của năm học — không hỏi thứ
chưa dạy — nhưng dồn trọng số sang nơi nhóm 9–9,5 thật sự khác nhau:

| | 1★ | 2★ | 3★ | 4★ | 5★ |
|---|---|---|---|---|---|
| Đề trường | 40% | 30% | 20% | 10% | 0% |
| **Đề ở đây** | 0% | 10% | 25% | **40%** | **25%** |

Sáu mốc bám đúng sáu thời điểm nhà trường thật sự dừng lại để đánh giá (đầu ·
giữa · cuối mỗi học kỳ), phủ 10% → 28% → 50% → 58% → 78% → 100% chương trình.
Mốc **đầu học kỳ I** là mốc các nơi hay bỏ qua nhất và lại quan trọng nhất: lỗ
hổng năm trước không tự lành, phát hiện ở tuần 4 thì còn cả năm để vá.

**Hệ quả phải nói trước, nếu không gia đình sẽ hoảng:** một em giỏi thật làm đề
này thường đúng **55–70%**, không phải 90%. Con số trên đề này **không phải
điểm trường**. Nên mọi kết quả đi kèm một phép quy đổi ngược luôn tự gọi mình
là **ước lượng**, và không bao giờ in ra một con số điểm trường cụ thể — không
đề trường nào giống đề trường nào.

### Lộ trình cá nhân hoá — sáu chặng, mỗi chặng một cửa

Mỗi chặng bám một mốc năm học và kết thúc bằng một **bài test cửa**; chặng cuối
là **bài thi tổng kết ở mức vận dụng cao** (30 câu). Điều kiện qua cửa có **hai
vế**, không phải một:

> Đúng ≥ 55% tổng đề **VÀ** ≥ 35% riêng phần 4★–5★.

Vế thứ hai là vế quan trọng: đủ tổng mà hụt phần khó nghĩa là em đang gom điểm
ở phần dễ của một đề vốn đã ít phần dễ, và chặng sau sẽ dựng đúng trên phần còn
hụt đó.

**Chưa qua cửa KHÔNG phải trượt.** Nó có nghĩa là chặng ấy được *kéo dài* và
nội dung được chỉnh — chặng sau dựng trên chặng này, đi tiếp là xây trên nền
rỗng. Với bài thi tổng kết: lùi kỳ thi, vá đúng phần còn hụt, thi lại — **không
hạ chuẩn đề**.

**Cửa an toàn.** Nếu sàng lọc tâm lý có miền cần báo ngay, hệ thống **không
dựng lộ trình** cho tới khi khai đủ `{ ten, vai, luc }` — ai đã nói chuyện với
em, vai trò gì, lúc nào — và **tên người ấy được ghi vào lộ trình**. Cửa không
mở bằng `true` hay một chuỗi qua loa. Đây không phải "cấm vĩnh viễn": một cái
cửa mở được bằng nút "bỏ qua" thì không phải cửa, nhưng một cái cửa không bao
giờ mở được thì người ta sẽ tắt hẳn bài sàng lọc đi, và thế còn tệ hơn.

### Làm bài được, lưu được, dùng được

Bản dựng đầu có đủ bộ chấm, đủ luật đo, đủ trang lộ trình — mà **không dùng
được**. Bản rà soát tìm ra ba lỗ cộng lại:

1. **Không có biểu mẫu** để học viên làm bài (0 form trong toàn bộ khu riêng).
2. **Không có chỗ lưu** kết quả (0 bảng trong cơ sở dữ liệu).
3. **Khu riêng không bao giờ ghi lớp vào hồ sơ học viên** — nên `lop` luôn rỗng.

Hệ quả: trang lộ trình cá nhân hoá **vĩnh viễn** hiện `CHƯA_ĐỦ_CĂN_CỨ` cho mọi
tài khoản thật. Tôi từng gọi việc không lưu là *"bảo đảm riêng tư mạnh nhất"* —
đó là một lời biện hộ cho một việc chưa làm xong. **Riêng tư đúng nghĩa là kiểm
soát truy cập, không phải mất dữ liệu.**

Nay: `/nha/khao-sat` liệt kê sáu bộ theo **ba tầng sức nặng**, bốn bộ làm được
ngay bằng **biểu mẫu HTML thật** (chạy cả khi tắt JavaScript), nộp xong chấm và
lưu, rồi lộ trình dựng từ kết quả thật. Thiếu lớp thì trang **hỏi thẳng bằng
một ô chọn** thay vì báo lỗi rồi để gia đình tự đoán phải vào đâu khai.

**Luật riêng tư khai theo TỪNG công cụ** (`src/khao-sat/kho.js`), không theo cả
kho:

| Công cụ | Ai đọc được (ngoài chính chủ) | Lưu câu thô |
|---|---|---|
| Đề năng lực · Phương pháp học · DISC · MBTI | phụ huynh, giáo viên, cố vấn | có |
| **Sàng lọc tâm lý** | **phụ huynh, cố vấn** (giáo viên **không**) | **không** |
| Thần số học | phụ huynh | không |

Với sàng lọc tâm lý, hệ thống **chỉ lưu điểm miền và kết luận**. Biết *"miền
giấc ngủ ở mức đáng chú ý"* là đủ để hành động; biết em trả lời câu nào mấy
điểm thì không thêm gì cho việc giúp em, mà lại là thứ nặng nhất nếu lộ.

**Mỗi lần làm là một bản ghi mới**, không ghi đè: xu hướng qua ba tháng nói
nhiều hơn một lát cắt, và ghi đè là xoá mất xu hướng ấy. Gia đình yêu cầu xoá
thì **xoá thật**.

```bash
node bin/gita.js khao-sat --lop=9     # luật đo + khung năm học + ma trận đề
curl localhost:3000/khao-sat/luat-do
```

---

## 6b-quinquies. Hai cái cây — và bức tường giữa chúng

Hệ thống có **hai** cái cây, và chúng không bao giờ được gặp nhau.

| | **Cây giá trị** | **Cây Tiền** |
|---|---|---|
| Ai xem | mọi gia đình | **chỉ vai quản trị** |
| Trả lời câu hỏi | "Đi hết chương trình thì con tôi được gì?" | "Dồn cố vấn vào nhà nào?" |
| Nội dung | 5 tầng năng lực học viên làm được | phân loại gia đình theo giá trị đồng hành |
| Ở đâu | `src/nha/cay-gia-tri.js` · `/nha/cay-gia-tri` | `src/cay-tien/` · `/cay-tien/*` |

### Cây giá trị — năm tầng, thứ gia đình nhìn thấy

Năm tầng không phải năm gói bán. Chúng là năm **thứ học viên làm được**, xếp
theo thứ tự phải có cái trước mới có cái sau, mỗi tầng kèm một **dấu hiệu gia
đình tự quan sát được ở nhà** — không cần tin lời hệ thống:

| Tầng | Trả lời | Dấu hiệu nhìn thấy ở nhà |
|---|---|---|
| 1. Nhìn thật | Con tôi đang thực sự đứng ở đâu? | Con nói được tên phần mình yếu, thay vì "con kém toán" |
| 2. Chắc gốc | Vá xong lỗ hổng cũ chưa? | Bài cũ làm lại sau hai tuần vẫn đúng |
| 3. Đi đều | Học thành nếp chưa? | Phụ huynh thôi phải nhắc; con tự mở bài |
| 4. Làm được đề thật | Vào phòng thi thì sao? | Làm hết đề trong giờ, không bỏ trắng phần cuối |
| 5. Tự đi | Rời hệ thống con có tự học được không? | Giải được bài ngoài chương trình và giải thích được vì sao |

Trang này **không hứa điểm số**. "Con sẽ đạt 1200 HSA" là câu không ai giữ
được, và một lời hứa như thế biến mọi tầng phía dưới thành đường tắt để lách.
Tầng hứa **năng lực**, và năng lực thì kiểm chứng được. Có mục kiểm quét đúng
trang này để bắt mọi con số thang điểm lọt vào.

Học viên đang ở tầng nào thì **chỉ ghi khi có dấu vết của tầng đó**. Thiếu dấu
vết thì trang nói "chưa đủ căn cứ" — đoán lên cho đẹp thì gia đình mừng một
hôm rồi mất lòng tin vào mọi con số còn lại.

### Cây Tiền — nội bộ điều hành

Hệ thống hoá từ tám chương của *Cây Tiền* (Lý Tiễn) thành **tám năng lực**:
chọn gia đình → hiểu gia đình → biết vì sao họ chọn mình → đo giá trị → phục
vụ theo mức → tăng giá trị → thành người đồng hành → nhân bản cách làm. Mỗi
năng lực phải trả lời thêm một câu sách không cần trả lời: *"dữ liệu nào trong
hệ thống này đáp được?"* — và **2/8 năng lực hiện ghi thẳng là CHƯA CÓ dữ
liệu** (lý do rời bỏ, sổ giới thiệu). Một khung mà mọi ô đều xanh là khung
chưa ai dùng thật.

**Trục giá trị KHÔNG phải doanh thu.** Hệ thống chưa có một đồng dữ liệu thanh
toán nào, nên bất kỳ con số tiền nào ở đây cũng là bịa. Trục đứng đo **giá trị
đồng hành** — gia đình đi *đều* tới đâu và học viên *tiến* tới đâu — và báo cáo
nói đúng như vậy mỗi lần in ra.

Hai trục cắt nhau thành bốn nhóm, cộng một nhóm chưa chấm:

| Nhóm | Là gì | Ưu tiên | Nhịp chạm |
|---|---|---|---|
| **Cây cần cứu** | giá trị cao, đang rời đi | **1** | trong 48 giờ |
| Cây tiền | đi đều và tiến thật | 2 | mỗi tuần |
| Cây chớm héo | nhịp thưa dần, tiến chững | 2 | mỗi tuần |
| Cây đang lớn | đều nhưng chậm, hoặc nhanh nhưng chưa đều | 3 | hai tuần |
| **Hạt mới** | chưa đủ dấu vết để chấm | 2 | mỗi tuần trong 6 tuần đầu |

Ba chỗ đáng chú ý trong thiết kế:

1. **Sàng lọc là bước ĐẦU TIÊN.** Gia đình dưới 20 bài / 5 buổi / 14 ngày
   không bị chấm, mà vào nhóm *Hạt mới* — và *Hạt mới* **không phải** nhóm giá
   trị thấp. Chấm một gia đình mới học ba hôm rồi xếp họ vào nhóm thấp là một
   quyết định sai ra bằng dữ liệu không đủ, và nó **tự ứng nghiệm**: ít chăm
   sóc thì họ đi thật.
2. **Cây cần cứu ưu tiên cao hơn Cây tiền.** Một giờ dành cho gia đình sắp rời
   đi giữ được nhiều hơn một giờ dành cho gia đình vốn đã đi đều.
3. **Nguy cơ rời bỏ lấy MAX, không lấy trung bình.** Trung bình ba dấu hiệu là
   sai theo hướng nguy hiểm nhất: một gia đình im lặng 20 ngày mà trước đó rất
   chăm sẽ ra 30/100 rủi ro, tức là được xếp "đang đi tốt" đúng lúc họ đã bỏ
   đi. Chăm chỉ tháng trước không làm cho việc biến mất tuần này bớt đáng lo.

Nhóm nói về **quan hệ**, không nói học viên giỏi hay kém: một học viên yếu đi
đều đặn nằm ở nhóm cao hơn một học viên giỏi đã bỏ ba tuần.

### Bức tường — được máy cưỡng chế, hai lớp

Luật: **không gia đình nào được đọc thấy mình bị xếp vào nhóm nào.** Lý do
không phải giữ bí mật kinh doanh — mà là một đứa trẻ biết hệ thống đang xếp
hạng gia đình mình theo giá trị sẽ học với một tâm thế khác, và cái khác ấy
phá đúng thứ hệ thống này tồn tại để tạo ra.

Luật nào chỉ nằm trong trí nhớ người viết mã thì sớm muộn cũng hỏng, nên luật
này thành **phép đo chạy trong bộ kiểm**:

* **lớp chữ** — `soiRoRi()` quét **177 trang công khai** và trang khu riêng với
  **cả bốn vai**, so với bảng 22 cụm cấm (có dấu, không dấu, mã nhóm, và cả
  chữ đã bị đổi thành `&quot;`). Một chữ lọt là trượt.
* **lớp cổng** — mọi cổng `/cay-tien/*` tự kiểm quyền ngay trong tay xử lý;
  khách gọi nhận **403**, kể cả giáo viên.

Phép soi chữ **không bắt được mọi thứ** — nói thẳng để không ai tin nhầm: một
gói JSON `{giaTri:{diem:0.7}}` lọt ra thì nó im lặng, vì cấm chữ "giaTri" sẽ
bắt oan cả *Cây giá trị* của khách. Đó chính là lý do phải có lớp cổng. Bỏ một
trong hai lớp là tường thủng, dù lớp còn lại vẫn xanh.

Có một mục kiểm riêng chứng minh **bức tường có răng**: thả 9 chuỗi rò rỉ thật
vào và đòi bắt hết, đồng thời đòi *không* bắt oan "Cây giá trị".

```bash
node bin/gita.js cay-tien                  # nội bộ điều hành
node bin/gita.js cay-tien --gia-dinh=HV01  # một gia đình
```

---

## 6b-ter. Lộ trình học — bản đồ toả tròn, đường đi thẳng

Mặt **trong** của hệ thống: chỗ học viên đi theo một trục thẳng tới đích đã
chọn. Mười phòng, mười trục, sáu mươi bước, mỗi trục bám **cấu trúc đề thật**
của hệ chuẩn tương ứng — đúng số câu, đúng số phút, đúng thang điểm, lấy từ
`src/matran/chuan.js` chứ không chép tay.

| Mái | Phòng | Nền |
|---|---|---|
| Chọn đích | Toán phổ thông · Tiếng Anh nền · HSA · TSA · SPT · SAT · IELTS · Olympiad | Luyện hằng ngày |

Quanh bản đồ là **mười một năng lực** (nền tảng toán, tư duy logic, xử lý số
liệu, tốc độ làm bài, đọc hiểu, từ vựng, ngữ pháp, phát âm, viết học thuật,
nhìn thật, nhịp luyện). Chúng **không bấm vào được** — chúng sáng dần theo tiến
độ của những phòng nuôi chúng.

### Ba luật, cả ba do máy cưỡng chế

**1. Một đường, không nhánh.** Động cơ không có hàm nhảy tới bước bất kỳ; chỉ
có `nop`, đẩy đúng một bước. Kiểm thử soi chính mặt API để chặn việc thêm hàm
nhảy về sau.

**2. Không cung cấp thông tin xem trước.** Bước chưa tới không có tên, không
có mô tả, chỉ còn số thứ tự. `soiRoRi` quét mọi thứ sắp gửi ra ngoài — cả dữ
liệu lẫn HTML đã dựng — tìm dấu vết bước sau. Thanh tra chạy phép soi đó **130
lần** mỗi lượt. Cho xem trước là mời người ta đọc lướt hết rồi bỏ.

**3. Qua được bước là phải nộp đầu ra.** Phần lớn đầu ra là **số**: số câu
đúng, số phút, số dạng đạt. Có số thì bước so sánh cuối mỗi trục mới có nghĩa.

### Bốn vai, một chỗ đứng

Học viên ở bước 3, trợ lý AI nói chuyện bước 5, cố vấn hỏi bước 1, giáo viên
AI dạy bước 6 — bốn người đều nhiệt tình và mỗi người kéo một hướng. Nên cả
bốn bản giao việc nhận vào **đúng một thứ**: kết quả của `truc.hienThi`, đã
cắt sẵn phần chưa tới. Bốn vai không thể lệch nhau vì không có chỗ để lệch.
Hai vai là mô hình sinh văn bản còn mang **hàng rào**: chỉ bàn bước hiện tại.

---

## 6c. Biên soạn có ràng buộc — không ghép số ngẫu nhiên

**150 bản thiết kế bài** phủ **104 trên 104 dạng**, mỗi bản là một *quyết định
sư phạm*: dạy gì, nhắm đúng bẫy nào, bối cảnh ra sao, lời giải mấy bước. Không
bản nào trùng ý đồ. Trải từ lớp 1 đến lớp 12, thêm nhóm liên môn và nhóm
tiếng Anh.

**Bốn dạng từng để trống** — đọc hiểu ý chính, đọc hiểu suy luận, viết luận,
nói theo chủ đề — nay đã có học liệu, nhưng **không phải bằng cách ghép máy**.
Ghép câu bằng máy ra một đoạn không có ý chính nào để mà hỏi. Văn bản của bốn
dạng này **viết tay** (`src/lang/en-corpus.js`: 18 + 18 đoạn đọc hiểu, 14 đề
luận, 14 thẻ nói), còn máy giữ phần nó làm được và làm nghiêm: đo quan hệ giữa
từng phương án với văn bản để bảo đảm đề chỉ có **đúng một** phương án hợp lệ.
Chi tiết ở mục 6e.

Bộ sinh phải **dùng hết ý đồ trước, mới đến tham số**. Không gian tham số sinh
từ **ràng buộc toán học** (Δ chính phương, ƯCLN bằng 1, số không chính phương…),
không có `random`. Cùng seed cho đúng cùng bộ bài, từng chữ một.

Chống trùng ba lớp chặn ngay tại cửa kho. Chữ ký khung bài gộp **cả dấu hệ số**
— đổi dấu vẫn là cùng một khuôn — nhưng tách theo **hình dạng đáp án**, vì
"vô nghiệm" và "hai nghiệm" đòi kết luận khác hẳn nhau.

**546 bài** phủ **100%** (417/417) ô (dạng × tầng) có bản thiết kế, đủ mặt cả
năm bậc sao: 42 bài 1★ · 105 bài 2★ · 173 bài 3★ · 194 bài 4★ · 32 bài 5★.

**Cổng phát hành chặn theo CHỨNG MINH, không chặn theo bậc sao.** Trước đây
cổng chặn mọi bài từ 4★ trở lên để chờ người duyệt; hệ quả đo được là kho
không có lấy một bài 4★ hay 5★ nào ở trạng thái phát hành, nên **mọi đề đều
thiếu đúng dải dùng để phân loại học viên giỏi** — đề hứa một đằng, dựng ra
một nẻo. Chặn theo bậc sao là chặn nhầm trục: phần khó của bài khó chính là
phần máy kiểm chắc nhất (máy tự giải lại rồi thay nghiệm ngược). Nay cổng
chặn theo điều máy thật sự biết: **bài máy tự giải lại và khớp thì phát hành;
bài máy không kiểm được thì dừng chờ người thật, bất kể dễ hay khó.**

### Lời giải nói cả VIỆC LÀM lẫn CĂN CỨ

Mỗi bước giải gồm hai phần tách bạch:

* **việc** — thao tác: làm gì, ra số nào. Đây là phần học viên chép theo.
* **căn cứ** — định lý, công thức hay điều kiện nào cho phép làm thao tác đó.
  Đây là phần học viên mang sang bài khác được.

Lời giải chỉ có thao tác thì học viên chép được nhưng gặp bài khác một chút là
bế tắc, vì thứ chuyển được sang bài mới là căn cứ chứ không phải thao tác.

**648/648 bước của cả 150 bản thiết kế đều nêu căn cứ** — có mục thanh tra đo
tỉ lệ này, và `gita kho-de` in ra con số ấy.

Căn cứ đi theo bài tới mọi nơi bài xuất hiện: trang công khai, phiếu tải về,
bảng điều khiển giáo viên, bàn học viên, và bộ đọc thành tiếng (đọc thao tác
trước, rồi đọc căn cứ chậm hơn một nhịp để người nghe kịp nối hai phần).
Bộ soát tiếng Việt và bộ chấm ký hiệu cũng soi phần căn cứ y như phần thao
tác — miễn trừ nó thì nửa lời giải không ai soát.

Mọi nơi đọc lời giải đều đi qua `src/content/loi-giai.js`, nên bản thiết kế cũ
khai bước giải bằng chuỗi vẫn chạy: bước ấy có thao tác mà chưa có căn cứ.

Mỗi bài đều đi qua **ba chốt** trước khi vào kho: máy giải lại độc lập · rubric
5 chiều (không đủ 4,5/5 thì chặn) · chống trùng ba lớp. Bài toán chốt bằng cách
thay số; bài tiếng Anh chốt bằng bộ đọc lại ngữ pháp (xem mục 6e).

Với lớp 11–12, bộ kiểm chứng chỉ biết TÍNH GIÁ TRỊ chứ không biết đạo hàm hay
nguyên hàm, nên mỗi bài được gắn một phép kiểm **độc lập với đường lối giải**:
đạo hàm kiểm bằng **đồng nhất thức sai phân chính xác**
(f(x+h) − f(x) = h·f′(x) + a²h² là đẳng thức đúng tuyệt đối, không phải xấp xỉ);
nguyên hàm và tích phân kiểm bằng **Simpson** (chính xác tuyệt đối với đa thức
bậc ≤ 3) hoặc **cầu phương Gauss–Legendre 5 điểm** cho hàm mũ, lôgarit.

## 6d. Đo và dạy

**Chẩn đoán đầu vào**: thang bậc thích ứng theo mô hình Rasch. Khởi điểm suy từ
lớp đang học — không bắt học sinh lớp 9 làm lại bài lớp 3. Không xếp cao hơn
tầng cao nhất thật sự làm đúng. Báo sai số chuẩn và mức tin cậy thật.

**Phiên luyện tập**: chọn bài theo 5 nguồn ưu tiên (nợ ôn → lỗ hổng nền → vùng
phát triển gần → dạng mới → duy trì), mỗi bài đều nói rõ **vì sao được chọn**.
Có trần tỉ lệ từng nguồn: học viên có lỗ hổng vẫn phải được thấy cái mới.

Thích ứng ngay trong phiên: sai 2 bài liên tiếp thì **lùi ba nấc** — bài dễ hơn
cùng dạng → tầng thấp hơn → **dạng tiên quyết**. Không gỡ được thì nói thẳng là
kho đang hổng, không im lặng.

Chấm tự động bằng bộ kiểm tra toán học: viết khác cách vẫn đúng; sai thì chỉ ra
thiếu hay thừa nghiệm; không chấm máy được thì **chờ giáo viên, không đoán**.

**Thi cuối tầng — mỗi lượt 20 câu.** Đề phủ được nhiều dạng nhất mà cỡ đề cho
phép, trộn độ khó theo **ma trận riêng của từng chặng năng lực**, loại bài vừa
luyện 30 ngày, có giới hạn thời gian, chấm theo trọng số bậc sao. Thi trượt phải
chờ 3 ngày mới thi lại.

Dùng chung MỘT ma trận cho cả mười tầng là nói dối ở cả hai đầu: nó đòi 10% câu
"Thách thức" cho học viên lớp một, và vẫn giữ 15% câu "Nhận biết" trong đề tuyển
chọn. Nên ma trận chia theo bốn chặng, trọng tâm dịch dần từ nhận biết sang
thách thức:

| Chặng | Tầng | ★ | ★★ | ★★★ | ★★★★ | ★★★★★ |
|---|---|---|---|---|---|---|
| Khởi động và nền tảng | 0–1 | 7 câu | 8 câu | 5 câu | — | — |
| Vững và thành thạo | 2–3 | 4 câu | 6 câu | 6 câu | 4 câu | — |
| Nâng cao, giỏi, xuất sắc | 4–6 | 2 câu | 4 câu | 6 câu | 6 câu | 2 câu |
| Chuyên, tinh hoa, đỉnh cao | 7–9 | 1 câu | 3 câu | 5 câu | 7 câu | 4 câu |

Con số **20** không tuỳ tiện: đó là số nhỏ nhất chia chẵn cho mọi tỉ lệ trong
bảng, nên đề dựng ra có **đúng** số câu từng bậc như bảng đã in. Lấy 12 câu như
trước thì 5% thành 0,6 câu và cấu trúc đề lệch khỏi bảng công bố. Có kiểm thử
dựng đề ở cả mười tầng và bắt lệch dù chỉ một câu.

**Lên tầng phải đủ cả 7 tiêu chí.** Điểm thi 100% cũng không bù được tiêu chí khác.

**Lộ trình 90 ngày** sinh từ chân dung: 13 tuần, 3 chặng, mốc kiểm tuần 4–8–13.
Khối lượng buổi học lấy từ **ngưỡng chú ý đo được**. Cắt theo sức chứa thật
(≥12 bài mỗi dạng); phần dư nói thẳng là để sang chu kỳ sau. Rà soát hằng tuần
và chỉnh phần còn lại, **mỗi lần chỉnh đều lưu lý do**.

## 6e. Phát âm tiếng Anh — phiên âm IPA, chẩn đoán lỗi, và giọng đọc

Tiếng Việt và tiếng Anh khác nhau ở chỗ gốc, nên không dùng chung một bộ máy
phát âm được:

| | tiếng Việt | tiếng Anh |
|---|---|---|
| âm tiết | đơn âm tiết | đa âm tiết, có **trọng âm** |
| cao độ | do **thanh điệu**, sai là đổi nghĩa | do trọng âm và ngữ điệu câu |
| âm cuối | 8 âm, không bật hơi | bắt buộc bật rõ, có cụm 3–4 phụ âm |
| chữ và âm | gần như khớp một–một | cách nhau rất xa: `ough` đọc được sáu kiểu |

**47 âm vị** (12 nguyên âm đơn + 2 nguyên âm không căng + 9 nguyên âm đôi +
24 phụ âm), mỗi âm có mô tả khẩu hình bằng tiếng Việt và **tần số formant
F1/F2/F3** — nhờ vậy bộ tổng hợp formant sẵn có đọc được tiếng Anh thật, không
phải phát lại tệp thu sẵn.

**Hai đường phiên âm, và luôn nói rõ đang đi đường nào.** Chữ viết tiếng Anh
không suy ra âm được, nên:

1. **Từ điển** — 464 từ chép tay: toàn bộ từ chức năng (chiếm phần lớn số lượt
   xuất hiện trong văn bản thật), từ bất quy tắc, và từ vựng phổ thông có chữ
   viết đánh lừa (*colonel, receipt, Wednesday*).
2. **Hình thái** — tách hậu tố đều rồi tra gốc: `teachers` ← `teacher`,
   `stopped` ← `stop`, `carries` ← `carry`.
3. **147 luật chữ–âm** có xét **ngữ cảnh trái và phải** của từng chữ cái, dùng
   khi hai đường trên không có.

Mỗi kết quả kèm trường `nguon`: `từ điển` · `từ điển + hậu tố "-es"` · `luật`.
Không bao giờ đoán mà giấu.

Những chỗ dễ sai đã xử lý đúng, có test chặn:

* **đuôi `-s` quyết định theo ÂM, không theo chữ** — `makes` là /meɪks/ chứ
  không phải /meɪkz/, vì chữ `e` đứng trước là chữ câm, âm thật đứng trước là
  /k/;
* **phụ âm tự làm hạt nhân âm tiết** — `student` /ˈstjuː.dn̩t/ có **hai** âm
  tiết dù chỉ một nguyên âm; `little` hai, `beautiful` ba, `films` một;
* **trọng âm** lấy từ từ điển thì giữ nguyên cả trọng âm phụ (`ˌedʒuˈkeɪʃn`),
  không có thì suy theo hậu tố (`-tion`, `-ity` kéo trọng âm về trước; `-ee`,
  `-ese` tự nhận);
* **dạng yếu** của từ chức năng khi nói nhanh (`can` → /kən/), nhưng **không
  áp dụng ở đầu và cuối câu** — bỏ qua luật này thì máy đọc ra một chuỗi /ə/
  không ai nghe được;
* **nối âm** phụ âm cuối sang nguyên âm đầu: *sheep_on*.

**Cặp tối thiểu được TÌM RA, không chép sẵn.** Hệ thống so từng dãy âm trong
từ điển và giữ lại các cặp khác nhau **đúng một âm cùng loại**: **506 cặp**
thuộc **131 thế đối lập**. Từ điển lớn thêm thì kho cặp tự lớn theo, và không
bao giờ có cặp sai.

**Chẩn đoán lỗi người Việt là DỰ BÁO, không phải chấm bằng tai.** Người Việt
sai tiếng Anh theo những kiểu rất đều vì hệ âm tiếng Việt thiếu đúng những âm
đó, nên đọc cấu trúc âm của từ là biết trước sẽ sai ở đâu: âm khó (/θ/, /ð/,
/z/, /ʃ/, /dʒ/, /r/, thế đối lập dài–ngắn /iː/–/ɪ/), **âm cuối bị nuốt**,
**cụm phụ âm bị chèn "ơ" vào giữa**, và **trọng âm đọc đều tăm tắp**. Mỗi
cảnh báo kèm cách sửa cụ thể, không kèm lời khen sáo.

```
gita phat-am "sheep"

  sheep   /ˈʃiːp/   1 âm tiết · nhấn âm tiết 1 · nguồn: từ điển

  NGƯỜI VIỆT HAY SAI Ở ĐÂY  độ khó 4/5
    [âm cuối] /p/ → thường đọc thành bị nuốt mất
      Âm cuối /p/ phải nghe thấy được. Tiếng Việt không có âm cuối này nên
      rất dễ bỏ quên — đọc chậm lại và bật rõ ở cuối từ.
    [âm khó] /iː/ → thường đọc thành /i/ ngắn — mất hẳn thế đối lập dài–ngắn
      Kéo dài gần gấp đôi và căng môi. Luyện theo cặp: sheep – ship.
```

`gita cap-am "iː" "ɪ"` dựng nguyên một tiết dạy phân biệt hai âm: mô tả khẩu
hình · vì sao người Việt khó · cách đặt lưỡi đặt môi · cặp luyện · câu luyện
có phiên âm. `gita doc-anh "..."` đọc ra tệp WAV, có ngữ điệu câu hỏi và câu
trần thuật khác nhau; `--giong=my` đổi sang giọng Mỹ (`ɒ`→`ɑː`, `əʊ`→`oʊ`,
đọc `/r/` cuối từ).

### Học liệu tiếng Anh được KIỂM LẠI thật

Bài toán chốt bằng cách thay số vào biểu thức. Bài ngữ pháp không có số để
thay, nên nếu không làm gì thì mọi bài tiếng Anh sẽ vào kho mà **không ai giải
lại**. Hệ thống làm theo đúng nguyên tắc đã dùng cho môn toán — **hai đường
độc lập phải gặp nhau**:

* **đường sinh** GHÉP câu đích từ các mảnh đã biết (chủ ngữ, động từ đã chia,
  tân ngữ, trạng ngữ);
* **đường kiểm** ĐỌC LẠI câu nguồn như một người học đọc đề: tách câu thành
  thành phần, tra ngược hình thái về động từ gốc và thì, rồi tự biến đổi.

Lệch nhau là bài bị chặn ngay tại cửa kho. Mức độ độc lập được nói thẳng trong
mã: bị động · chia thì · tường thuật · điều kiện là **độc lập thật**; mệnh đề
quan hệ **độc lập một phần** (dùng chung danh sách cụm danh từ); kết hợp từ
không kiểm lời giải mà **kiểm chất lượng đề** — bảo đảm đáp án đúng có trong
bảng và **mọi phương án nhiễu đều KHÔNG đi được với cụm đó**, chính là chỗ đề
trắc nghiệm hay hỏng vì có hai đáp án cùng đúng.

Bộ soát tiếng Việt cũng được dạy **bỏ qua phần không phải tiếng Việt** — câu
tiếng Anh trong ngoặc kép, khối `[NGHE]…[/NGHE]`, và **ngữ liệu mà chính bài
khai báo ở `check.english`** (đoạn đọc hiểu, đề luận, thẻ nói) là dữ liệu của
bài, không phải lời văn của người soạn. Đem thước tiếng Việt đo chúng thì câu
tiếng Anh nào cũng bị báo "thiếu dấu", còn lỗi thật trong phần lời Việt thì
chìm nghỉm. Cắt theo chuỗi bài **khai báo**, chứ không cắt theo "trông giống
tiếng Anh" — đoán như thế thì một câu tiếng Việt gõ thiếu dấu cũng lọt.
Sau khi sửa: **236/236 bài đạt, 0 lỗi chặn**.

### Bốn dạng không ghép được bằng máy — và cách hệ thống vẫn kiểm được chúng

Đọc hiểu ý chính, đọc hiểu suy luận, viết luận và nói theo chủ đề **không ghép
được bằng máy**: ghép câu bằng máy ra một đoạn không có ý chính nào để mà hỏi.
Trước đây bốn dạng này để trống, và bản đồ ma trận ghi thẳng chỗ trống đó.

Nay văn bản của bốn dạng được **viết tay**, nằm ở `src/lang/en-corpus.js`:
18 đoạn đọc hiểu ý chính (lớp 4–9), 18 đoạn đọc hiểu suy luận (lớp 4–9),
14 đề luận IELTS Task 2 (lớp 10–12) và 14 thẻ nói IELTS Part 2 (lớp 9–12).
Bốn bản thiết kế cuối `src/content/blueprints/tieng-anh.js` chỉ dựng đề quanh
văn bản có sẵn — **64 bài, cả 64 đều qua máy kiểm**.

Đề nhiều dòng **giữ nguyên chỗ xuống dòng** ở mọi nơi bài xuất hiện. Nghe thì
nhỏ, nhưng nhồi đoạn văn và bốn phương án vào một khối thì trên màn hình chúng
dính liền nhau — *"…saves a tonne of material. A. The sorting and rinsing…"* —
và người học không tách được đâu là bài đâu là lựa chọn. Trang công khai tách
mỗi dòng thành một khối và thụt các dòng A–D; phiếu tải về dùng ngắt dòng cứng
của Markdown, vì một lần xuống dòng thường bị Markdown nuốt thành khoảng trắng.

Máy không chấm được bài luận thành band — chấm band cần người, và hệ thống nói
thẳng điều đó. Cái máy kiểm được là **chất lượng đề**, đo bằng cách đọc lại
chính đoạn văn, tự tách câu, tự đếm:

| Dạng | Điều máy đo trên phương án đúng | Điều máy đo trên từng nhiễu |
|---|---|---|
| Ý chính | trải trên **≥ 3 câu**, bám bài **≥ 70%** số từ nội dung, **không câu nào** của đoạn chứa đủ nó | *chi tiết* nằm gọn trong một câu · *ngoài bài* bám **≤ 50%** · *quá rộng* có từ phạm vi (every, always) mà đoạn không hề nói |
| Suy luận | bám bài ≥ 70%, nối **≥ 2 câu**, **chưa câu nào nêu thẳng** | *đã nêu* có một câu chứa đủ nó · *ngoài bài* bám ≤ 50% · *trái bài* có từ phủ định vượt bài |
| Viết luận | dạng đề **đọc ra từ chữ của đề** (`to what extent do you agree` ≠ `discuss both views`), câu chủ đề đủ dấu hiệu của đúng dạng đó | không nhiễu nào đủ dấu hiệu của dạng ấy |
| Nói theo chủ đề | thẻ có **đúng 4 gạch đầu dòng**, dàn ý bám chủ đề và **chạm đủ cả 4** | *bỏ sót* chạm ≤ 3 gạch · *lạc đề* không nhắc chủ đề của thẻ |

Phép đo này bắt được đúng cái làm hỏng một đề trắc nghiệm: **hai phương án
cùng hợp lệ**. Trong lúc biên soạn, 64 đơn vị văn bản đầu tiên có **17 đơn vị
bị chính phép đo này đánh trượt** — và cách xử lý là **sửa văn bản**, không
phải hạ ngưỡng đo.

## 6f. Báo cáo

Ba cấp — học viên, lớp, hệ thống. Mọi tỉ lệ đi kèm **cỡ mẫu**; dưới 10 lượt thì
nói thẳng là chưa đủ căn cứ. Báo cáo lớp tách em có học và em chưa học, đo độ
phân hoá, và chỉ ra **dạng cả lớp cùng yếu** để dạy lại chung. Báo cáo hệ thống
nêu ô kho còn trống, bài chờ duyệt, bài lệch độ khó so với dữ liệu thật — mỗi
phát hiện kèm việc cần làm. Xuất CSV cho cả ba cấp.

## 6g. Giọng nói giảng viên

500 agent là **500 giảng viên, 500 giọng**. Giọng được suy ra tất định từ mã
agent — không lưu bảng, không ngẫu nhiên — nên dựng lại hệ thống bao nhiêu lần,
thầy dạy Toán vẫn đúng giọng đó. Đo thật: 500/500 chữ ký giọng khác nhau, nam
249 / nữ 251, cao độ 96–240 Hz, mỗi khối một chất giọng riêng.

**Không dùng cổng dịch không chính thức.** Bộ tổng hợp mặc định là
**formant tiếng Việt tự dựng**, chạy ngay trong tiến trình: không cần mạng,
không cần khoá, không giới hạn ký tự, kết quả tất định. Nó dựng sóng từ đầu —
xung thanh môn Rosenberg, ba cộng hưởng formant trượt theo nguyên âm, nhiễu
trắng cho âm xát, bật hơi cho âm tắc — và **đi đúng sáu thanh điệu**: kiểm
chứng bằng đo cao độ trên sóng thật, thanh huyền hạ giọng, thanh sắc lên giọng,
thanh nặng ngắn hơn 20% và tắt sớm. Muốn giọng hay hơn thì cắm thêm nhà cung
cấp: piper (nơ-ron, chạy cục bộ), Google Cloud TTS, Azure, ElevenLabs. Thiếu
khoá thì nhà cung cấp đó báo "chưa sẵn sàng" và hệ thống lùi về bộ nội bộ chứ
không trả lỗi cho học viên.

Trước khi đọc, văn bản đi qua bộ **đọc tiếng Việt cho toán học**: `15` →
"mười lăm" (không phải "mười năm"), `21` → "hai mươi mốt", `x^{2}` → "ích bình
phương", `\dfrac{-b+\sqrt{b^{2}-4ac}}{2a}` → "âm b cộng căn bậc hai của b bình
phương trừ bốn ac trên hai a", `ƯCLN(12,18)` → "ước chung lớn nhất của mười hai
và mười tám", `VNĐ` → "đồng", `ĐKXĐ` → "điều kiện xác định".

Một bài không bị đọc thành một hơi. Hệ thống dựng **kịch bản giảng bài** gồm
các vai: công bố giọng AI → tên dạng → đọc đề → **nghỉ 2,5 giây cho em nghĩ** →
gợi ý → từng bước giải → cảnh báo bẫy → đáp án → động viên. Tầng thấp đọc chậm
hơn và nghe hết gợi ý; tầng cao đọc nhanh hơn, chỉ nghe gợi ý đầu. Chế độ
"đọc đề" tuyệt đối không lộ đáp án.

**Pháp lý — Luật An ninh mạng 116/2025/QH15 được thi hành bằng mã, không phải
bằng danh sách gạch đầu dòng:**

| # | Yêu cầu | Phần mã đảm nhiệm |
|---|---------|-------------------|
| 1 | Đồng ý có văn bản | `ConsentRegistry.grant()` chặn khi thiếu mã hồ sơ |
| 2 | Đánh dấu nội dung AI | dấu nghe được ở đầu tệp **và** dấu chìm nhúng trong bit thấp nhất của sóng, lặp suốt tệp, đọc lại được |
| 3 | Nhật ký không sửa được | chuỗi băm nối tiếp, sửa một mắt xích là lộ vị trí |
| 4 | Thời hạn lưu trữ | `sweepRetention()` — mặc định 180 ngày |
| 5 | Xác minh tuổi | dưới 18 bắt buộc có phiếu của người giám hộ |
| 6 | Công bố công khai | bản công bố đính kèm **mọi** phản hồi giọng |
| 7 | Quyền được xoá | `forget()` xoá đồng ý, gỡ định danh khỏi nhật ký mà không làm đứt chuỗi băm, và cấp biên bản |

Chưa có phiếu đồng ý thì học viên có định danh **không phát được giọng** — trả
403 kèm hướng dẫn, không phải 200 kèm âm thanh.

**Đo thật trên một lõi** (Node 22, không mạng): dựng 4,6 giây tiếng mất 131ms —
nhanh gấp **35 lần thời gian thực**; 50 câu khác nhau mất 1,97 giây, tức
**~25 câu/giây/lõi**; câu lặp lại lấy từ bộ đệm, tỉ lệ trúng 0,98 và 50 câu chỉ
mất 32ms. Bộ tổng hợp chạy đồng bộ nên với lớp đông nên bật thêm lõi hoặc đọc
trước bài của buổi học — con số trên là để tính chứ không phải để khoe.

## 6h. Ba tuyến chuyên biệt · hậu kỳ phòng thu · giọng hát

Nói và hát không phải một việc. Khi nói, cao độ do **thanh điệu** quyết định;
khi hát, cao độ do **nốt nhạc** quyết định còn thanh điệu chỉ còn là gợi ý ở
đầu mỗi nốt. Nhồi hai việc đó vào một hàm thì hỏng cả hai, nên hệ thống tách
thành ba tuyến.

**Tuyến 1 — Giọng nói tiếng Việt.** Chọn bộ tổng hợp theo *mục đích*, không
theo sở thích:

| Mục đích | Thứ tự ưu tiên | Hậu kỳ |
|---|---|---|
| Phát trực tiếp | ZeroTTS → piper → nội bộ | tắt (không chờ được cả câu) |
| Giảng bài | VieNeu-TTS → ZeroTTS → piper → nội bộ | `giang-bai`, −16 LUFS |
| Bài dài · podcast | VieNeu-TTS → G-OmniVoice → nội bộ | `phat-thanh`, −14 LUFS |
| Đọc chính xác (đề thi) | G-OmniVoice → ZeroTTS → nội bộ | `giang-bai` |
| Thông báo hệ thống | ZeroTTS → nội bộ | `phat-thanh` |

Ba bộ nơ-ron tiếng Việt (**ZeroTTS** WER 1,03%, **VieNeu-TTS v3** 48 kHz có
dấu cảm xúc, **G-OmniVoice** WER 2,59% / SIM 0,890) có bộ chuyển tiếp thật,
gọi được cả qua máy chủ HTTP cục bộ lẫn chạy thẳng Python. Chưa cài thì mỗi bộ
nói rõ **thiếu gì và đặt biến nào**, rồi hệ thống lùi về bộ formant nội bộ —
không có đường nào dẫn tới "không có tiếng".

**Hậu kỳ chuẩn phòng thu — viết bằng JavaScript, không cần ffmpeg.** Bản
thiết kế ban đầu dùng chuỗi filter của FFmpeg; nhưng hậu kỳ mà phải gọi tiến
trình ngoài thì thiếu một gói là im tiếng, và cả dự án có luật không phụ thuộc
chạy thật. Nên chuỗi được viết lại bằng DSP thật:

```
lọc rumble → EQ chuông (biquad RBJ) → de-esser dò dải 6,5 kHz
→ nén song song → bão hoà hài (bậc 2 + bậc 3) → hồi âm Schroeder
→ chuẩn độ vang ITU-R BS.1770-4 → giới hạn đỉnh có nhìn trước
```

Năm cài đặt sẵn: `giang-bai` · `phat-thanh` · `hat-pop` · `hat-ballad` ·
`moc`. **Đo thật:** mọi nguồn đều về đúng độ vang đích, sai số lớn nhất
**0,51 dB**, không bản nào vượt trần đỉnh. Nén song song giữ được chênh lệch
to–nhỏ (không san phẳng như nén thẳng), de-esser hạ dải xì mà giữ nguyên thân
giọng, bão hoà sinh hài bậc 2 từ 0,0006 lên **0,0517** và bậc 3 từ 0 lên
**0,0623** với lệch một chiều 3,5·10⁻¹⁰.

**Tuyến 2 — Hát.** Nhận lời + giai điệu, trả về bản hát đã hậu kỳ.

- Giai điệu vào bằng **ký âm chữ** dễ gõ (`sol4:1 la4:1 si4:1 đô5:2`, tên nốt
  Việt hay Anh đều được) hoặc bằng **tệp MIDI thật** — trình đọc SMF viết
  thẳng bằng JavaScript (MThd/MTrk, số biến độ dài, đổi nhịp độ giữa bài),
  kèm trình ghi để xuất ngược ra `.mid`. Vòng ghi–đọc khớp **từng nốt**.
- Lời ghép vào nốt có kiểm tra: thiếu lời, thừa lời, dấu `-` là **luyến**
  (ngân tiếp âm tiết trước). Nhiều bè thì lấy bè cao nhất. Vượt quãng giọng
  thì **dịch theo quãng tám**, rộng quá quãng thì báo thẳng chứ không hát ép.
- Bộ hát nội bộ dựng tiếng bằng chính bộ formant của phần nói, thêm: **rung
  giọng** vào sau ~25–45% nốt và mạnh dần, **luyến** giữa hai nốt, **kéo dài
  nguyên âm** (chỉ nguyên âm — kéo cả phụ âm là nghe như băng chạy chậm),
  **ngân âm cuối mũi**, **lấy hơi** ở chỗ nghỉ. Năm cách hát: pop · ballad ·
  dân ca · thiếu nhi · đọc rap.
- Riêng cho tiếng Việt: **nét thanh điệu giữ ở đầu nốt rồi tan dần về đúng cao
  độ**. Hát bỏ hết thanh thì "má" thành "ma"; giữ nguyên thanh thì thanh huyền
  kéo cả nốt xuống gần một cung, nghe phô. **Đo thật trên sóng:** lệch lớn
  nhất **14 cent** (một phần bảy nửa cung).
- **SoulX-Singer** và **OpenUtau + DiffSinger** cắm thêm được. SoulX-Singer tự
  khai là **chưa có tiếng Việt** và từ chối hát tiếng Việt thay vì hát bừa.

**Tuyến 3 — Đổi giọng hát (SVC).** Đây là phần nguy hiểm nhất về pháp lý vì
chạm vào giọng hai con người thật, nên cổng chặn **hai đầu**: người hát gốc
phải có phiếu `VOICE_RECORD`, chủ giọng đích phải có phiếu `VOICE_CLONE` kèm
mã hồ sơ văn bản. Thiếu một bên là dừng, không có đường vòng. Và **cố ý không
có bản thay thế nội bộ**: chưa cài **So-VITS-SVC** thì hệ thống nói thẳng là
chưa làm được, thay vì tạo ra thứ dễ bị lạm dụng nhất bằng mã tự chế.

**Hiệu năng đo thật:** nói kèm hậu kỳ nhanh gấp **17,5×** thời gian thực; hát
kèm hậu kỳ nhanh gấp **32,7×** (dựng 12 giây tiếng hát mất 367ms).

## 7a. Làm phim — GITAfilm trong Nexus

Kịch bản → phân cảnh → **bản nháp xem được** → dựng video thật. Một luật chi
phối toàn bộ: **không tiêu tiền trước khi duyệt được nhịp phim**, nên đường đi
chia ba nấc, nấc sau chỉ mở khi nấc trước đạt.

**Nấc 1 — tiền kỳ, $0.** Kịch bản được đọc **bằng luật, không bằng mô hình**:
tiêu đề cảnh `CẢNH 1 - INT. LỚP HỌC - CHIỀU MUỘN`, dòng nhân vật viết hoa, lời
thoại, đoạn hành động. Ba lý do không giao việc này cho LLM: không có khoá thì
không đọc được kịch bản ai cũng đọc được bằng mắt; mô hình sinh JSON không ổn
định nên hai lần chạy ra hai bản phân cảnh; và mô hình lặng lẽ bịa — thêm nhân
vật, sửa lời thoại cho "mượt". LLM chỉ dùng để làm giàu, thiếu nó phim vẫn dựng.

**Khoá ngoại hình** chặn lỗi lớn nhất của phim AI — sang cảnh sau nhân vật
thành người khác. Mỗi nhân vật có **một** chuỗi mô tả duy nhất, rút từ chính
kịch bản (`tóc bạc` → `silver grey hair`, `kính gọng vàng` → `gold-rimmed
glasses`), chèn **nguyên văn** vào mọi prompt có nhân vật đó. Prompt nào thiếu
khoá bị bắt ngay ở khâu soát, không đợi xem phim mới biết.

**Thời lượng cảnh quay được ĐO, không ước theo số chữ.** Hệ thống có sẵn bộ
phiên âm tiếng Việt nên lời thoại được tách âm tiết, cộng trường độ từng âm vị,
cộng chỗ ngắt ở dấu câu — rồi cảnh quay được đặt theo con số đo được cộng
khoảng lấy đà và khoảng lặng. Nhờ vậy lỗi **"thoại dài hơn cảnh"** bị bắt
trước khi gọi API tốn tiền.

**Soát chất lượng** chạy trước mọi lần tiêu tiền, mỗi phép soát đều đo được:
thoại tràn khỏi cảnh · prompt mất khoá ngoại hình · nhân vật tả sơ sài · cảnh
vượt thời lượng mô hình đã chọn · hai cảnh trùng prompt (phim lặp hình) · nhịp
phim · vượt ngân sách. Kịch bản mẫu đạt **100/100**; ép cảnh thoại còn 2,5s thì
chặn ngay 5 lỗi.

**Nấc 2 — bản nháp, vẫn $0.** Ba thứ dựng được ngay, không cần khoá nào:

- **Đường tiếng thật** — toàn bộ lời thoại đọc bằng bộ giọng tiếng Việt của hệ
  thống, mỗi nhân vật một giọng riêng theo giới, đặt đúng mốc thời gian từng
  cảnh, chuẩn độ vang cho **cả phim** (không phải từng câu, nếu không thì câu
  to câu nhỏ như ghép từ nhiều buổi thu).
- **Bảng phân cảnh** — SVG vẽ tại chỗ: đúng cỡ cảnh, vị trí nhân vật theo quy
  tắc một phần ba, tông màu theo thời điểm trong ngày, lời thoại, thời lượng.
- **Phim nháp (animatic)** — một trang HTML tự chạy: khung phân cảnh đổi theo
  đúng mốc thời gian, đồng bộ với đường tiếng. Mở bằng trình duyệt là duyệt
  được nhịp phim trước khi trả một đồng nào.

**Nấc 3 — dựng thật.** Sáu mô hình qua fal.ai (Kling 3.0 · Veo 3.1 · Runway
Gen-4.5 · Sora 2 · Wan 2.5 · Pika 2.5), chọn theo nhu cầu thật của từng cảnh
và **nói rõ vì sao chọn**: có nhân vật quen → mô hình giữ được khuôn mặt; có
lời thoại → mô hình dựng được khẩu hình; cảnh thiết lập → mô hình có chất điện
ảnh; cảnh dài hơn 15 giây → chỉ Sora 2 làm được. Ba hướng ưu tiên được **so
sánh giá trước** để chọn có căn cứ (kịch bản mẫu: $10,16 / $5,87 / $5,87).

Chốt chặn nằm trong mã, không nằm trong hướng dẫn: **soát phải đạt**, phải
**dưới trần chi tiêu**, và phải **xác nhận tay**. Chưa có khoá thì trả 503 kèm
lối ra ("vẫn dựng được bản nháp") chứ không im lặng.

Khi đã có clip, hệ thống sinh sẵn **EDL** (mở bằng phần mềm dựng), tệp concat
và **kịch bản ffmpeg** có kiểm tra thiếu clip — chạy một lệnh ra phim.

## 7b. Xưởng ảnh — GITA Photo Studio trong Nexus

Mô tả bằng tiếng Việt → prompt đầy đủ thông số máy ảnh → khung dựng bố cục →
tám tầng hậu kỳ → soát và xuất. Ba cửa vào, xếp theo "làm được ngay" giảm dần:

**Cửa 1 — chuẩn bị ($0, không cần mạng).** Prompt được ghép bằng luật và
**liệt kê rõ hệ thống đã thêm gì** (bố cục, ống kính, ánh sáng, chất màu) —
người dùng phải biết ảnh mình nhận được đã bị thêm những gì. Bố cục chọn
**tất định theo băm của chính mô tả**, không bốc thăm: cùng một yêu cầu luôn
ra cùng một bố cục nên bộ ảnh dựng lại được; muốn khác thì đổi số hiệu tấm.
Mỗi bố cục có **toạ độ chủ thể thật**, nên khung dựng vẽ ra được — bố cục nói
bằng lời thì không kiểm được, nói bằng toạ độ thì kiểm được.

**Cửa 2 — hậu kỳ ($0, không cần mạng, không cần khoá, không cần ffmpeg).**
Đây là phần làm việc thật sự của xưởng, chạy trên ảnh có sẵn:

```
đọc ảnh → phóng to Lanczos-3 nhiều lần → chất màu máy ảnh
→ đường tông giả lập phim → làm sắc có ngưỡng → hạt phim và tối góc
→ soát bảy phép đo → xuất PNG gốc + JPEG giao hàng + siêu dữ liệu
```

Bản thiết kế gốc giao mọi khâu cho ffmpeg (`scale=...:flags=lanczos`,
`unsharp=5:5:0.8`, `eq=contrast=1.12`, `signalstats`). Máy chủ không có
ffmpeg, nên toàn bộ được viết lại bằng thuật toán thật — kể cả **bộ đọc/ghi
PNG** (zlib có sẵn trong Node) và **bộ mã hoá/giải mã JPEG baseline** (Huffman,
lượng tử, DCT/IDCT, nội suy màu). Không đọc nổi một tệp ảnh thì không tầng nào
chạy được.

**Chất màu là SỐ, không phải chuỗi lệnh.** Mỗi hồ sơ máy ảnh là một bộ tham
số hệ thống hiểu được — nên "hồ sơ này khác hồ sơ kia ở đâu" là một phép trừ,
và kiểm thử đo được chất màu có đổi đúng hướng không. **Đo thật:** 8/8 hồ sơ
cho chất màu riêng biệt; ARRI tương phản 32,2 và lạnh −3,2 so với Hasselblad
28,7 / −1,5. Giả lập phim dùng **đường tông thật cho từng kênh** (vai tối nâng
lên, vùng sáng nén lại, ba kênh cong khác nhau) — nhân hệ số bão hoà không bao
giờ ra chất đó.

**Soát bảy phép đo**, mỗi lỗi kèm đúng tham số cần chỉnh: phơi sáng · tương
phản · dải tông · **độ nét** (phương sai Laplace) · cháy sáng · bẹp tối · cân
bằng trắng. Hai điểm bản thiết kế gốc bỏ sót và ở đây có: ảnh mờ và vùng trắng
mất chi tiết đều bị bắt.

**Hiệu năng đo thật trên một lõi** (không mạng, không ffmpeg): một tấm
512×341 chạy trọn tám tầng lên 4K (3072×2046, 6,3MP) mất **4,2 giây**, xuất
PNG gốc + JPEG chất lượng 94 mất thêm **3,4 giây**. Mã hoá JPEG 1,9MP mất
595ms, giải mã 961ms.

**Cửa 3 — sinh ảnh (cần mạng).** Pollinations (miễn phí, không cần khoá) ·
ComfyUI tại máy (0 đồng, dữ liệu không rời máy) · Hugging Face (có khoá). Chưa
có cổng nào chạy được thì trả 503 kèm lối ra, không im lặng.

## 8a. Một lệnh ra một bài giảng có hình, có tiếng

```bash
gita bai-giang "Phương trình bậc hai"
```

Một câu tiếng Việt vào, một tệp video ra. Không cài ffmpeg, không gọi dịch vụ
ngoài, không tốn một đồng nào.

```
Chủ đề khớp dạng M9.PT2.GIAI — Giải phương trình bậc hai một ẩn (độ khớp 86/100)

✔ Giải phương trình bậc hai một ẩn
   15 đoạn · 1042 khung · 1:44 (104.2s) · 1280×720 @10fps
   55.89 MB · dựng trong 7.4s · nguồn kho-hoc-lieu · đã thẩm định
```

**Đường đi.** Chủ đề gõ tự do → khớp vào dạng bài trong chương trình (104 dạng,
tất định, gõ "pytago" cũng ra) → lấy nội dung ĐÃ THẨM ĐỊNH từ kho học liệu →
dựng khung hình → đọc lời giảng bằng giọng nội bộ → đo thời lượng từng đoạn theo
tiếng đọc thật → đóng gói AVI (MJPEG + PCM).

**Ba nguồn nội dung, xếp theo độ tin cậy giảm dần.** Kho học liệu đã thẩm định
(mặc định, ngoại tuyến) · kịch bản tự soạn · mô hình ngôn ngữ (cần khoá, và
nội dung bị đánh dấu `canDuyet: true` — phải có người duyệt trước khi đem dạy).
Không nguồn nào chạy được thì hệ thống nói thẳng, không dựng một bài rỗng.

**Đóng gói video không cần ffmpeg.** `src/film/avi.js` tự ghi container RIFF/AVI:
`hdrl` (avih + hai strl) · `movi` (chunk `00dc` hình xen kẽ `01wb` tiếng) · bảng
tra `idx1` để trình phát tua được. Đọc lại bằng `docAVI()` để tự kiểm: 1042/1042
khung giải nén ra JPEG đúng 1280×720, đường tiếng dài 104,2 giây khớp đúng phần
hình, kích thước khai trong RIFF khớp độ dài tệp thật.

**Chữ tiếng Việt trên khung hình.** Phông 5×7 tự dựng, 203 chữ. Tiếng Việt được
tách bằng chuẩn hoá NFD rồi ghép lại: "ế" = "e" + dấu mũ + dấu sắc, nên 96 chữ
La-tinh cộng 8 dấu phủ hết 134 chữ có dấu.

## 8b. Chuẩn ký hiệu toán — quy ước quốc tế, cưỡng chế bằng máy

Hệ thống có **hai dạng viết toán, cả hai đều chuẩn**, dùng cho hai việc khác nhau:

| Dạng | Dùng ở đâu | Ví dụ |
|---|---|---|
| LaTeX | lưu trữ học liệu (điều A02 Hiến pháp), giao diện web | `\dfrac{-b+\sqrt{\Delta}}{2a}` |
| Unicode phẳng | khung hình video, dòng lệnh, tệp văn bản | `(−b + √∆)⁄(2a)` |

`src/math/kyhieu.js` là cây cầu hai chiều giữa chúng, và cũng nhận luôn lối viết
thô của máy:

```
x^2 - 5x + 6 = 0   →  x² − 5x + 6 = 0
x_1 = 3            →  x₁ = 3
Delta = b^2 - 4ac  →  ∆ = b² − 4ac
sqrt(x+1)          →  √(x + 1)
\dfrac{1}{2}       →  ½          x \in \mathbb{R}  →  x ∈ ℝ
```

**Ba chỗ dễ làm hỏng văn bản, đã xử lý và có kiểm định riêng:**

1. Dấu `/` trong tiếng Việt phần lớn KHÔNG phải phân số. `Ngày 20/11`,
   `60 km/h`, `và/hoặc`, `http://…` đều được giữ nguyên; chỉ đổi thành `⁄` khi
   quanh đó có dấu hiệu toán thật.
2. Dấu `-` cũng vậy: `đen-ta`, `cô-sin`, `Bài 3-4` giữ nguyên; `5 - 3 = 2` thành
   `5 − 3 = 2` (dấu trừ U+2212).
3. Unicode **không có** bản mũ cho mọi ký tự (`q` chẳng hạn). Gặp trường hợp đó
   thì giữ nguyên lối viết cũ và báo lỗi, chứ không nuốt mất ký tự — nuốt mất là
   đổi nghĩa công thức.

`Δ` (chữ cái Hy Lạp U+0394) và `∆` (ký hiệu toán U+2206) là **hai ký tự khác
nhau**; bộ chuẩn hoá luôn đổi về `∆`.

**Đọc lên thành lời.** Ký hiệu đẹp trên màn hình mà máy đọc không hiểu thì bài
giảng hỏng một nửa. Bộ đọc tự bắc cầu Unicode → LaTeX rồi đọc:

```
∆ = b² − 4ac          →  "đen-ta bằng bê bình phương trừ bốn ac"
x = (−b + √∆)⁄(2a)    →  "ích bằng âm bê cộng căn bậc hai của đen-ta trên hai a"
Với mọi x ∈ ℝ, x² ≥ 0 →  "với mọi ích thuộc tập số thực, ích bình phương
                           lớn hơn hoặc bằng không"
```

**Phông chữ phải theo kịp.** 60 ký hiệu toán (`⁰¹²³…ⁿ ₀₁₂…ₙ √ ∛ ∆ ⁄ − · ≤ ≥ ≠
∈ ∉ ⊂ ∪ ∩ ∅ ∀ ∃ ⇒ ⇔ ℝ ℕ ℤ ℚ ℂ ∑ ∫ ∞ α β θ π ½ ¼ ¾ …`) được vẽ vào bảng phông,
khai báo bằng hình chứ không bằng byte thập lục phân để đọc lại còn kiểm được.

## 8c. Nhà khoa học toán — giải, trình bày, và tự soát lại

```bash
gita giai "x^2 - 5x + 6 = 0"        # giải + trình bày + thẩm định
gita soat bai-giai.md               # thẩm định một bài có sẵn
gita phieu "phương trình bậc hai"   # phiếu học tập từ kho đã thẩm định
gita phuong-phap "học trước quên sau"
```

**Tính toán thật trước, mô hình sau.** Hệ số, biệt thức, nghiệm, dấu hiệu nhẩm
nghiệm — tất cả tính bằng bộ máy toán nội bộ (`src/math/verifier.js`), chạy
ngoại tuyến, kiểm lại được. Mô hình ngôn ngữ chỉ được mời vào chỗ nó giỏi hơn
(liên hệ thực tiễn cho chủ đề chưa có trong bảng biên soạn sẵn).

**Không dập khuôn.** Mỗi cách giải chỉ hiện khi nó THẬT SỰ áp dụng được cho
phương trình đang xét:

| Phương trình | Các cách được bày ra | Vì sao |
|---|---|---|
| `x² − 3x + 2 = 0` | công thức nghiệm · **nhẩm nghiệm** · Vi-ét | a + b + c = 0 |
| `x² − 5x + 6 = 0` | công thức nghiệm · Vi-ét | không có dấu hiệu nhẩm, nghiệm nguyên |
| `x² − 6x + 9 = 0` | công thức nghiệm · **hằng đẳng thức** | ∆ = 0 |
| `2x² − 7x + 3 = 0` | công thức nghiệm | nghiệm ½ không nguyên nên bỏ Vi-ét |

**Thẩm định bốn cấp** (`src/math/thamdinh.js`) chạy trên mọi bài trình bày:

```
Cấp 1 · KÝ HIỆU   còn ^ _ sqrt() vec() Δ hay lệnh LaTeX chưa dựng không
Cấp 2 · VĂN PHONG giọng máy ("Tôi sẽ", "Tuyệt vời!"), viết tắt (pt, đc, kl),
                  biểu tượng cảm xúc, ngôi thứ nhất
Cấp 3 · CẤU TRÚC  đủ Đề bài · ĐKXĐ · Lời giải · Kiểm tra lại · Kết luận
Cấp 4 · NỘI DUNG  THAY NGHIỆM NGƯỢC VÀO PHƯƠNG TRÌNH
```

Cấp 4 là cấp không thể làm bằng cách dò chuỗi, và nó được làm thật: bài đúng đạt
100/100; sửa đáp số từ `S = {2; 3}` thành `S = {2; 5}` thì rớt ngay xuống 80/100
với lỗi *"thiếu nghiệm 3; thừa nghiệm 5"* — **dù ký hiệu và văn phong vẫn đạt**.

Bộ soát cũng được dạy để không bắt oan: `∆ ⇒ ≤ ∑ ∫` không bị nhầm thành biểu
tượng cảm xúc, mã dạng `M9.PT2.GIAI` không bị nhầm là viết tắt của "phương trình".

**Phiếu học tập lấy bài từ kho đã thẩm định**, không phải từ đề mô hình bịa ra:
mỗi bài trong phiếu đều đã qua bộ xác minh toán học khi nhập kho, có đáp án, số
sao độ khó và thời gian chuẩn; phiếu tự chia ba nhóm A · B · C theo độ khó thật.

**24 phương pháp học** ghép theo dấu hiệu học sinh nói ra, tất định và khớp theo
TỪ chứ không theo chuỗi con — "trời hôm nay đẹp quá" không được khớp vào "kiến
thức rời rạc" chỉ vì chữ "trời" có chứa "rời".

## 8d. Chuẩn tiếng Việt cho học liệu

Một đề toán lớp 3 viết bằng câu bốn mươi tiếng, dùng từ của lớp 9, thì học sinh
trượt ở khâu **đọc** chứ không phải khâu **toán** — mà nhìn vào điểm số thì
không ai thấy điều đó. Tầng ngôn ngữ đo đúng chỗ ấy.

**Soát bốn cấp**, song song với bốn cấp của bên toán:

| Cấp | Soát gì | Cách soát |
|---|---|---|
| 1 · chính tả | âm tiết có đúng luật cấu tạo, dấu thanh có đúng chỗ | luật cấu tạo âm tiết, không cần từ điển |
| 2 · văn phong | giọng máy, viết tắt, biểu tượng cảm xúc | dùng chung chuẩn với bên toán |
| 3 · độ đọc | câu có vừa sức lớp đó không | đếm tiếng mỗi câu, tỉ lệ vần khó |
| 4 · từ vựng | có dùng từ chương trình chỉ dạy ở lớp trên không | tra bảng 2.380 thuật ngữ dựng từ chính chương trình |

**Soát chính tả bằng luật, không bằng từ điển.** Tiếng Việt có thứ mà tiếng
Anh không có: âm tiết có luật. Mỗi tiếng là *(âm đầu) + (âm đệm) + âm chính +
(âm cuối) + thanh điệu*, và tập các mảnh ấy là hữu hạn. Nhờ vậy `ngiêm` sai vì
`ngi` không phải âm đầu hợp lệ, `trứơc` sai vì dấu rơi ngoài âm chính — bắt
được mà không cần một quyển từ điển nào.

**Đặt dấu thanh đúng âm chính.** Luật: dấu đặt trên âm chính; nguyên âm đôi có
âm cuối thì đặt chữ thứ hai (`cuộc`, `nghiêng`), không có âm cuối thì chữ thứ
nhất (`của`, `mía`). Hệ thống dựng được cả kiểu mới của sách giáo khoa (`hoà`,
`thuỷ`) lẫn kiểu cũ (`hòa`, `thủy`), và **không coi kiểu nào là sai** — chỉ đòi
một văn bản dùng nhất quán một kiểu.

**Không bắt oan là yêu cầu ngang hàng với bắt được lỗi.** Mỗi trường hợp dưới
đây từng bị một phiên bản trước báo sai, nay đều có mục kiểm riêng:
`lôgarit`, `vectơ` (từ mượn viết liền) · `ƯCLN`, `BCNN` (chữ viết tắt) ·
`5 kg`, `2 m` (đơn vị đo) · `Δ`, `x²` (ký hiệu toán) · `M9.PT2.GIAI` (mã dạng
bài) · `Đo tiến bộ…` (chữ Đ hoa nằm trong dải Unicode của chữ thường).

**Chỗ nó không làm được thì nói thẳng.** Không biết `bàn` hay `bàng` mới đúng
trong câu, vì cả hai đều hợp luật. Không bắt được lỗi gõ mà phần sai lại tách
thành hai âm tiết hợp lệ — đó là cái giá để không báo oan `lôgarit` và `vectơ`.

**Bảng ngưỡng theo lớp là quy ước biên tập, không phải số đo.** Hệ thống chưa
có kho ngữ liệu đủ lớn để hiệu chỉnh, nên bảng được ghi rõ là quy ước và sửa
được bằng cấu hình. `thongKeTheoLop()` trả về **số đo thật** của kho kèm cỡ mẫu
để khi nào kho đủ dày thì hiệu chỉnh bằng dữ liệu chứ không bằng phỏng đoán.

**Bài học bốn kỹ năng** dựng từ chính học liệu đã thẩm định: bài đọc là đề bài
thật, bài nghe là lời giảng thật, bài nói là những tiếng vần khó **có trong
chính bài đó**, bài viết là khung trình bày lời giải. Không đoạn văn nào do máy
bịa ra rồi gọi là học liệu.

```bash
gita tieng-viet de-bai.md --lop=3     # soát bốn cấp, thoát mã 1 nếu không đạt
gita am-tiet "Giải phương trình"      # tách âm đầu · đệm · chính · cuối · thanh
gita soi-kho                          # soi ngôn ngữ cả kho, chấm theo đúng lớp
gita bai-ngon-ngu "phương trình bậc hai"
```

## 8e. Trang công khai — dựng sẵn ở máy chủ, chuẩn SEO kiểm được

177 trang HTML hoàn chỉnh ngay từ byte đầu tiên: trang chủ, danh mục chương
trình, **104 trang dạng bài**, **12 trang riêng cho từng lớp**, 3 trang ôn thi
chuyển cấp, thư viện đề thi, **thư viện tài liệu** (kèm 12 trang lọc theo lớp),
trang thi thử, 12 trang lọc chương trình theo lớp, 2 trang lọc theo nhóm ngoài
môn Toán, 24 trang phương pháp học, lộ trình, bộ công cụ, sơ đồ trang và luật
thu thập.

**Thư viện đề thi liệt kê BỘ ĐỀ, không liệt kê tệp đề chép sẵn.** Mỗi lần bấm
vào là hệ thống dựng một đề mới: phủ đủ dạng của phạm vi, trộn độ khó theo ma
trận cố định (nhận biết 15% · thông hiểu 25% · vận dụng 30% · vận dụng cao 20%
· thách thức 10%), loại bài đã làm trong ba mươi ngày gần nhất, và tính giờ từ
thời gian chuẩn của từng bài. Vì vậy cột trong bảng là **số câu · thời gian ·
số dạng phủ** — những con số tính được — chứ không phải "lượt làm" và "điểm cao
nhất" của một tệp nằm sẵn.

**Thư viện tài liệu sinh phiếu luyện lúc bấm tải.** Mỗi dạng bài có một phiếu
gồm bốn phần: dạng cần nắm trước · bẫy thường gặp · ví dụ mẫu có lời giải từng
bước · bài tự luyện. Phiếu trả về dạng Markdown tại `/tai-lieu/<mã dạng>.md`,
dựng từ kho đã qua bộ xác minh toán học. Dạng chưa có học liệu thì đường dẫn
trả 404 chứ không trả một tệp rỗng.

**Đi vào theo lớp, không bắt người đọc tự lọc.** Thanh điều hướng có bảng thả
xuống chia theo lớp 1 đến lớp 12, ôn thi chuyển cấp và hai nhóm ngoài môn Toán.
Bảng này mở được bằng chuột, bằng phím, và **vẫn mở được khi tắt JavaScript**
vì dựng bằng thẻ `details` chứ không bằng mã.

**Trang riêng cho từng lớp** nói được thứ danh mục phẳng không nói ra: ngoài
danh sách dạng bài, mỗi trang lớp còn có **bảng nền từ lớp dưới** — dạng nào ở
lớp dưới đang được lớp này dùng lại, và dùng lại ở mấy chỗ. Học viên vướng ở
lớp 9 thường không phải vì bài lớp 9 khó, mà vì một dạng lớp 8 còn hổng; bảng
đó chỉ đúng vào chỗ hổng. Kèm theo là chiều ngược lại: lớp trên sẽ dùng lại lớp
này ở đâu.

**Trang ôn thi chuyển cấp không phải một kho bài riêng** mà là một lát cắt qua
đúng chương trình đang vận hành: lấy các lớp liên quan rồi siết ngưỡng tầng.
Nhờ vậy nội dung ôn thi không bao giờ lệch khỏi chương trình học hằng ngày.

**Trang không nói suông mà trưng ra việc đang làm.** Trang chủ và mỗi trang
dạng bài đều dựng MỘT BÀI THẬT từ bản thiết kế, in cả lời giải, rồi cho bộ kiểm
chứng giải lại ngay trong lúc trang hiện ra và in luôn vết kiểm đó. Công thức
được dựng tại máy chủ bằng bộ dựng LaTeX của chính hệ thống, nên hiện ra đúng
`x²`, `∆`, phân số có gạch ngang thật — và đọc được cả khi tắt JavaScript. Số
liệu trên biểu đồ đếm ngay lúc dựng trang, không phải con số chép tay.

**Thiết kế dựng hoàn toàn bằng CSS và SVG nội tuyến**: không một phông chữ hay
tệp ảnh nào tải từ ngoài, nên trang vẫn đúng hình trong mạng nội bộ đóng kín.
Linh vật, hình trang trí trên băng mời học thử và biểu đồ đều vẽ thẳng trong
trang. Nền SÁNG là chính vì người đọc là phụ huynh và học sinh phổ thông, nhưng
giao diện tối được thiết kế riêng chứ không đảo màu máy móc; mọi hiệu ứng
chuyển động tắt theo thiết lập trợ năng `prefers-reduced-motion`.

**Mỗi lớp một màu, giữ nguyên suốt cả trang.** Học sinh nhận ra lớp mình theo
màu nhanh hơn theo số, nên mười hai lớp có mười hai màu cố định: thẻ khoá học
ngoài trang chủ, ô trong bảng thả xuống, dải trên đầu trang lớp và nhãn trong
chân trang đều cùng một màu. Thẻ khoá học của mỗi lớp KHÔNG dùng lời giới thiệu
viết sẵn mà in tên ba dạng bài thật của lớp đó — đổi chương trình thì thẻ đổi
theo, không ai phải nhớ đi sửa lại.

**Vì sao dựng ở máy chủ.** Công cụ tìm kiếm đọc HTML. Trang trắng chờ
JavaScript vẽ ra phải chờ vòng thu thập thứ hai, và nhiều bộ thu thập không
quay lại.

**Chiều sâu là nội dung thật.** Mỗi trang dạng bài có tên dạng, tầng, điều kiện
cần nắm trước (liên kết chéo sang dạng tiên quyết), bẫy thường gặp, thời gian
chuẩn, yêu cầu lên tầng và các dạng cùng lớp — đọc từ chương trình đang vận
hành, không phải khuôn mẫu lặp lại.

**Mười lăm mục SEO được viết thành luật kiểm được**, chạy trong `npm run verify`
và `npm test`: độ dài tiêu đề 20–62 ký tự · thẻ mô tả 70–165 ký tự · đúng một
thẻ `h1` · canonical · `lang="vi"` · dữ liệu có cấu trúc · thẻ chia sẻ mạng xã
hội · ảnh có `alt` · không tải tài nguyên từ tên miền khác · đủ chữ đọc được ·
liên kết nội bộ · liên kết bỏ qua thanh điều hướng · đọc được khi tắt JavaScript ·
tệp mô tả ứng dụng · mọi khối mã mang nonce.

**177/177 trang đạt, 177 tiêu đề và 177 thẻ mô tả không trùng nhau.** Trùng
tiêu đề là lỗi nặng: công cụ tìm kiếm coi hai trang là một và bỏ bớt một trang.

**Dữ liệu có cấu trúc** khai theo schema.org: `EducationalOrganization` ·
`WebSite` kèm ô tìm kiếm · `BreadcrumbList` · `Course` cho từng dạng bài ·
`FAQPage` cho phần hỏi đáp trang chủ.

**Không gọi ra ngoài.** Không phông chữ tải về, không thư viện, không mã theo
dõi. Trang nhẹ, không rò dữ liệu người đọc, và không hỏng khi mạng ngoài trục
trặc.

**Chính sách bảo mật nội dung dùng nonce, không mở `unsafe-inline`.** Mỗi lượt
tải sinh một nonce mới; mọi khối `<style>` và `<script type="application/ld+json">`
trong trang mang đúng nonce của lượt đó. Mở `unsafe-inline` là vô hiệu hoá gần
hết tác dụng chặn mã lạ của chính sách, nên ở đây không mở. Kèm
`X-Frame-Options`, `Permissions-Policy` và `Referrer-Policy`.

**Cài được lên màn hình chính.** `/manifest.webmanifest` khai tên, biểu tượng,
màu nền và ba lối tắt: Chương trình · Hành trình của tôi · Bàn làm việc Toán.

**Chữ trên trang phải qua chính bộ soát tiếng Việt của hệ thống.** Một sản phẩm
soát chính tả mà trang giới thiệu của nó lại sai chính tả thì không còn gì để
nói. 177/177 trang sạch: không giọng máy, không viết tắt, không sai chính tả.

**Đường dẫn `/` chia theo thứ người gọi yêu cầu**: trình duyệt xin `text/html`
thì nhận trang, máy gọi API vẫn nhận đúng khối JSON như trước, và `/api` luôn
trả JSON.

## 8f. Thanh tra — một lệnh, cả hệ thống bị soi

```bash
gita thanh-tra
```

Huy động **thật** lực lượng trợ lý AI rồi chạy đủ các phép soát của từng tầng,
gom vào một biên bản. Thoát mã 1 nếu có mục chặn, cắm thẳng vào quy trình phát
hành.

```
▸ LỰC LƯỢNG          500 trợ lý AI · 12 mảnh việc · 12 xong · 0 chồng chéo · 0 vi phạm Hiến pháp
▸ NỀN HỆ THỐNG       nhật ký móc xích nguyên vẹn · 50 điều · 104 dạng · 7 tiêu chí
▸ KHOA HỌC TOÁN      4/4 bài đạt 100/100 · bài sai đáp số rớt còn 80/100
▸ KHOA HỌC NGÔN NGỮ  83 bài đạt 100% · 2.380 thuật ngữ tra được cấp lớp
▸ TRANG CÔNG KHAI    177 trang đạt SEO · 177 tiêu đề riêng · 0 trang sai tiếng Việt
▸ BÀI GIẢNG          9 đoạn · 466 khung · 58,4 giây · 13,9 MB · thẩm định 100/100
KẾT QUẢ: 15/15 mục đạt — TOÀN BỘ ĐẠT        3,5 giây · chi phí $0
```

## 8g. Ban quản trị quan hệ gia đình — 30 trợ lý AI chuyên môn

Một ban riêng lo mảng quan hệ với gia đình học viên: từ lúc một gia đình biết
tới hệ thống, qua tư vấn lộ trình, học phí, chăm sóc, cho tới lúc em nhỏ đi
hết chặng. Ba mươi trợ lý này **không phải ba mươi trợ lý nữa cộng vào 500**.
Đây là tầng quyết định: họ đọc dữ liệu thật, cân nhắc, và **giao việc xuống**
lực lượng 500 khi việc cần sức làm — cây cầu ấy ở `src/crm/hop-tac.js`, có ba
cửa thật chứ không phải một dòng mô tả.

### Bốn cấp, và trần quyền đi NGƯỢC với cấp

| Cấp | Số người | Trần tự chủ | Vì sao |
|---|---|---|---|
| 1 · Điều hành | 5 | 2 | nhìn rộng nhất nên một lệnh sai lan xa nhất |
| 2 · Trưởng ban | 8 | 3 | điều phối một mảng, soạn thảo và đề xuất |
| 3 · Chuyên viên | 12 | 3 | việc hẹp, hỏng thì hỏng một chỗ và lùi lại được |
| 4 · Vi chuyên viên | 5 | 3 | một việc lặp lại, đo được |

### Bốn mức động chạm — thang đo "lùi lại được tới đâu"

```
1 · chỉ đọc          không đổi gì
2 · soạn nháp        có sản phẩm, chưa rời khỏi hệ thống
3 · ghi vào sổ nhà   đổi dữ liệu của chính mình, sửa lại được, có vết
4 · chạm ra ngoài    tới một con người thật — KHÔNG LÙI LẠI ĐƯỢC
```

**Mức 4 không cấp cho ai, kể cả bằng lệnh của Super Admin.** Mọi việc mức 4
dừng ở hàng chờ duyệt cho tới khi một người thật gật đầu. Luật này nằm ở ba
tầng độc lập — sổ trợ lý, bàn điều khiển, bộ cân mức — và mục kiểm
`Không trợ lý nào tự chạm được ra ngoài hệ thống` soi cả ba cùng lúc.

### Một lượt đi qua chín cửa

```
1 ghi câu hỏi   2 hỏi bàn điều khiển   3 định tuyến   4 cân mức tự chủ
5 chạy tổ       6 hàng chờ duyệt       7 soi lời trả lời (bức tường)
8 ghi tín hiệu  9 nhật ký kiểm toán
```

Cửa 3 đọc không ra ý định thì **hỏi lại**, không đẩy bừa cho trợ lý điều hành.
Cửa 7 soi mọi câu trả lời qua bức tường Cây Tiền **kể cả khi người hỏi là quản
trị** — màn hình quản trị cũng có lúc bị chụp lại và gửi đi.

### Công cụ đọc dữ liệu thật, kho rỗng thì nói là rỗng

Hai mươi công cụ đọc kho học viên, kho khảo sát, sổ cái dòng tiền, sổ rủi ro,
bốn mươi tình huống và lưới mười nghìn điểm chạm. **Không công cụ nào trả dữ
liệu giả lập.** Kho chưa có bản ghi thì công cụ trả về đúng câu ấy —
*"Chưa có kỳ học phí nào trong kho"* — kèm việc nên làm tiếp. Mục kiểm
`Công cụ nói "chưa có" khi kho rỗng` chặn cả trường hợp một con số sáu chữ số
lọt vào lúc kho đang trống.

### KPI: chấm trên chỉ số có nguồn, đếm riêng chỉ số chưa có

62 chỉ số cho 30 trợ lý, trong đó **37 đã nối được nguồn đo và 25 khai rõ là
chưa**. Bảng điểm loại 25 chỉ số ấy khỏi phép tính và in ra con số ấy, thay vì
chấm bừa cho bảng đỡ trống. Trợ lý chưa đủ 10 lượt thì không chấm — mọi tỷ lệ
dưới ngưỡng ấy đều là nhiễu.

### Mục quản trị dành riêng cho Super Admin

`/quan-tri/crm` — bàn điều khiển dựng sẵn ở máy chủ, không một dòng
JavaScript, canh cửa bằng chính bức tường Cây Tiền. Bảy việc làm được: xem ai
đang được làm gì, hạ mức tự chủ, khoá trợ lý, siết ngưỡng, dừng cả ban trong
một lệnh, duyệt việc đang chờ, đọc bảng điểm.

**Bốn luật không ai nới được, kể cả Super Admin:**

- Không **nâng** mức tự chủ vượt trần đã khai trong sổ. Chỉ hạ được.
- Không cấp mức 4 cho bất kỳ ai.
- Không tắt được nhật ký kiểm toán.
- Không xoá được một việc đã quyết ở hàng chờ.

Và mọi lệnh đều đòi **tên người ra lệnh** — không có lệnh nào "của hệ thống".

### Hàng chờ duyệt: im lặng không bao giờ là đồng ý

Việc chờ quá 72 giờ thì đóng lại ở trạng thái **hết hạn**, không thành đồng ý.
Bảng đo của hàng chờ in riêng số việc hết hạn, vì con số đáng lo nhất không
phải "bao nhiêu việc đang chờ" mà là "bao nhiêu việc đã chết vì không ai tới".

### Hợp tác với lực lượng 500 và hai nhà khoa học

```bash
POST /crm/hop-tac/giao-viec        # một trợ lý trong 500 làm
POST /crm/hop-tac/giao-viec  {"to":true}   # cả tổ: chia việc · song song · phản biện · gộp
POST /crm/hop-tac/nha-khoa-hoc     # nhà khoa học toán hoặc ngôn ngữ
```

Ba mươi trợ lý CRM biết việc quan hệ gia đình. Họ **không** biết thẩm định một
lời giải toán hay soi chính tả bốn cấp — nên việc ấy giao sang đúng người, thay
vì để họ đoán.

### Năm việc chạy nền

`ban-tin-sang` (7 giờ) · `soi-cong-no` (6 giờ) · `quet-bat-thuong` (1 giờ) ·
`don-het-han` (1 giờ) · `cham-kpi` (24 giờ). Một việc đang chạy thì lần tới bỏ
qua chứ không xếp hàng; việc hỏng thì ghi lại chứ không kéo sập vòng lặp.

```bash
GET  /crm                      # trạng thái toàn ban
POST /crm/hoi                  # hỏi một việc
GET  /crm/kpi                  # bảng điểm 30 trợ lý
GET  /crm/duyet                # hàng chờ duyệt
POST /crm/duyet/<mã>/quyet     # người thật quyết
GET  /crm/quan-tri             # bàn điều khiển Super Admin
GET  /quan-tri/crm             # màn hình quản trị (HTML)
```

## 8h. Đăng nhập bằng khuôn mặt thật

### Vì sao KHÔNG so khớp khuôn mặt ở máy chủ

Yêu cầu là **khuôn mặt thật, không phải ảnh**. Có hai cách làm, và chúng khác
nhau về bản chất.

**Cách bị loại — máy chủ tự so khớp.** Trình duyệt chụp ảnh, gửi lên, máy chủ
so đặc trưng. Chống ảnh giả bằng "kiểm tra sống": bảo chớp mắt, quay đầu.
Cách này hỏng ở ba chỗ:

1. **Kiểm tra sống kiểu ấy không chống được ảnh.** Nó chặn được ảnh in, còn
   một đoạn **video** quay sẵn trên điện thoại thì chớp mắt, quay đầu, cười
   đủ cả. Camera thường chỉ thấy màu, không thấy chiều sâu. Kết quả là lớp
   bảo vệ *trông như thật* mà không thật — nguy hiểm hơn không có gì, vì
   người ta tin vào nó.
2. **Nó bắt hệ thống giữ sinh trắc học của trẻ em.** Nghị định 13/2023/NĐ-CP
   xếp đặc trưng khuôn mặt vào dữ liệu cá nhân **nhạy cảm**. Mật khẩu lộ thì
   đổi được; khuôn mặt lộ thì không, và đứa trẻ mang hậu quả suốt đời.
3. Nó cần một mô hình học máy — hệ này không phụ thuộc gói ngoài nào.

**Cách được chọn — để phần cứng của máy làm.** WebAuthn với bộ xác thực gắn
trong máy: Windows Hello Face, Touch ID, Face ID.

- **Windows Hello Face dùng camera hồng ngoại đo chiều sâu.** Ảnh in không
  qua. Màn hình điện thoại phát video cũng không qua — vì nó phẳng, và cảm
  biến thấy điều đó. Đây đúng là *"khuôn mặt thật, không phải ảnh"*, và nó
  thật ở **tầng phần cứng**, không phải ở một đoạn mã đoán xem người dùng có
  chớp mắt không.
- **Đặc trưng khuôn mặt không bao giờ rời khỏi máy người dùng.** Nó nằm trong
  vùng bảo mật của con chip. Máy chủ chỉ nhận một **khoá công khai** — thứ lộ
  ra cũng không làm gì được. Không có kho sinh trắc học nào để rò.
- Không cần gói phụ thuộc nào. Đây là tiêu chuẩn ngân hàng đang dùng.

### Bốn điều kiện, máy chủ tự kiểm lại từ dữ liệu đã ký

Trình duyệt được *yêu cầu*, nhưng yêu cầu không phải bảo đảm — kẻ tấn công
gửi thẳng gói tin thì không có trình duyệt nào để mà yêu cầu.

| Điều kiện | Nếu thiếu |
|---|---|
| Cờ **đã xác minh người dùng** bật | nhận cả mã PIN, không còn là sinh trắc học |
| Bộ xác thực **gắn trong máy** | nhận cả khoá cắm rời |
| **Bộ đếm lần ký phải tăng** | bỏ qua dấu hiệu có bản sao khoá riêng |
| **Thách thức dùng một lần**, sống 2 phút | nhận cả gói tin phát lại |

### Dữ liệu trẻ em

Tài khoản học viên gắn khuôn mặt thì **phải có người giám hộ đồng ý**, khai đủ
bốn vế: họ tên, quan hệ với em, thời điểm, và xác nhận rằng **đồng ý này rút
lại được**. Gỡ máy bất cứ lúc nào chính là đường rút lại ấy — một đồng ý không
rút lại được thì không phải đồng ý.

### Điều hệ thống này KHÔNG làm, nói thẳng

Máy chủ **không biết khuôn mặt ai**. Nó chỉ biết: một bộ xác thực gắn trong
máy, đã xác minh người dùng bằng sinh trắc học, giữ khoá riêng khớp với khoá
công khai đã đăng ký. Cụ thể là vân tay hay khuôn mặt thì do người dùng chọn
lúc cài Windows Hello, và máy chủ **không được biết** — đó là điều đúng đắn,
không phải thiếu sót.

### Vẫn còn mật khẩu

Đây là lối vào **thêm**, không thay lối cũ. Mất camera, hỏng cảm biến, đổi
máy — vẫn vào được bằng mật khẩu. Một hệ chỉ có đúng một cách vào là một hệ
khoá được người dùng ra ngoài.

Nút "Đăng nhập bằng khuôn mặt" **chỉ hiện khi máy thật sự có cảm biến**. Hiện
sẵn rồi báo lỗi khi bấm là hứa một thứ không có.

```bash
POST /khuon-mat/xin-dang-nhap    # công khai — không tiết lộ tài khoản nào tồn tại
POST /khuon-mat/dang-nhap        # công khai — đi qua đúng hàng rào khoá tài khoản
POST /khuon-mat/xin-dang-ky      # đòi phiên đăng nhập
POST /khuon-mat/dang-ky          # đòi phiên; học viên phải có đồng ý giám hộ
GET  /khuon-mat/may              # danh sách máy đã gắn, tối đa 5
DELETE /khuon-mat/may/<mã>       # gỡ máy — cũng là đường rút lại đồng ý
```

**Ràng buộc của chuẩn:** ngoài `localhost`, WebAuthn **chỉ chạy trên HTTPS**.
Đây là luật của trình duyệt, không tắt được. Khai `WEBAUTHN_RP_ID` và
`WEBAUTHN_ORIGINS` khớp tên miền thật.

## 8i. Chống lừa đảo — bắt đầu bằng việc nói rõ cái gì đã an toàn sẵn

### Đăng nhập bằng khuôn mặt TỰ NÓ đã chống lừa đảo

Điều này dễ bị quên, nên nói trước. Chữ ký WebAuthn được gắn chặt vào **gốc**
của trang:

```
Người dùng bị dẫn tới gitamath-that.vn, một bản sao giống hệt.
Họ bấm đăng nhập, quét mặt như mọi ngày.
Trình duyệt ký kèm gốc "gitamath-that.vn".
Kẻ tấn công mang chữ ký ấy sang gitamath.vn.
Máy chủ thật thấy gốc sai, TỪ CHỐI.
```

Người dùng không cần tinh ý, không cần nhìn thanh địa chỉ, không cần biết lừa
đảo là gì. **Máy làm thay họ.** Mật khẩu không bao giờ làm được điều này: mật
khẩu gõ vào trang giả là trang giả có mật khẩu.

### Bốn đường vòng còn lại, và bốn lớp canh

Lừa đảo không chỉ nhắm vào lúc đăng nhập.

| Đường vòng | Lớp canh |
|---|---|
| Mật khẩu vẫn còn đó làm lối dự phòng | Việc nguy hiểm đòi **quét mặt lại**, dù đang đăng nhập |
| Chiếm phiên — vé bị chép từ máy nhiễm mã độc | Máy lạ đăng nhập là **báo động**, không phải lặng lẽ |
| Lừa chính người dùng tự làm việc nguy hiểm | Gom **dấu hiệu tấn công** và chấm, không để mỗi lần một nơi |
| Dò xem tài khoản nào có thật | Trả lời **giống hệt nhau** dù tài khoản có hay không |

### Sáu việc nguy hiểm, mỗi việc kèm một câu nói rõ vì sao

`doi-mat-khau` · `gan-khuon-mat` · `go-khuon-mat` · `doc-tam-ly` ·
`xoa-du-lieu` · `cap-quyen`

Bước xác thực lại **chỉ nhận khuôn mặt**, không nhận mật khẩu gõ lại — vì kẻ
đã chiếm được phiên thường đã có sẵn mật khẩu. Hiệu lực **5 phút**: đủ làm
xong việc, không đủ để quên.

### Điều hệ thống này KHÔNG làm

Nó **không chấm "điểm tin cậy"** rồi tự chặn người dùng theo một con số không
ai giải thích được. Bảy dấu hiệu, mỗi dấu hiệu nói được lý do bằng một câu và
kèm việc phải làm; mỗi lần chặn đều chỉ ra dấu hiệu nào gây ra. *Một hệ chặn
người mà không nói được vì sao thì người bị chặn oan không có đường nào kêu.*

Dò tài khoản được đếm theo **số TÊN khác nhau** đã thử, không phải số lần thử
— thử một tài khoản hai mươi lần là người quên mật khẩu, thử hai mươi tài
khoản mỗi cái một lần mới là đang dò.

Địa chỉ mạng và chuỗi trình duyệt **không bao giờ lưu nguyên bản**, chỉ lưu
dấu vân băm 22 ký tự — đủ để nhận ra máy cũ, không đủ để theo dõi ai.

```bash
GET  /chong-lua-dao                 # quản trị — tình trạng lớp canh
GET  /chong-lua-dao/tinh-hinh       # quản trị — dấu hiệu gom theo nguồn
GET  /chong-lua-dao/dau-hieu        # quản trị — nhật ký dấu hiệu
GET  /chong-lua-dao/cho-lam/<việc>  # học viên — hỏi trước khi hiện nút
GET  /chong-lua-dao/may-cua-toi     # học viên — chỉ thấy máy của chính mình
POST /chong-lua-dao/doi-mat-khau    # học viên — đi qua cổng việc nguy hiểm
```

## 8j. Sơ đồ tư duy ba chiều — cho bài tập và tổng hợp kiến thức

### Vì sao ba chiều chứ không phải hai

Đồ thị tiên quyết của **104 dạng** có nhiều cạnh cắt nhau; vẽ phẳng thì thành
một búi chỉ rối. Trải ra ba chiều và cho **xoay được** thì người học tự tìm
được góc nhìn mà ở đó đường đi rõ ra. Đây là lý do kỹ thuật, không phải lý do
trang trí.

### Ba cách dựng, cho ba việc học khác nhau

| Kiểu | Trả lời câu hỏi |
|---|---|
| `toan-canh` | Cả chương trình trải theo tầng — chỗ nào dày, chỗ nào thưa |
| `duong-di` | *"Muốn học dạng này thì phải biết gì trước?"* — đi ngược hết chuỗi tiên quyết |
| `quanh-mot` | *"Ôn chủ đề này thì có gì quanh nó?"* — hàng xóm hai chiều, xa tối đa 3 bước |

### Không một nút nào được bịa

Mọi nút là một dạng **có thật** trong `src/curriculum`; mọi đường là một quan
hệ tiên quyết **đã khai**. Hệ quả có chủ ý: sơ đồ này **xấu ở đúng những chỗ
chương trình còn thưa**. Ba dạng hiện chưa ai khai quan hệ tiên quyết nên đứng
rời — hệ thống **nói ra con số ấy** thay vì nối bừa cho hình tròn đều đẹp mắt.

### Tự viết phép chiếu, không thư viện, không WebGL

Bốn việc, cả bốn là hình học phổ thông: xoay quanh hai trục · chiếu phối cảnh ·
xếp theo chiều sâu (xa vẽ trước, gần vẽ sau — thứ tự vẽ **chính là** che khuất)
· mờ dần theo xa.

WebGL bị loại vì nó cần thư viện, cần card màn hình chạy được — máy tính trường
học thường không có — và **không dựng được ở máy chủ**. Sơ đồ này phải **in ra
giấy được**, **nhúng vào báo cáo được**, **mở được trên máy cũ**. SVG làm được
cả ba.

### Ba lỗi bắt được bằng cách dựng hình ra rồi NHÌN

1. **Nhãn chồng lên nhãn.** Bản đầu chỉ lọc nhãn theo chiều sâu, và vùng giữa
   hình dày đặc chữ đè chữ — hai mươi nhãn chồng nhau cho ít thông tin hơn năm
   nhãn rời. Nay mỗi nhãn phải **xin một ô chữ nhật** trên màn hình; xét theo
   thứ tự **gần trước**, nên nút gần luôn thắng nút xa.
2. **Chú thích đếm thiếu.** 18 dạng tiếng Anh và liên môn rơi ngoài bảng màu,
   nên chú thích cộng ra 86 trong khi hình có 104 chấm. Nay có mục kiểm bắt
   buộc **tổng chú thích phải bằng số nút**.
3. **Hình tràn ra ngoài khung.** Phép chiếu phóng to nút ở gần, nên ở một số
   góc xoay tầng dưới cùng rơi xuống `y = 745` của một khung cao `720` và bị
   cắt cụt. Nay sau khi chiếu, hệ đo lại hộp bao thật rồi **thu cả hình cho
   vừa khung**.

Cả ba nay do máy canh, ở 5 góc xoay khác nhau, trong `npm run verify` và
`npm test`.

### Tất định tuyệt đối

Không `Math.random`, không giờ hệ thống. Cùng một sơ đồ và cùng một góc nhìn
thì ra **cùng một tệp ảnh tới từng chữ số** — nên ảnh trên màn hình và ảnh in
ra giấy là một, và lệch một chút là biết ngay.

Tệp SVG trả về **không có kịch bản, không phông chữ ngoài, không một đường dẫn
nào ra mạng**. Một tệp ảnh gửi qua lại giữa thầy và trò mà chạy được mã thì nó
không còn là ảnh, nó là một cái bẫy.

### Xoay ở đâu

Trang `ui/so-do-3d.html`: kéo chuột để xoay, thả ra thì máy chủ dựng lại ảnh
(giãn nhịp 110ms). Xoay ở **máy chủ** chứ không ở trình duyệt, để ảnh đang xem
và ảnh tải về **là cùng một tệp**.

```bash
GET /so-do-3d/dang       # danh sách dạng để chọn gốc
GET /so-do-3d            # ?kieu=toan-canh|duong-di|quanh-mot &goc= &buoc=
GET /so-do-3d/hinh.svg   # ?doc= &ngang= &rong= &cao= &nen=0 &chu=het
GET /so-do-3d/soi        # sơ đồ tự soi chính nó
```

## 8k. Trợ lý trò chuyện — chín cửa cho một lượt hỏi

### Vì sao chín cửa chứ không phải một

Một trợ lý trò chuyện hỏng theo kiểu **không hiện ra trên màn hình**. Nó vẫn
trả lời trơn tru; chỉ là nó vừa đọc dữ liệu của một gia đình khác, vừa hứa một
điều không ai hứa được, hoặc vừa làm một việc mà người dùng không hỏi.

```
1. Lọc đầu vào          hình dạng tấn công
2. Hàng rào nhắc lại    lệnh giấu trong câu
3. Đọc ý → kế hoạch     khai TRƯỚC sẽ gọi những gì
4. Khiên việc           mỗi công cụ có phục vụ đúng việc không
5. Mở lượt cho lính gác ghi kế hoạch xuống để về sau so
6. Chạy từng bước       lính gác soi từng bước một
7. Soạn câu trả lời     CHỈ từ dữ liệu vừa đọc được
8. Canh đường ra        che số riêng, chặn hứa hẹn và chẩn đoán
9. Ký vào sổ công chứng cả lượt, không chối được
```

### Ba chỗ làm khác bản thiết kế gốc, và lý do

Bản thiết kế gốc hỏi một **mô hình ngôn ngữ** ở ba chỗ: "hành động này có
ALIGNED không", "độ lệch là bao nhiêu từ 0 đến 1", "đây có phải injection
không". Ở đây cả ba đều thay bằng **bảng khai đọc được**, vì ba lý do đều là
lý do thật:

1. **Mô hình trả lời không tất định.** Cùng một hành động, hôm nay ALIGNED,
   mai DEVIATED. Một bức tường đổi ý theo ngày thì không phải bức tường.
2. **Chính mô hình ấy là thứ đang bị tấn công.** Nhờ nó gác cửa cho chính nó
   là nhờ người đang bị lừa tự kiểm tra xem mình có bị lừa không.
3. **Bảng khai thì người quản trị mở ra xem được;** hỏi mô hình thì không ai
   xem được gì.

Hệ quả: cả chín cửa **chạy được khi không có mạng và không có khoá**. Một lớp
phòng thủ chỉ hoạt động khi trả được tiền là một lớp phòng thủ không có.

### Lớp 2 tự khai khi nó không soi được

Hàng rào nhắc lại cần một mô hình nhỏ. Không có thì nó **nói thẳng** là đang ở
chế độ rút gọn và *yếu hơn hẳn* bản đầy đủ — chứ không im lặng tụt xuống rồi
vẫn báo "năm lớp đang chạy". Một lớp phòng thủ tự nhận mạnh hơn thực tế còn
nguy hiểm hơn không có lớp ấy, vì người ta sẽ dựa vào nó.

### Sáu công cụ bị cấm tuyệt đối, kể cả với quản trị

`doc-tam-ly` · `doc-cay-tien` · `gui-thu` · `xoa-du-lieu` · `cap-quyen` ·
`doi-mat-khau`

Đây không phải "chưa khai" — đây là khai thẳng rằng **CẤM**. Hai thứ khác
nhau: chưa khai là sót, cấm là quyết định.

### Phạm vi dữ liệu không được rộng ra giữa chừng

Mỗi công cụ khai `chamAi` (`chung` · `minh` · `pham-vi`); khiên việc so nó với
việc đang hỏi; lính gác canh không cho rộng ra giữa lượt. **Ba chỗ cùng đọc
một khai báo** — nới một chỗ mà quên hai chỗ kia thì không đi qua được.

Phép đối chiếu ấy đã bắt được một lỗi thật ngay lần chạy đầu: việc *"hỏi về
chương trình học"* khai phạm vi công khai nhưng liệt công cụ đọc **lộ trình
của một học viên cụ thể**. Lính gác chặn đúng — nhưng nó chặn một câu hỏi
công khai hoàn toàn vô can, vì cái sai nằm ở bảng chứ không ở câu hỏi.

### Sổ công chứng Ed25519 — và điều nó KHÔNG chứng minh

Nhật ký móc xích chứng minh được *không ai sửa giữa chừng*. Nó **không** chứng
minh được dòng ấy do chính hệ thống này ghi — vì băm không cần bí mật nào, nên
người có quyền ghi đè cả tệp vẫn dựng lại được cả cuốn sổ sạch sẽ.

Chữ ký Ed25519 bịt đúng chỗ ấy. Nhưng nó vẫn **không chứng minh nội dung ghi
vào là đúng sự thật**. Một lời nói dối được ký vẫn là một lời nói dối — chỉ là
một lời nói dối không chối được.

```bash
GET  /tro-chuyen                 # tình trạng chín cửa
POST /tro-chuyen/hoi             # hỏi một câu
GET  /tro-chuyen/cong-cu         # 12 công cụ đọc dữ liệu thật
GET  /tro-chuyen/bang-viec       # bảng việc + danh sách cấm tuyệt đối
GET  /tro-chuyen/so-cong-chung   # quản trị — sổ ký, kèm khoá công khai
```

## 8l. Tiếp thị nội dung — trang viết từ số liệu thật

### Chỗ dễ nói dối nhất trong cả hệ thống

Nên nói trước điều xưởng nội dung **không** làm:

- **Không** viết "hơn 10.000 học viên tin dùng" khi kho có 0 học viên
- **Không** viết "98% phụ huynh hài lòng" khi chưa ai được hỏi
- **Không** viết một lời chứng thực của một phụ huynh không có thật

Mọi con số đọc thẳng từ kho đang chạy, và **mỗi con số in kèm nguồn** — một
đường dẫn tới tệp mã, không phải một cái nhún vai. Kho rỗng thì mảng ấy
**không được sinh ra** — không sinh ra với số 0, tuyệt đối không sinh ra với
một con số nghe cho đẹp.

Điều này làm trang lúc mới dựng trông **thưa**. Đó là đúng: một trang thưa nói
thật thì sửa được bằng cách làm cho có thật; một trang dày nói dối thì không.

### Bốn luật cứng, máy cưỡng chế

| Luật | Vì sao |
|---|---|
| Không hứa kết quả | Dùng **chung một bộ khuôn** với lớp canh đường ra của trợ lý — một luật, một chỗ định nghĩa |
| Không nhồi từ khoá | Viết cho cỗ máy tìm kiếm thì đọc lên nghe như máy nói, và ba mẹ cảm được |
| Không giọng máy | 12 cụm chỉ xuất hiện khi chữ do máy sinh mà không ai đọc lại |
| Không lộ phân loại nội bộ | Bức tường sẵn có của hệ này |

### Phép đo nhồi từ khoá đã phải HẸP LẠI hai lần

Ghi lại, vì nó nói rõ phép đo này đo được gì và không đo được gì:

- **40 → 120 từ.** Đoạn bốn mươi từ lặp chữ *"đúng"* ba lần đã vượt ngưỡng 5%.
  Ba lần trong bốn mươi từ là tiếng Việt bình thường.
- **120 → 250 từ có nghĩa.** Kịch bản hướng dẫn một màn hình nhắc chữ *"màn"*
  tám lần. Trong một bài nói **về** một màn hình thì danh từ chính phải lặp.

Và một lỗi thứ ba: cửa mở đếm **tổng** số từ trong khi tỉ lệ tính trên số từ
**có nghĩa** — hai mẫu số khác nhau cho cùng một phép đo.

Nâng ngưỡng làm phép đo **hẹp lại**, và nói thẳng điều đó: nó chỉ còn soi văn
xuôi dài — đúng chỗ nhồi từ khoá thật sự xảy ra. *Một phép đo hẹp mà đúng thì
dùng được; một phép đo rộng mà báo oan bốn lần liên tiếp thì người ta sẽ tắt
nó đi.*

### Vòng cải tiến KHÔNG tự sửa trang

Nó **đề xuất**; người duyệt. Lý do rất cụ thể: một vòng tự sửa trang theo số
liệu sẽ trôi dần về phía câu chữ nào được bấm nhiều nhất — và câu được bấm
nhiều nhất trong ngành này thường là **câu hứa nhiều nhất**. Vòng tối ưu theo
lượt bấm sẽ **tự đi tới** chỗ hứa hẹn, không cần ai cố ý.

Ba chỉ số nó **chưa đo được** đều khai tên: trích dẫn của mô hình ngoài, thứ
hạng tìm kiếm, thời gian đọc trang. Khai ra chứ không in số 0 như thể đã đo.

```bash
GET  /tiep-thi/trang          # trang soạn từ số liệu thật
GET  /tiep-thi/do-thi         # đồ thị tri thức dùng chung
POST /tiep-thi/soi-noi-dung   # soi một đoạn bất kỳ
POST /tiep-thi/vong-cai-tien  # chạy một vòng — chỉ đề xuất
```

## 8m. Video hướng dẫn — 40 video, giọng dựng tại máy

### Cùng một màn, bốn vai, bốn video khác nhau

Một giáo viên xem video quay cho học viên sẽ nghe toàn những việc mình không
làm; một học viên xem video quay cho quản trị sẽ nghe những nút mình không bấm
được. Cả hai đều tắt giữa chừng.

**10 màn hình × 4 vai = 40 video**, mỗi video tám chương.

### Chương thứ ba đọc từ MÃ, không khai tay

Chương *"quyền của bạn ở đây"* là chương dễ nói sai nhất, và nói sai ở đây thì
người xem đi làm một việc họ không có quyền, thất bại, rồi nghĩ hệ thống hỏng.

Nên nó **không được viết tay**. Bộ quét đọc mã giao diện, tìm mọi lượt gọi API
(174 lượt trên 10 màn), tra từng cổng qua đúng bảng cổng quyền của hệ, rồi lấy
mức cao nhất.

Bộ quét ấy đã phải sửa **năm lần**, và mỗi lần đều vì nó nói sai về quyền:

1. Đoán tên tệp kèm bằng cách đổi `.html` thành `.js` — trung tâm chỉ huy nạp
   `app.js` nên đọc ra **0 cổng**, và mức quyền tụt xuống "ai cũng xem được"
2. Chỉ bắt `G.api(` mà không bắt `api(` — hai màn lớn nhất đọc ra 0 cổng
3. Đọc cả **chú thích**, nên ví dụ minh hoạ `api('/duong/dan')` trong tệp hạ
   tầng chung bị đếm như lượt gọi thật ở **mọi** màn
4. **Bỏ qua phương thức HTTP** — `POST /accounts/login` là công khai nhưng
   nhóm ACCOUNTS ở mức quản trị, nên mọi màn có thanh đăng nhập bị gắn nhãn
   "chỉ quản trị"
5. Cắt đường dẫn tại `${` thay vì đổi thành ký tự thay — `/learner/${id}/portrait`
   teo lại thành `/learner/`

Lỗi thứ tư là lỗi tệ nhất: nó nói với một **học viên** rằng màn sơ đồ tư duy
là màn quản trị, trong khi em ấy mở được.

### "Giọng chuẩn chất lượng" nghĩa là gì ở đây

Nói rõ, vì hai chữ *"chất lượng"* dễ hiểu thành hai thứ khác hẳn nhau.

**Được:** giọng tổng hợp **tại máy** bằng `src/voice` — âm vị học tiếng Việt,
đúng thanh điệu, đọc được số và ký hiệu toán, qua hậu kỳ phòng thu (nén,
lọc xì, cân âm lượng theo BS.1770). Không cần mạng, không cần khoá, chi phí
**0₫**, và **tất định** — cùng kịch bản ra cùng tệp âm thanh. Video xuất ra
AVI (MJPEG + PCM), mở bằng VLC hay Windows Media Player, **không cần ffmpeg**.

**Không được:** giọng người thật thu phòng. Không bộ tổng hợp nào ở đây bằng
được một phát thanh viên. Đó là giới hạn có thật, nói thẳng chứ không gọi
tránh đi.

Nên xưởng để sẵn một đường **thay giọng theo từng chương**: ai có bản thu
người thật thì nạp vào đúng chương, chương nào chưa có thì dùng giọng máy.
Không phải chọn một trong hai. Kịch bản lời đọc xuất ra `.txt` kèm nhịp đọc
đề xuất (150 từ/phút) để gửi thẳng cho phát thanh viên.

```bash
GET  /video-huong-dan/danh-muc          # 10 màn × 4 vai
GET  /video-huong-dan/kich-ban.txt      # ?man=&vai= — gửi đi thu giọng
POST /video-huong-dan/dung              # dựng một video thật
```

## 8n. Đưa lên Cloudflare — và nói rõ cái gì KHÔNG lên được

```bash
npm run cloudflare                                  # dựng + tự soi sản phẩm
npx wrangler pages deploy cloudflare/dist --project-name gitamath
npm run zip                                         # xuất hai gói .zip
```

### Hai gói .zip, và phép đếm lại

`npm run zip` xuất **hai** gói, vì chúng dùng cho hai việc khác nhau:

| Gói | Dùng để |
|---|---|
| `gitamath-cloudflare-<ngày>.zip` | Thả thẳng vào Cloudflare Pages |
| `gitamath-ma-nguon-<ngày>.zip` | Dựng lại, chạy máy chủ, sửa mã |

Một gói nén **thiếu tệp không báo lỗi** — nó mở ra bình thường, chỉ là khi
chạy thì thiếu một mô-đun, và lúc ấy đã ở trên máy người khác. Kho này từng
mất nguyên một tầng dữ liệu đúng theo kiểu ấy.

Nên gói xuất xong thì **mở lại, liệt kê, đối chiếu từng đường dẫn** với danh
sách nguồn — và danh sách nguồn của gói mã nguồn là `git ls-files`, không phải
cây thư mục trên đĩa: thứ nằm trên đĩa mà git không theo dõi thì clone sạch
cũng không có, nên đem nó vào gói là **giấu đi một lỗi thay vì sửa**. Thiếu
một tệp là thoát với mã 1.

### Cái gì lên được, cái gì không

Cloudflare Pages phục vụ **tệp tĩnh**: không chạy tiến trình Node lâu dài,
không có hệ tệp ghi được, không giữ trạng thái giữa hai yêu cầu. Hệ này thì
ngược lại. Nên nói cho rõ, vì đây là chỗ dễ hứa quá tay nhất:

| Lên được Cloudflare | Cần máy chủ Node thật |
|---|---|
| Trang giới thiệu dựng sẵn từ số liệu thật | Đăng nhập, tài khoản, phiên |
| Sơ đồ chương trình dạng SVG tĩnh | Trợ lý trò chuyện (đọc dữ liệu học viên) |
| 40 kịch bản lời đọc | Dựng video, dựng bài giảng |
| Mọi tệp giao diện | Mọi thứ ghi xuống đĩa |

**Nối hai phần:** chạy máy chủ Node ở đâu cũng được — kể cả một máy trong văn
phòng. `cloudflared tunnel --url http://localhost:9999` đưa nó ra ngoài mà
không phải mở cổng nào trên tường lửa. Trỏ một tên miền con vào đường hầm ấy,
rồi mở ba dòng đã dựng sẵn trong `dist/_redirects`.

### Kịch bản dựng tự soi lại sản phẩm

Nó không chỉ sinh tệp rồi báo xong — một trang tĩnh hỏng thì hỏng **im lặng**,
nó vẫn trả về 200. Nên sau khi sinh, kịch bản soi lại: có kịch bản lọt vào
không, có đường nào gọi ra ngoài không, có tệp nào rỗng không, có **chuỗi nào
trông giống khoá bí mật** không, có liên kết nội bộ nào đứt không, và mọi câu
có qua được luật nội dung không. Một mục không đạt là **từ chối hoàn tất**.

`wrangler.toml` **không khai biến môi trường nào** — cố ý: không bí mật nào đi
lên Cloudflare thì không bí mật nào rò từ Cloudflare.

### Số liệu đóng băng, và ngày đóng băng in trên trang

Trang tĩnh mang số liệu của **ngày dựng**, và mỗi trang in rõ ngày ấy. Một con
số không ghi ngày là một con số không kiểm lại được.

## 9. API — 394 endpoint, 43 nhóm (11 cổng công khai)

`CORE · ORCHESTRATOR · AGENTS · GOVERNANCE · COMPLIANCE · INTELLIGENCE · OPS ·
SUPERADMIN · RECOVERY · RESILIENCE · COMMAND · CURRICULUM · LEARNER · CYCLE ·
ACCOUNTS · ORG · IO · AUTHORING · ASSESS · INSIGHT · REPORT · VOICE · FILM ·
PHOTO · BAIGIANG · KHOAHOC · NGONNGU · PHATAM · KHAOSAT · CAYTIEN · QUYTRINH ·
TRAINGHIEM · CHIENLUOC · ANTOAN · ANTOANVIEC · SODO3D · TROCHUYEN · TIEPTHI ·
VIDEOHD · CRM · CRMSUPER · KHUONMAT · KHUONMATVAO`

Phân bố quyền thực tế: **11 công khai · 44 học viên · 86 giáo viên · 253 quản trị**.
Nhóm chưa khai mức rơi vào `quan-tri` — mặc định là ĐÓNG, không phải mở.

Nhánh `/scientist/*` (không phải `/math/*` — nhánh đó đã thuộc bộ xác minh học
liệu): `analyze · plan · solve · present · worksheet · method · methods ·
cross-subject · normalize · validate`. Nhánh `/lesson/*`: `status · forms ·
match · outline · build · jobs · :id · :id/video`. Nhánh `/lang/*`: `status ·
check · normalize · readability · syllables · sentences · vocabulary/:lop ·
term · lesson · audit-bank · diagnose/:learnerId`. Trang công khai chiếm các
đường dẫn `/`, `/chuong-trinh`, `/lo-trinh`, `/phuong-phap`, `/cong-cu`,
`/sitemap.xml`, `/robots.txt`; `/api` luôn trả JSON.

Mã lỗi nghiệp vụ được ánh xạ sang **mã HTTP đúng nghĩa** (401/403/404/405/409/410/429),
không trả 200 kèm `{ok:false}`.

## 10. Kiểm chứng

```
npm test          998/998 đạt
npm run verify    159/159 mục đạt
npm run smoke     394/394 lượt gọi API đạt
npm run cloudflare 6/6 mục soi sản phẩm tĩnh đạt
npm run zip       4/4 mục đếm lại gói đạt
node bin/gita.js to-thanh-tra --ca-ngay   10/10 lượt · 80 mục soi · đạt 100%
```

`npm run verify` không in số liệu trang trí — mỗi mục gọi thẳng vào mã đang chạy và
in ra con số thực tế, có mục nào trượt thì thoát với mã 1 (dùng được trong CI).

## 11. Cấu trúc

```
src/
  kernel/       store · queue · event bus · event store
  data/         lưu trữ bền vững (ảnh chụp + nhật ký ghi trước) · nhập/xuất CSV
  lang/         chuẩn tiếng Việt: chinhta (luật cấu tạo âm tiết, đặt dấu thanh)
                docde (độ đọc theo lớp) · tuvung (thuật ngữ theo lớp)
                scientist: soát bốn cấp · bài học bốn kỹ năng · soi cả kho
  web/          trang công khai dựng sẵn ở máy chủ · tầng SEO tự chấm lại
                nha/: khu riêng của gia đình · chong-gia-mao: vé CSRF cho biểu
                mẫu, HMAC của token phiên, không lưu thêm bảng nào
                cay-hinh: vẽ cây giá trị bằng SVG tất định — 50 quả đọc thẳng
                từ lưới 50 ô, không một số ngẫu nhiên nào
  math/         bộ đọc biểu thức (không eval) · xác minh toán học
                kyhieu: chuẩn ký hiệu quốc tế hai chiều LaTeX ↔ Unicode phẳng
                vanphong: soát lối viết sách giáo khoa · thamdinh: bốn cấp
                scientist: đọc vị đề · pháp đồ · đa cách giải · phiếu học tập
  school/       trường · lớp · nhóm
  report/       báo cáo học viên · lớp · hệ thống
  governance/   hiến pháp 50 điều · HEART · guardrails · safe-meta-prompt · nhật ký móc xích
  ai/           định tuyến LLM · bandit Thompson có seed · cache theo phân loại · Entropy MoE
  agents/       bảng phân công · roster 500 · runtime 9 cổng kiểm soát · KPI · công suất · điều độ
  security/     mật mã · két Super Admin 15 tầng · khôi phục · ảnh chụp phiên bản · chống nhiễu
  curriculum/   phân cấp dạng–độ–tầng–khối · chuẩn lên tầng 7 tiêu chí
  content/      kho học liệu · bản thiết kế bài · bộ sinh có ràng buộc · gieo kho
                chống trùng 3 lớp · ký hiệu · chấm 5★
  learner/      telemetry · mastery BKT+SM-2 · chân dung · chu kỳ 90 ngày
                chẩn đoán đầu vào · phiên luyện · thi cuối tầng · hành vi · lộ trình
  command/      tổng động viên · chất vấn chéo · hợp đồng giao việc
  voice/        âm vị học tiếng Việt · đọc số và đọc toán · tổng hợp formant
                500 hồ sơ giọng · nhà cung cấp thay thế · kịch bản giảng bài
                lớp pháp lý (đồng ý · dấu AI · nhật ký · hạn lưu trữ · quyền xoá)
                dsp: hậu kỳ phòng thu bằng JavaScript (biquad · nén song song ·
                de-esser · bão hoà hài · hồi âm · BS.1770 · limiter)
                music: nhạc lý · đọc và ghi tệp MIDI · tách bè · dịch quãng giọng
                singing: hát tiếng Việt (rung giọng · luyến · nét thanh điệu)
                pipelines/: ba tuyến nói · hát · đổi giọng hát
  photo/        đọc/ghi PNG và JPEG baseline tự viết · xử lý ảnh (Lanczos-3,
                mặt nạ làm sắc, chỉnh màu, đường tông phim, hạt, tối góc)
                chu: phông 5×7 tự dựng, chữ Việt có dấu + 60 ký hiệu toán
                8 hồ sơ máy ảnh + 6 giả lập phim bằng tham số · bố cục tất định
                soát 7 phép đo · cổng sinh ảnh miễn phí và chạy tại máy
  film/         đọc kịch bản bằng luật · hồ sơ nhân vật có khoá ngoại hình
                phân cảnh đo theo tiếng nói thật · chọn mô hình và dự toán
                soát chất lượng trước khi tiêu tiền · bảng phân cảnh SVG
                đường tiếng + phim nháp + EDL + kịch bản ffmpeg
                avi: tự ghi container RIFF/AVI (MJPEG + PCM), không cần ffmpeg
  nexus/        bài giảng một lệnh: khung hình (slide) · soạn và dựng
                diem-cham: chuỗi giá trị 8 chặng, đo trên dữ liệu học viên thật
  khao-sat/     sáu công cụ (DISC · MBTI · thần số · tâm lý học đường · phương
                pháp học · năng lực môn theo mốc năm học) · hien-phap-do: luật
                đo đứng trên cả sáu · phan-tich: sáu mặt, không gộp một chỉ số
                lo-trinh-ca-nhan: sáu chặng, mỗi chặng một cửa · kho: luật riêng
                tư từng công cụ, gọt câu trả lời thô trước khi lưu
  cay-tien/     NỘI BỘ ĐIỀU HÀNH — khung 8 năng lực · phân loại 5 nhóm gia đình
                buc-tuong: hai lớp chặn, quét 22 cụm từ cấm trên mọi trang công
                khai của cả bốn vai
  nha/          cay-gia-tri: năm tầng mặt khách hàng — thứ gia đình ĐƯỢC BIẾT
  matran/       ma trận 5 hệ chuẩn × 10 tầng · cay-50: lưới 5 tầng × 10 cấp,
                15 mốc cứng đo bằng máy, mốc mềm tự khai giới hạn
  dong-tien/    so-cai: bảng giá · ngân sách chặn cứng · khoản chưa nối nguồn
                được nêu tên trước con số, và không được bịa số
  thanh-tra/    tổ 10 trợ lý AI × 10 lượt mỗi ngày · phep-soi: 10 phép soi đọc
                thẳng trạng thái hệ thống đang chạy
  quy-trinh/    so-tay: 8 kịch bản trực sự cố — dấu hiệu · làm ngay · ai quyết ·
                CẤM LÀM; nối thẳng với mã thanh tra viên
  an-toan/      cong-tuoi: cổng tuổi + đồng ý người giám hộ, mặc định ĐÓNG
                nen-mong: quét kho mã tìm mô-đun được gọi mà không tồn tại
                khoa-rieng: mỗi việc một khoá HKDF, từ chối bí mật yếu
                soi-thiet-ke: bảy câu hỏi trước khi dựng tính năng mới
                khuon-mat: WebAuthn/FIDO2 — máy chủ chỉ giữ khoá công khai
                cbor: bộ đọc CBOR tối giản, từ chối khoá trùng và byte thừa
                chong-lua-dao: 6 việc nguy hiểm đòi quét mặt lại · 7 dấu hiệu
                tấn công, mỗi dấu hiệu nói rõ lý do; KHÔNG chấm điểm tin cậy
  chien-luoc/   thẻ điểm cân bằng: ban-do (4 viễn cảnh, 5 luật nhân quả cưỡng
                chế bằng máy) · thuoc-do (cấm đặt chỉ tiêu cho thứ chưa đo được)
                rui-ro (biện pháp phải là tệp mã có thật) · an-toan (chặn cứng
                phía chi; KHÔNG tự chứng nhận pháp lý) · ra-soat (ngày/tháng/quý)
  trai-nghiem/  hệ thống trải nghiệm khách hàng: khung 10 tầng · 40 tình huống ·
                5 kênh có cổng chặn · lưới 10.000 điểm chạm ba mức nền/tốt/wow
                trao-quyen: 10 quyền có trần cứng · phuc-hoi: thang 10 bước
                từ chối nhảy bước · do-luong: NỘI BỘ, chỗ mù in trước con số
  tro-chuyen/   trợ lý trò chuyện chín cửa: loc-vao · hang-rao (nhắc lại) ·
                khien-viec (bảng việc, 6 công cụ cấm tuyệt đối) · canh-ra
                (che số riêng, chặn hứa hẹn và chẩn đoán) · linh-gac (so
                đường đi thật với kế hoạch ghi trước) · cong-chung (Ed25519)
                cong-cu: 12 công cụ đọc dữ liệu thật, khai phạm vi tĩnh
  tiep-thi/     nội dung viết từ số liệu thật: luat-noi-dung (4 luật cứng) ·
                do-thi (đồ thị tri thức dùng chung cho chat và nội dung) ·
                xuong-noi-dung (kho rỗng thì BỎ mảng, không in số 0) ·
                vong-cai-tien (chỉ đề xuất, không tự sửa trang) ·
                dung-trang-tinh (dựng HTML cho Cloudflare Pages)
  xuong-video/  video hướng dẫn theo vai: man-hinh (quét mã giao diện để suy
                mức quyền từng màn) · kich-ban (8 chương × 4 vai) · xuong
                (nối vào đường ống bài giảng sẵn có, giọng dựng tại máy)
  so-do-tu-duy/ sơ đồ tư duy ba chiều: khong-gian (xoay · chiếu phối cảnh ·
                xếp theo chiều sâu · mờ dần, viết tay không thư viện)
                so-do (dựng từ 104 dạng thật, ba kiểu: toàn cảnh · đường đi ·
                quanh một dạng) · ve (SVG tất định, không kịch bản, in được)
  crm/          ban quản trị quan hệ gia đình: 30 trợ lý AI chuyên môn · 4 cấp
                muc-tu-chu (cân rủi ro, hai luật cứng ngoài phép tính)
                dieu-phoi: 9 cửa mỗi lượt · cho-duyet: việc chạm ra ngoài
                luôn đòi người · kpi · quan-tri · lich-viec · trang quản trị
  api/          cong-quyen: bảng quyền bốn mức, nhóm chưa khai thì ĐÓNG
  orchestrator/ api/ observability/
kich-ban-mau/   kịch bản mẫu để chạy thử xưởng phim
ui/hanh-trinh.html  màn hình học viên: tầng đang ở · chặng chu kỳ 90 ngày ·
                buổi học hôm nay · tổng kết đo bằng số liệu thật
bin/gita.js     dòng lệnh: bai-giang · giai · soat · phieu · phuong-phap · dang
                tieng-viet · am-tiet · bai-ngon-ngu · soi-kho · thanh-tra
                phat-am · cap-am · doc-anh · khao-sat · to-thanh-tra
                dong-tien · quy-trinh · trai-nghiem · the-diem · an-toan
                diem-cham · cay-tien · ma-tran
ui/             cổng chọn vai · góc học tập · phòng giáo viên · trung tâm chỉ huy
                sơ đồ tư duy 3D (kéo chuột xoay, máy chủ dựng lại ảnh)
                khung hỏi trợ lý (hiện rõ vết: đã tra nguồn nào, ký sổ nào)
                xưởng bài giảng · bàn làm việc Toán · bàn soát tiếng Việt
                bộ dựng LaTeX tự viết · hạ tầng chung (biểu đồ SVG thuần)
                lớp nghe giảng (nút Nghe, chọn giọng, nhãn "Giọng AI")
desktop/        bản cài đặt Windows 64-bit (Electron + electron-builder)
scripts/        verify · roster-table · smoke · load-test
                dung-ban-windows · dung-ban-cloudflare · xuat-zip
cloudflare/     bản tĩnh cho Cloudflare Pages + _headers + _redirects
test/           998 test
RUNBOOK.md      sổ tay vận hành: cài đặt, việc hằng ngày, xử lý sự cố
```

## 12. Yêu cầu

Node.js ≥ 20.11 (khuyến nghị 22). Không có phụ thuộc runtime.
Electron + electron-builder chỉ cần khi đóng gói bản cài Windows.
