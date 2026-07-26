# Hướng dẫn Cập nhật từ Upstream (i3 GitHub)

Tài liệu này hướng dẫn cách cập nhật mã nguồn mới nhất từ kho chứa chính thức của i3 (GitHub upstream) về dự án này mà vẫn giữ nguyên các tính năng tùy biến đã phát triển (`floating_all` và `keep_empty_space`).

---

## Quy trình cập nhật (Step-by-Step)

### Bước 1: Thêm địa chỉ i3 gốc làm remote `upstream`
Bạn chỉ cần thực hiện lệnh này **một lần duy nhất** để liên kết project này với kho i3 gốc:
```bash
git remote add upstream https://github.com/i3/i3.git
```

### Bước 2: Tải các cập nhật mới nhất từ i3 gốc
```bash
git fetch upstream
```

### Bước 3: Gộp (Merge) hoặc Lắp ghép (Rebase) mã nguồn
Chọn một trong hai phương pháp sau để cập nhật:

#### Phương pháp 1: Dùng `git merge` (Đơn giản & an toàn nhất)
Tự động tạo một commit gộp để kéo toàn bộ thay đổi mới từ nhánh gốc của i3 (thường là nhánh `next`) vào nhánh hiện tại của bạn:
```bash
git merge upstream/next
```

#### Phương pháp 2: Dùng `git rebase` (Lịch sử commits sạch đẹp hơn)
Tạm thời nhấc các commits chứa tính năng tùy biến của bạn ra, cập nhật code mới từ i3 gốc vào trước, sau đó đặt lại các commits tùy biến lên trên cùng:
```bash
git rebase upstream/next
```

---

## Cách xử lý khi xảy ra xung đột (Merge Conflict)

Nếu i3 gốc và code tùy biến sửa đổi trên cùng một dòng của cùng một file, Git sẽ báo xung đột và tạm dừng quá trình merge/rebase.

### 1. Tìm các file bị xung đột
Chạy lệnh sau để kiểm tra:
```bash
git status
```
Các file có trạng thái `both modified` màu đỏ là các file cần xử lý.

### 2. Giải quyết xung đột thủ công
Mở các file xung đột bằng text editor (như VS Code). Bạn sẽ thấy các khối xung đột được đánh dấu như sau:
```c
<<<<<<< HEAD
// Tính năng tùy biến của dự án này (keep_empty_space...)
=======
// Các thay đổi mới cập nhật từ i3 gốc trên GitHub
>>>>>>> upstream/next
```
- Hãy chỉnh sửa code sao cho kết hợp hài hòa cả hai thay đổi (hoặc giữ lại tính năng của chúng ta nếu phần code gốc bị thay thế không cần thiết).
- Xóa các dòng đánh dấu (`<<<<<<<`, `=======`, `>>>>>>>`).

### 3. Tiếp tục quá trình gộp
Lưu file đã sửa đổi lại, chạy lệnh staging:
```bash
git add <tên_file_đã_sửa>
```
Sau đó tiếp tục hoàn tất:
- **Nếu đang dùng `git merge`**:
  ```bash
  git commit -m "Merge updates from upstream"
  ```
- **Nếu đang dùng `git rebase`**:
  ```bash
  git rebase --continue
  ```

---

## Bước cuối cùng: Biên dịch và chạy thử bản mới
Sau khi gộp mã nguồn thành công, hãy tiến hành biên dịch lại để cập nhật binary hệ thống:
```bash
meson compile -C build
sudo rm /usr/bin/i3 && sudo cp build/i3 /usr/bin/i3
i3-msg restart
```

---

## Hướng dẫn Đưa dự án lên GitHub cá nhân

Để lưu trữ dự án tùy biến này lên tài khoản GitHub cá nhân của bạn mà vẫn giữ nguyên lịch sử commits và các nhánh (nhánh `next`), hãy làm theo các bước sau:

### Bước 1: Tạo một Repository trống trên GitHub của bạn
1. Truy cập vào GitHub cá nhân của bạn và chọn **New Repository**.
2. Đặt tên (ví dụ: `i3-custom`).
3. **QUAN TRỌNG**: Không tick vào bất kỳ lựa chọn nào như *Add a README*, *Add .gitignore*, hoặc *Choose a license* (giữ repository hoàn toàn trống rỗng).
4. Nhấn **Create repository**.
5. Copy đường dẫn repository cá nhân của bạn (dạng HTTPS hoặc SSH, ví dụ: `https://github.com/username/i3-custom.git`).

### Bước 2: Chuyển remote gốc của i3 thành `upstream`
Hiện tại, remote tên `origin` đang trỏ tới kho gốc của i3. Chúng ta sẽ đổi tên nó thành `upstream` để giải phóng tên `origin` cho GitHub cá nhân của bạn:
```bash
git remote rename origin upstream
```

### Bước 3: Thêm GitHub cá nhân của bạn làm remote `origin` mới
Chạy lệnh này (thay thế URL bằng link repository cá nhân bạn đã copy ở Bước 1):
```bash
git remote add origin https://github.com/username/i3-custom.git
```

### Bước 4: Đẩy toàn bộ code và các nhánh lên GitHub cá nhân
Đẩy tất cả các nhánh và tags lên tài khoản cá nhân:
```bash
# Đẩy tất cả các nhánh (bao gồm nhánh phát triển chính)
git push -u origin --all

# Đẩy tất cả tags (phiên bản)
git push -u origin --tags
```

Từ lần sau, khi bạn thực hiện thay đổi và muốn lưu lên GitHub cá nhân, bạn chỉ cần gõ:
```bash
git add .
git commit -m "mô tả thay đổi"
git push origin
```
Và khi muốn cập nhật từ i3 gốc trên GitHub, bạn chỉ cần gõ `git fetch upstream` rồi gộp như hướng dẫn ở phần trên!

