# 📐 Hướng dẫn Quản lý Kích thước và Layout trong Jetpack Compose

## Nguyên tắc cốt lõi: "Modifiers from the outside in"

Trong Jetpack Compose, một Component (ví dụ: `TransactionItem`, `SummaryCard`) **chỉ nên quản lý việc sắp xếp nội dung bên trong nó**. Kích thước tổng thể, vị trí đứng, và tỷ lệ chiếm màn hình của Component đó phải do **Component cha quyết định** thông qua việc truyền `Modifier` từ ngoài vào.

*   **❌ Sai lầm phổ biến:** Cố định `modifier = Modifier.fillMaxWidth()` ngay bên trong định nghĩa của một Component tái sử dụng.
*   **✅ Best Practice:** Truyền `modifier` vào Component và để màn hình cha (Screen) định đoạt kích thước dựa trên ngữ cảnh sử dụng.

---

## Phân loại nhóm Modifier quản lý kích thước

### 1. Nhóm bọc nội dung (Content-Driven)
Yêu cầu Component cha cấp một khung vừa đúng bằng nội dung bên trong, không quan tâm đến không gian dư thừa.

*   **`wrapContentWidth()` / `wrapContentHeight()` / `wrapContentSize()`**
    *   **Cơ chế:** Bỏ qua lực ép không gian từ Component cha (như `LazyVerticalGrid` hay `Row`), giữ nguyên kích thước gốc của phần tử con và thực hiện căn lề.
    *   **Ứng dụng:** Giữ cố định kích thước của `CategoryDropZone` (ví dụ `60.dp`) và buộc nó nằm chính giữa ô lưới khi nằm trong `LazyVerticalGrid`.
*   **`widthIn(min = ..., max = ...)` / `heightIn(...)`**
    *   **Cơ chế:** Cho phép nội dung co giãn nhưng bị giới hạn bởi kích thước tối thiểu và tối đa.
    *   **Ứng dụng:** Đảm bảo một Nút bấm (Button) luôn đạt chiều rộng tối thiểu (ví dụ `minWidth = 64.dp`) bất kể text bên trong ngắn cỡ nào.

### 2. Nhóm chiếm không gian (Parent-Driven)
Phụ thuộc hoàn toàn vào không gian trống mà Component cha cấp cho.

*   **`fillMaxWidth()`/ `fillMaxHeight()` / `fillMaxSize()`**
    *   **Cơ chế:** Chiếm toàn bộ không gian còn trống theo chiều ngang/dọc/cả hai.
    *   **Ứng dụng:** Thường dùng ở tầng lắp ráp Screen (như `HomeScreen`) để dàn đều danh sách hoặc thẻ ra sát mép màn hình.
*   **`weight(1f)`**
    *   **Cơ chế:** Chia tỷ lệ không gian còn trống cho các phần tử cùng cấp bên trong `Row` hoặc `Column`.
    *   **Ứng dụng:** Dùng để đẩy và căn lề linh hoạt. Ví dụ: Trong dòng giao dịch, cột tên chiếm `weight(1f)` sẽ tự động giãn ra và đẩy phần số tiền dạt sát sang mép phải.

### 3. Nhóm đo lường nội tại (Intrinsic Measurement)
Giải quyết các trường hợp các phần tử con cần đồng bộ kích thước với nhau trong một lần đo duy nhất của Compose.

*   **`height(IntrinsicSize.Min)` / `width(IntrinsicSize.Min)`**
    *   **Cơ chế:** Đo kích thước thực tế của phần tử lớn nhất bên trong, sau đó ép tất cả các phần tử cùng cấp còn lại phải có kích thước bằng phần tử đó.
    *   **Ứng dụng:** Tạo các đường kẻ phân cách (`VerticalDivider`) tự động kéo dài bằng đúng chiều cao của khối chữ bên cạnh (như đã áp dụng thành công trong `SummaryCard`).
*   **`IntrinsicSize.Max`**
    *   **Cơ chế:** Đo đạc không gian lớn nhất mà nội dung *có thể* chiếm nếu hoàn toàn không bị giới hạn (thường dùng để tính độ rộng tối đa trước khi một đoạn text bị ép xuống dòng).

---

## 🛠 Ví dụ

### Bước 1: Loại bỏ Modifier hạn chế ở tầng Component

*Nguyên tắc: Component con không được tự ý điền đầy màn hình nếu không được cha cho phép.*
```kotlin
// ❌ TRƯỚC KHI TỐI ƯU (Cố định fillMaxWidth bên trong)
@Composable
fun TransactionItem(modifier: Modifier = Modifier, state: TransactionItemUiState) {
    Row(
        // Lỗi: Tự ý bắt buộc phải chiếm toàn màn hình
        modifier = modifier.fillMaxWidth() 
    ) { 
        // ... nội dung UI
    }
}

// ✅ SAU KHI TỐI ƯU (Linh hoạt theo cha)
@Composable
fun TransactionItem(modifier: Modifier = Modifier, state: TransactionItemUiState) {
    Row(
        // Đúng: Giữ nguyên modifier, Component cha sẽ định đoạt
        modifier = modifier 
    ) { 
        // ... nội dung UI
    }
}
```

### Bước 2: Thiết lập kích thước từ tầng Screen (Cha)
*Nguyên tắc: Gọi fillMaxWidth() tại nơi lắp ráp giao diện tổng thể.*

```kotlin
@Composable
fun ProcessingTransactionList(transactions: List<TransactionItemUiState>) {
    LazyColumn(
        // Cha quyết định toàn bộ List chiếm hết chiều ngang
        modifier = Modifier.fillMaxWidth() 
    ) {
        items(transactions) { it ->
            TransactionItem(
                // Màn hình Cha quyết định từng Item bên trong cũng phải dàn hết mép
                modifier = Modifier.fillMaxWidth(), 
                state = it
            )
        }
    }
}
```