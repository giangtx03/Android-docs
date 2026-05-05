# Tiêu Chuẩn Xây Dựng Custom Component Trong Jetpack Compose

Để xây dựng một Custom Component có khả năng tái sử dụng cao, linh hoạt và dễ bảo trì trong Jetpack Compose, cần tuân thủ 5 nguyên tắc cốt lõi sau:

## 1. Luôn truyền `Modifier` làm tham số
Component không nên tự quyết định kích thước bên ngoài (padding ngoài, width, height) hay vị trí của chính nó. Hãy để màn hình gọi nó quyết định. Tham số `modifier` nên được đặt ở vị trí đầu tiên (hoặc thứ hai) và luôn có giá trị mặc định.

**✅ Code chuẩn:**
```kotlin
@Composable
fun MyCustomButton(
    text: String,
    modifier: Modifier = Modifier, // Luôn có giá trị mặc định là rỗng
    onClick: () -> Unit
) {
    Button(
        onClick = onClick,
        modifier = modifier // Áp dụng modifier từ ngoài truyền vào
    ) {
        Text(text)
    }
}
```

## 2. Sử dụng Slot API để tăng độ linh hoạt
Thay vì chỉ truyền các kiểu dữ liệu cơ bản (như String cho text, Int cho icon), hãy dùng Slot API (truyền `@Composable` lambda). Điều này giúp component nhận bất kỳ khối UI nào từ bên ngoài, tăng khả năng mở rộng.

✅ Code chuẩn:

```Kotlin
@Composable
fun CustomFilledButton(
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
    content: @Composable RowScope.() -> Unit // Nhận một khối UI bất kỳ
) {
    Button(
        onClick = onClick,
        modifier = modifier,
        content = content 
    )
}
```

## 3. State Hoisting (Đẩy State lên trên)
Custom Component nên `Stateless` (không giữ trạng thái). Nó chỉ nhận dữ liệu để hiển thị và báo cáo sự kiện khi người dùng tương tác, không nên tự thay đổi logic bên trong nó.

✅ Code chuẩn:

```Kotlin
@Composable
fun CustomCheckbox(
    isChecked: Boolean, // Nhận state từ ngoài
    onCheckedChange: (Boolean) -> Unit // Bắn sự kiện ra ngoài
) {
    Checkbox(checked = isChecked, onCheckedChange = onCheckedChange)
}
```

## 4. Tách biệt Styling và Theming
Không bao giờ hardcode màu sắc như `Color.Red`. Luôn gọi màu và kiểu chữ từ hệ thống Theme (`MaterialTheme.colorScheme, MaterialTheme.typography`) để tự động hỗ trợ Dark Mode và dễ dàng thay đổi sau này.

✅ Code chuẩn:
```Kotlin
@Composable
fun ErrorText(text: String, modifier: Modifier = Modifier) {
    Text(
        text = text,
        modifier = modifier,
        color = MaterialTheme.colorScheme.error, // Tự động đổi màu theo Theme
        style = MaterialTheme.typography.bodyMedium
    )
}
```
## 5. Cung cấp Preview chuẩn mực
Luôn tạo @Preview cho nhiều trạng thái (Light/Dark mode, Mặc định/Lỗi/Loading) để dễ dàng kiểm tra component mà không cần chạy máy ảo.

✅ Code chuẩn:
```Kotlin
@Preview(uiMode = Configuration.UI_MODE_NIGHT_YES, showBackground = true)
@Composable
private fun CustomFilledButtonPreview() {
    MyTheme {
        CustomFilledButton(onClick = {}) { Text("Nút Đăng Ký") }
    }
}
```