# Kiến Trúc Theme Trong Jetpack Compose (Material + Custom Colors)

## Tại Sao Nên Dùng MaterialTheme Kết Hợp Custom Colors?
*   **Tận dụng Component có sẵn:** Các UI component của Google (`Button`, `Card`, `Scaffold`...) đều tự động tìm kiếm và sử dụng màu từ `MaterialTheme.colorScheme`.
*   **Tiết kiệm thời gian:** Không phải tự viết lại các hiệu ứng phức tạp (Ripple), quản lý trạng thái (Pressed, Disabled) hay Accessibility.
*   **Mở rộng linh hoạt:** Tạo `CompositionLocal` để bổ sung các màu đặc thù (VD: Màu thu nhập, chi tiêu) mà Material không hỗ trợ sẵn.

## Hướng Dẫn Cài Đặt (Ví dụ: Ứng dụng Ví Điện Tử Mivio)

### Bước 1: Khai báo Data Class cho Custom Colors
Tạo một class để chứa các màu đặc thù, sau đó tạo `CompositionLocal` để phân phối nó.
```kotlin
import androidx.compose.runtime.Immutable
import androidx.compose.runtime.staticCompositionLocalOf
import androidx.compose.ui.graphics.Color

@Immutable
data class MivioCustomColors(
    val income: Color = Color.Unspecified,
    val expense: Color = Color.Unspecified,
    // ... thêm các màu danh mục, trạng thái khác
)

val LocalMivioCustomColors = staticCompositionLocalOf { MivioCustomColors() }
```

### Bước 2: Thiết lập Theme kết hợp hỗ trợ Dark/Light Mode
```Kotlin
import androidx.compose.foundation.isSystemInDarkTheme
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.darkColorScheme
import androidx.compose.material3.lightColorScheme
import androidx.compose.runtime.Composable
import androidx.compose.runtime.CompositionLocalProvider

// 1. Map Material Colors (Sáng & Tối)
private val LightColorScheme = lightColorScheme(
    primary = Primary,
    background = BackgroundLight,
    // ...
)
private val DarkColorScheme = darkColorScheme(
    primary = Primary,
    background = BackgroundDark,
    // ...
)

// 2. Map Custom Colors (Sáng & Tối)
private val LightCustomColors = MivioCustomColors(
    income = IncomeLight,
    expense = ExpenseLight
)
private val DarkCustomColors = MivioCustomColors(
    income = IncomeDark,
    expense = ExpenseDark
)

// 3. Hàm Theme chính
@Composable
fun MivioTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    content: @Composable () -> Unit
) {
    val colorScheme = if (darkTheme) DarkColorScheme else LightColorScheme
    val customColors = if (darkTheme) DarkCustomColors else LightCustomColors

    CompositionLocalProvider(
        LocalMivioCustomColors provides customColors
    ) {
        MaterialTheme(
            colorScheme = colorScheme,
            typography = Typography, 
            content = content
        )
    }
}

// 4. Object hỗ trợ gọi màu nhanh
object MivioTheme {
    val customColors: MivioCustomColors
        @Composable
        get() = LocalMivioCustomColors.current
}
``` 

Bước 3: Cách sử dụng trong Giao diện
Bọc ứng dụng bằng MivioTheme tại MainActivity, sau đó ở các Component:

Dùng màu chuẩn: `MaterialTheme.colorScheme.primary`

Dùng màu custom: `MivioTheme.customColors.income`

```Kotlin
@Composable
fun TransactionItem(isIncome: Boolean) {
    Text(
        text = "50,000 VND",
        color = if (isIncome) MivioTheme.customColors.income else MivioTheme.customColors.expense
    )
}
```