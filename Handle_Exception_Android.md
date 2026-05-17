# Mô hình hóa chi tiết, trực quan hóa luồng dữ liệu lỗi (Error Flow)
### 1. Sơ đồ mô hình hóa Luồng xử lý Lỗi (Error Flow Model)
Mô hình này chuyển đổi các ngoại lệ kỹ thuật thô (Raw Exceptions) thành các trạng thái giao diện an toàn (UiState) thông qua cơ chế bọc lỗi (Error Wrapping). 

```
[Internet / Server] 
         │  (IOException / HttpException 500, 401)
         ▼
 ┌────────────────────────────────────────────────────────┐
 │ DATA MODULE / LAYER                                    │
 │  ├─ OkHttp Interceptor: Bắt lỗi 401, tự động làm mới   │
 │  └─ BaseRepository (safeApiCall): try-catch toàn bộ    │
 └───────────────────────┬────────────────────────────────┘
                         │  (Chuyển đổi thành NetworkResult sealed class)
                         ▼
 ┌────────────────────────────────────────────────────────┐
 │ PLATFORM MODULE (Network Monitor)                      │
 │  └─ Lắng nghe phần cứng ──► Phát StateFlow<Boolean>    │
 └───────────────────────┬────────────────────────────────┘
                         │
                         ▼
 ┌────────────────────────────────────────────────────────┐
 │ PRESENTATION MODULE                                    │
 │  ├─ ViewModel: Nhận NetworkResult ──► Chuyển thành     │
 │  │             UiState.Error + Quản lý viewModelScope  │
 │  └─ UI (Compose/Activity): Vẽ Banner / Nút Thử lại     │
 └────────────────────────────────────────────────────────┘
 ```
### 2. Chi tiết Mô hình hóa từng Module
2.1. Data Module: Tầng cô lập và chuyển đổi lỗi (Error Isolation)Tầng này đóng vai trò như một "bộ lọc", chuyển đổi các Exception hệ thống thành dạng dữ liệu có cấu trúc định danh (NetworkResult).Định nghĩa cấu trúc dữ liệu lỗi toàn cục:
``` Kotlin
sealed interface NetworkResult<out T> {
    data class Success<out T>(val data: T) : NetworkResult<T>
    data class Error(val code: Int, val message: String) : NetworkResult<Nothing>
    data class Exception(val e: Throwable) : NetworkResult<Nothing>
}
```
Triển khai BaseRepository xử lý tập trung:

``` Kotlin
open class BaseRepository {
    suspend fun <T> safeApiCall(apiCall: suspend () -> Response<T>): NetworkResult<T> {
        return try {
            val response = apiCall()
            val body = response.body()
            if (response.isSuccessful && body != null) {
                NetworkResult.Success(body)
            } else {
                NetworkResult.Error(code = response.code(), message = response.message())
            }
        } catch (e: HttpException) {
            NetworkResult.Error(code = e.code(), message = e.message())
        } catch (e: IOException) {
            // Lỗi mất kết nối mạng hoặc timeout
            NetworkResult.Exception(e)
        } catch (e: Throwable) {
            // Các lỗi không xác định khác
            NetworkResult.Exception(e)
        }
    }
}
```
2.2. Platform Module: 
Giám sát trạng thái hệ thốngModule này độc lập và cung cấp thông tin phần cứng cho Presentation nhằm phục vụ tính năng Auto-Resume (Tự động tải lại khi có mạng).
``` Kotlin
interface NetworkMonitor {
    val isConnected: Flow<Boolean>
}

class AndroidNetworkMonitor(context: Context) : NetworkMonitor {
    private val connectivityManager = context.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager

    override val isConnected: Flow<Boolean> = callbackFlow {
        val callback = object : ConnectivityManager.NetworkCallback() {
            override fun onAvailable(network: Network) { trySend(true) }
            override fun onLost(network: Network) { trySend(false) }
        }
        
        val request = NetworkRequest.Builder()
            .addCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET)
            .build()
            
        connectivityManager.registerNetworkCallback(request, callback)
        
        // Trạng thái ban đầu
        val currentNetwork = connectivityManager.activeNetwork
        val caps = connectivityManager.getNetworkCapabilities(currentNetwork)
        trySend(caps?.hasCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET) == true)

        awaitClose { connectivityManager.unregisterNetworkCallback(callback) }
    }.distinctUntilChanged()
}
```
2.3. Presentation Module: 
Chuyển đổi trạng thái & Phản hồi UXTầng hiển thị chuyển đổi NetworkResult thành UiState để vẽ giao diện điều hướng, tuyệt đối không dùng từ khóa try-catch tại đây để bắt lỗi API nữa.Định nghĩa trạng thái giao diện UI State:
``` Kotlin
sealed interface UiState<out T> {
    object Loading : UiState<Nothing>
    data class Success<out T>(val data: T) : UiState<T>
    data class Error(val userMessage: String, val canRetry: Boolean) : UiState<Nothing>
}
```
Xử lý tại ViewModel (Kết hợp Reactive Retry):
``` Kotlin
class ProductViewModel(
    private val repository: ProductRepository,
    private val networkMonitor: NetworkMonitor
) : ViewModel() {

    private val _uiState = MutableStateFlow<UiState<List<Product>>>(UiState.Loading)
    val uiState = _uiState.asStateFlow()

    init {
        loadProducts()
        observeNetworkToRetry()
    }

    fun loadProducts() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            when (val result = repository.getProducts()) {
                is NetworkResult.Success -> _uiState.value = UiState.Success(result.data)
                is NetworkResult.Error -> _uiState.value = UiState.Error("Lỗi hệ thống (${result.code}). Vui lòng thử lại.", true)
                is NetworkResult.Exception -> _uiState.value = UiState.Error("Không có kết nối mạng. Đang chờ kết nối...", false)
            }
        }
    }

    // Tự động gọi lại API khi thiết bị có mạng trở lại (Auto-Resume)
    private fun observeNetworkToRetry() {
        viewModelScope.launch {
            networkMonitor.isConnected.collect { isConnected ->
                if (isConnected && _uiState.value is UiState.Error) {
                    loadProducts()
                }
            }
        }
    }
}
```
### 3. Hệ thống giám sát toàn cục dành cho Nhà phát triển (DX)
Để đảm bảo nhà phát triển có đầy đủ thông tin gỡ lỗi mà ứng dụng vẫn không bị crash ngoài ý muốn, ba lớp bảo vệ (Layers of Defense) sau cần được thiết lập:
- Lớp 1: Bắt lỗi bất đồng bộ bất ngờ tại ViewModelSử dụng CoroutineExceptionHandler làm lưới đỡ cuối cùng trong các tiến trình chạy ngầm của ViewModel phòng trường hợp logic tính toán hoặc parsing dữ liệu bị lỗi.

```Kotlin 
open class BaseViewModel : ViewModel() {
    protected val exceptionHandler = CoroutineExceptionHandler { _, throwable ->
        // Ghi nhận lỗi vào hệ thống log nội bộ hoặc Crashlytics
        Log.e("BaseViewModel", "Unhandled coroutine exception", throwable)
        FirebaseCrashlytics.getInstance().recordException(throwable)
    }
    
    // Sử dụng: viewModelScope.launch(exceptionHandler) { ... }
}
```
- Lớp 2: Lưới bảo hiểm toàn cục (Global Crash Catcher)Khởi tạo ngay khi ứng dụng vừa chạy (trong lớp Application) để bắt các Exception ở luồng chính (Main Thread) chưa được catch, đẩy thông tin lên Firebase và đóng app một cách êm đẹp thay vì hiển thị hộp thoại "App has stopped".Đoạn mã    
``` Kotlin 
override fun onCreate() {
    super.onCreate()
    
    Thread.setDefaultUncaughtExceptionHandler { thread, throwable ->
        // 1. Đẩy log lỗi nghiêm trọng lên Firebase Crashlytics ngay lập tức
        FirebaseCrashlytics.getInstance().recordException(throwable)
        
        // 2. Log logcat phục vụ debug nội bộ
        Log.e("CRITICAL_ERR", "Crash tại thread: ${thread.name}", throwable)
        
        // 3. Có thể điều hướng người dùng sang một Activity báo lỗi riêng biệt (ErrorActivity)
        // Hoặc để hệ thống tự kết thúc tiến trình an toàn
        android.os.Process.killProcess(android.os.Process.myPid())
        System.exit(1)
    }
}
```
4. Bảng tổng hợp đối chiếu chiến lược
Dưới đây là bảng tổng hợp tiêu chí xử lý ngoại lệ được chuẩn hóa lại dưới dạng Markdown hoàn chỉnh, loại bỏ các ký tự thừa và căn chỉnh trực quan để bạn dễ dàng lưu trữ hoặc đưa vào tài liệu dự án (Docs):

| Tiêu chí | Tầng Data | Tầng Platform | Tầng Presentation | Toàn cục (Application) |
| --- | --- | --- | --- | --- |
| **Nhiệm vụ chính** | Bọc lỗi hệ thống thành dữ liệu sạch (`NetworkResult`) | Giám sát trạng thái phần cứng (`Flow<Boolean>`) | Chuyển đổi dữ liệu thành trạng thái hiển thị (`UiState`) | Lưới bảo hiểm cuối cùng, ghi nhận Crash ẩn |
| **Hành vi phía UX** | Chạy ngầm, không ảnh hưởng trực tiếp đến UI | Không can thiệp trực tiếp | Hiển thị Banner, nút Thử lại, Tự động tải lại | Đóng ứng dụng êm đẹp hoặc chuyển sang màn hình bảo trì |
| **Hành vi phía DX** | Ghi log mã lỗi HTTP (4xx, 5xx) | Theo dõi sự thay đổi trạng thái IP/Mạng | Theo dõi vòng đời luồng (`viewModelScope`) | Đẩy dữ liệu dòng lỗi (Stacktrace) lên Firebase Crashlytics |
| **Kỹ thuật áp dụng** | `try-catch`, `Interceptors`, `Timeout` | `ConnectivityManager`, `callbackFlow` | `StateFlow`, `CoroutineExceptionHandler` | `Thread.setDefaultUncaughtExceptionHandler` |