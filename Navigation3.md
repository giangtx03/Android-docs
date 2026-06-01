
## 1. Bản chất của NavKey và NavDisplay

Trong kiến trúc của Navigation 3, em chỉ cần làm chủ 3 trụ cột sau:

### a. `NavKey`: Định danh Strongly-Typed

`NavKey` là một interface đóng vai trò là một định danh an toàn về kiểu (type-safe) và có thể tuần tự hóa (serializable) cho một điểm đến.

* **Cách hoạt động:** Thay vì dùng String như `home_screen/{id}` dễ gây lỗi runtime. chúng ta sẽ dùng `object` cho màn hình không có tham số và `data class` cho màn hình có tham số.
* **Serialization:** Bắt buộc phải được annotate với `@Serializable` (dùng `kotlinx.serialization`). Điều này giúp hệ thống tự động lưu và khôi phục trạng thái khi app bị hệ thống kill (Process Death).

### b. `NavBackStack`: Trạng thái (State) thuần túy

Nó chỉ đơn giản là một danh sách mutable chứa các `NavKey` đại diện cho lịch sử điều hướng. Khác với Navigation 2 giấu nhẹm state đi, ở đây chúng ta đẩy (push) hoặc lấy (pop) các item trực tiếp như thao tác với một List thông qua `add()` hoặc `removeLast()`.

### c. `NavDisplay`: Trái tim của UI (Thay thế NavHost)

Nếu `NavBackStack` lưu trữ trạng thái, thì `NavDisplay` render giao diện.

* **Cách hoạt động:** `NavDisplay` lắng nghe sự thay đổi của `NavBackStack`. Bất cứ khi nào stack thay đổi, nó lấy `NavKey` trên cùng (top entry), quét qua **EntryProvider** (một DSL mapping giữa Key và UI) và hiển thị Composable tương ứng.
* **Sự linh hoạt:** Vì state được tách biệt khỏi UI container, nên có thể đặt `NavDisplay` ở bất cứ đâu trong cây Compose. Nếu làm giao diện Split-screen trên Tablet, chúng ta hoàn toàn có thể chạy hai `NavDisplay` song song với hai `NavBackStack` độc lập cực kỳ dễ dàng.

---

## 2. Triển khai Navigation 3 trên Multi-Module

Nguyên tắc vàng của module hóa là: **Các feature module không được biết về nhau, chỉ có Composition Root (App module) mới biết tất cả để gắn kết chúng.**

### Bước 1: Setup tại Feature Module

Các Feature module không cần phụ thuộc vào một đồ thị chung. Mỗi feature sẽ tự định nghĩa `NavKey` và một `EntryProviderScope` builder riêng.

```kotlin
// module: feature:home

// 1. Định nghĩa Keys nội bộ của feature
@Serializable
data object HomeKey : NavKey

@Serializable
data class ProductDetailKey(val productId: String) : NavKey

// 2. DSL Builder cung cấp các màn hình
fun EntryProviderScope<NavKey>.homeEntryBuilder(
    onNavigateToProduct: (String) -> Unit,
    onBack: () -> Unit
) {
    entry<HomeKey> {
        HomeScreen(onProductClick = onNavigateToProduct)
    }
    
    entry<ProductDetailKey> { entry ->
        // Ép kiểu an toàn để lấy tham số
        val key = entry.key as ProductDetailKey 
        ProductDetailScreen(
            productId = key.productId, 
            onBackClick = onBack
        )
    }
}

```

### Bước 2: Ráp nối tại App Module (Composition Root)

Module `app` sẽ implementation các feature modules. Tại đây, tạo State tổng và cung cấp nó cho `NavDisplay`.

```kotlin
// module: app

@Composable
fun MainAppScreen() {
    // 1. Khởi tạo State với màn hình mặc định
    val backStack = rememberNavBackStack<NavKey>(HomeKey)

    // 2. Centralize các hàm điều hướng (hoặc đưa vào một lớp Router/Navigator)
    val navigateToProduct = { id: String -> backStack.add(ProductDetailKey(id)) }
    val popBack = { backStack.removeLast() }

    // 3. Render giao diện
    NavDisplay(
        backStack = backStack,
        entryProvider = entryProvider {
            // Gắn kết các UI builder từ mọi feature modules vào đây
            homeEntryBuilder(
                onNavigateToProduct = navigateToProduct,
                onBack = popBack
            )
            
            // authEntryBuilder(...)
            // settingsEntryBuilder(...)
        }
    )
}

```

---

Tuyệt vời! Việc em muốn đào sâu vào phần này chứng tỏ em đang nhìn nhận Navigation không chỉ là "chuyển màn hình", mà là một **kiến trúc luồng dữ liệu (Data Flow Architecture)**.

Anh sẽ làm rõ từng ý trong phần 3 bằng code thực tế để em dễ hình dung cách chúng ta áp dụng vào dự án lớn nhé.

---

### 3.1. Thiết kế lớp `Router` (Decoupling Navigation khỏi UI & ViewModel)

Nếu truyền các lambda `onNavigateTo...` qua nhiều tầng Composable, nó sẽ gây ra tình trạng "Prop Drilling" (truyền ngược lambda quá sâu). Nếu gọi thẳng trong ViewModel thì ViewModel lại bị dính dáng đến Compose.

**Giải pháp:** Tạo một `AppNavigator` độc lập dựa trên `SharedFlow`.

**Bước 1: Định nghĩa Navigator (Nằm ở module `core:navigation`)**

```kotlin
// Định nghĩa các action điều hướng cơ bản
sealed interface NavAction {
    data class Navigate(val key: NavKey) : NavAction
    data object Pop : NavAction
    data object PopToRoot : NavAction
}

// Interface để các ViewModel gọi
interface AppNavigator {
    val navActions: SharedFlow<NavAction>
    fun navigateTo(key: NavKey)
    fun popBack()
}

// Implementation
class AppNavigatorImpl : AppNavigator {
    private val _navActions = MutableSharedFlow<NavAction>(extraBufferCapacity = 1)
    override val navActions = _navActions.asSharedFlow()

    override fun navigateTo(key: NavKey) {
        _navActions.tryEmit(NavAction.Navigate(key))
    }

    override fun popBack() {
        _navActions.tryEmit(NavAction.Pop)
    }
}

```

**Bước 2: Sử dụng trong ViewModel (Feature Module)**
ViewModel giờ đây hoàn toàn "sạch", chỉ việc gọi interface thông qua Dependency Injection (Dagger/Hilt hoặc Koin).

```kotlin
@HiltViewModel
class HomeViewModel @Inject constructor(
    private val navigator: AppNavigator
) : ViewModel() {

    fun onProductClicked(productId: String) {
        // ViewModel không biết gì về Compose hay NavBackStack
        navigator.navigateTo(ProductDetailKey(productId))
    }
}

```

**Bước 3: Lắng nghe ở tầng AppRoot (App Module)**
Ở Composable cao nhất, chúng ta hứng `navActions` và thực thi lên `NavBackStack`.

```kotlin
@Composable
fun MainAppScreen(navigator: AppNavigator) {
    val backStack = rememberNavBackStack<NavKey>(HomeKey)

    // Lắng nghe event từ ViewModel và update backStack
    LaunchedEffect(Unit) {
        navigator.navActions.collect { action ->
            when (action) {
                is NavAction.Navigate -> backStack.add(action.key)
                is NavAction.Pop -> backStack.removeLast()
                is NavAction.PopToRoot -> {
                    // Xóa hết, chỉ giữ lại màn hình đầu tiên
                    while (backStack.size > 1) backStack.removeLast()
                }
            }
        }
    }

    NavDisplay(backStack = backStack, entryProvider = entryProvider { ... })
}

```

---

### 3.2. Cấu hình đa hình (Polymorphism) để chống Process Death

Khi hệ thống thiếu RAM và kill app, Compose sẽ cố gắng serialize (lưu) `NavBackStack` vào `Bundle`.
Vì `NavBackStack` chứa kiểu `NavKey` (là một Interface), `kotlinx.serialization` không thể biết được Key cụ thể bên trong là `HomeKey` hay `ProductDetailKey` để mà lưu.

**Cách setup chuẩn:**

```kotlin
// Gom tất cả các module có chứa NavKey lại
val appSerializersModule = SerializersModule {
    polymorphic(NavKey::class) {
        subclass(HomeKey::class, HomeKey.serializer())
        subclass(ProductDetailKey::class, ProductDetailKey.serializer())
        // Cứ có Key mới là phải khai báo vào đây
    }
}

@Composable
fun MainAppScreen() {
    // Truyền cấu hình vào lúc khởi tạo backStack
    val backStack = rememberNavBackStack<NavKey>(
        initialEntry = HomeKey,
        savedStateConfiguration = SavedStateConfiguration(
            serializersModule = appSerializersModule
        )
    )
    // ...
}

```

*Tip cho Senior:* Trong một app multi-module siêu lớn, có thể tạo một cơ chế auto-generate (dùng KSP - Kotlin Symbol Processing) để tự động gom các class này lại, khỏi lo việc quên đăng ký serializer.

---

### 3.3. Tối ưu Bottom Navigation với Đa Backstack (Multiple Backstacks)

Ở Navigation 2, việc giữ trạng thái (ví dụ: đang đọc dở bài viết ở Tab Home, bấm qua Tab Profile, quay lại Tab Home vẫn phải giữ nguyên bài viết đó) đòi hỏi setup `saveState` và `restoreState` khá nhọc.
Với Navigation 3, State tự nắm giữ.

**Kiến trúc:** Dùng 1 BackStack cho mỗi Tab.

```kotlin
enum class BottomTab { HOME, SEARCH, PROFILE }

@Composable
fun MainScreenWithBottomNav() {
    // 1. Lưu trạng thái tab hiện tại
    var currentTab by rememberSaveable { mutableStateOf(BottomTab.HOME) }

    // 2. Tạo 3 backstack hoàn toàn ĐỘC LẬP
    val homeStack = rememberNavBackStack<NavKey>(HomeRootKey)
    val searchStack = rememberNavBackStack<NavKey>(SearchRootKey)
    val profileStack = rememberNavBackStack<NavKey>(ProfileRootKey)

    // 3. Chọn backstack nào sẽ được hiển thị dựa trên Tab
    val activeBackStack = when (currentTab) {
        BottomTab.HOME -> homeStack
        BottomTab.SEARCH -> searchStack
        BottomTab.PROFILE -> profileStack
    }

    Scaffold(
        bottomBar = {
            MyBottomNavigation(
                selectedTab = currentTab,
                onTabSelected = { selected -> currentTab = selected }
            )
        }
    ) { padding ->
        // 4. Render! NavDisplay chỉ việc render cái activeBackStack.
        // Khi em đổi tab, instance activeBackStack thay đổi, NavDisplay tự update UI
        // Các stack kia nằm im trong bộ nhớ, không bị hủy (State được bảo toàn 100%).
        NavDisplay(
            modifier = Modifier.padding(padding),
            backStack = activeBackStack,
            entryProvider = entryProvider {
                // Khai báo builder cho toàn bộ các màn hình ở đây...
            }
        )
    }
}

```