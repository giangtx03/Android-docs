Để phân biệt rõ ràng 3 layer này, bạn hãy tưởng tượng ứng dụng của bạn như một **nhà hàng**:

* **Presentation (Giao diện):** Là các bạn phục vụ bàn và không gian nhà hàng. Nhiệm vụ là tiếp đón khách, ghi nhận yêu cầu (tương tác UI) và bưng món ăn ra (hiển thị dữ liệu).
* **Domain (Nghiệp vụ cốt lõi):** Là Bếp trưởng và công thức nấu ăn. Nhận yêu cầu, tính toán, quyết định cách chế biến (Business Logic). Bếp trưởng không cần biết khách ngồi bàn nào hay dùng đĩa sứ hay đĩa nhựa.
* **Data (Dữ liệu):** Là kho nguyên liệu và người đi chợ. Nhiệm vụ là đi lấy thịt, rau (từ Server/Database) mang về cho Bếp trưởng.

Dưới đây là sự phân định rạch ròi chức năng, nhiệm vụ của từng lớp và cách mở rộng cho các Android Components khác.

---

### 1. Phân biệt rõ 3 Layer cơ bản

#### A. DOMAIN LAYER (Tầng Cốt lõi)

* **Chức năng nhiệm vụ:** Chứa các quy tắc nghiệp vụ (Business Rules). Nó quyết định ứng dụng *làm gì*.
* **Đặc điểm nhận diện:**
* Thuần Kotlin/Java 100%. Không có bất kỳ dependency nào từ Android SDK (Không `Context`, không `LiveData`, không `StateFlow`, không thư viện Android).
* Là lớp độc lập nhất. Mọi layer khác đều phải phụ thuộc vào nó (Dependency Rule).


* **Thành phần chính:**
* **Entities (Models):** Các đối tượng dữ liệu lõi (ví dụ: `User`, `Order`).
* **Use Cases (Interactors):** Mỗi class đại diện cho một hành động duy nhất của hệ thống (ví dụ: `ValidateLoginUseCase`, `CalculateTaxUseCase`).
* **Repository Interfaces:** Giao ước quy định cách lấy dữ liệu (Ví dụ: `interface UserRepository { fun getUser(): User }`).



#### B. DATA LAYER (Tầng Dữ liệu)

* **Chức năng nhiệm vụ:** Tìm kiếm, lưu trữ và cung cấp dữ liệu. Lớp này thực thi (implement) các giao ước mà lớp Domain đưa ra. Nó quyết định dữ liệu được lấy từ *đâu* và *như thế nào*.
* **Đặc điểm nhận diện:** Chứa các thư viện và SDK của bên thứ 3 (Retrofit, Firebase, Room Database, SharedPreferences). Nó biết về hệ điều hành Android (chứa `Context` để truy cập DB cục bộ).
* **Thành phần chính:**
* **Repository Implementations:** Class thực thi Interface của Domain (Ví dụ: `UserRepositoryImpl`).
* **Data Sources:** Các nguồn dữ liệu cụ thể (RemoteDataSource dùng Retrofit, LocalDataSource dùng Room).
* **DTOs / Mappers:** Các class hứng dữ liệu dạng JSON/SQL và hàm chuyển đổi (map) chúng thành Domain Entities.



#### C. PRESENTATION LAYER (Tầng Giao diện)

* **Chức năng nhiệm vụ:** Quản lý giao diện, hiển thị dữ liệu cho người dùng và ghi nhận các sự kiện tương tác (click, vuốt).
* **Đặc điểm nhận diện:** Phụ thuộc hoàn toàn vào Android SDK và lifecycle (vòng đời).
* **Thành phần chính:**
* **UI Components:** `Activity`, `Fragment`, Jetpack Compose.
* **State Management:** `ViewModel`, `StateFlow`, `LiveData`. Nó gọi Use Cases từ Domain để lấy kết quả và đóng gói thành UI State.



---

### 2. Áp dụng mở rộng cho các Android Components khác

Hệ điều hành Android có 4 components chính (Activity, Service, BroadcastReceiver, ContentProvider) và nhiều công cụ hỗ trợ (WorkManager, AlarmManager, Location...).

**Quy tắc chung để xếp loại:**

* Nếu component đó đóng vai trò là **"Điểm vào" (Entry Point)** kích hoạt một luồng nghiệp vụ giống như UI -> Nó hoạt động như **Presentation**. Nó sẽ gọi `Use Cases`.
* Nếu component đó đóng vai trò là **"Nguồn cung cấp dữ liệu hoặc công cụ hệ thống"** -> Nó thuộc về **Data** (hoặc Infrastructure). Nó cung cấp dữ liệu cho `Repository`.

Dưới đây là cách xác định và mở rộng cụ thể:

#### 1. WorkManager / AlarmManager (Xử lý ngầm, lập lịch)

* **Bản chất:** Là các điểm vào (Entry point) được kích hoạt bởi hệ thống theo thời gian thay vì bởi người dùng.
* **Xếp lớp:** Nằm ở vành đai ngoài cùng (Framework). Thường được gom vào mô-đun `:app` hoặc tạo một package/module riêng biệt tên là `:background` hoặc `:worker`.
* **Cách kết nối kiến trúc:** Worker/Alarm sẽ được inject các **Use Cases** (Domain Layer) để thực thi logic.
* *Ví dụ:* `SyncDataWorker` chạy mỗi 12 giờ. Worker này sẽ gọi `SyncDataUseCase`. Tuyệt đối không để Worker gọi trực tiếp Retrofit hay Room DB.



#### 2. Broadcast Receiver (Lắng nghe sự kiện hệ thống)

* **Bản chất:** Nhận các sự kiện như SMS đến, mất mạng, pin yếu. Đây cũng là một "Điểm vào".
* **Xếp lớp:** Nằm cùng lớp với Presentation hoặc module `:app`.
* **Cách kết nối kiến trúc:** Tương tự Worker, BroadcastReceiver sẽ nhận/inject các **Use Cases**.
* *Ví dụ:* `SmsReceiver` lắng nghe tin nhắn OTP. Khi nhận được, nó gọi `VerifyOtpUseCase` để xử lý logic, thay vì viết logic verify ngay trong Receiver.



#### 3. Services (Foreground / Background)

* **Xếp lớp phụ thuộc vào mục đích sử dụng:**
* **Nhóm Presentation:** Nếu Service dùng để phát nhạc (MediaSessionService) và hiển thị Notification có các nút Play/Pause để người dùng tương tác -> Nó hoạt động như UI (Presentation). Nó sẽ lắng nghe State và gọi `Use Cases`.
* **Nhóm Data/Infrastructure:** Nếu Service (hoặc API hệ thống) dùng để liên tục theo dõi tọa độ GPS hoặc đếm số bước chân -> Nó là một **Data Source**.


* **Cách kết nối (nhóm Data):** Lớp Domain định nghĩa `interface LocationRepository`. Lớp Data tạo ra `LocationTrackerImpl` (sử dụng Location Service/FusedLocationProvider) để map tọa độ GPS thành `LocationEntity` và trả về cho Domain.

#### 4. Content Provider

* **Bản chất:** Là cầu nối để cung cấp dữ liệu của app bạn cho app khác, hoặc lấy dữ liệu từ app khác (ví dụ lấy Danh bạ điện thoại).
* **Xếp lớp:** Thuộc về **Data Layer** (Đóng vai trò là Local/System Data Source).
* **Cách kết nối kiến trúc:** Bạn viết một `ContactsDataSource` trong Data Layer, bên trong đó sử dụng `ContentResolver` để truy vấn danh bạ. Sau đó đẩy lên `ContactsRepositoryImpl`. Domain không hề biết bạn dùng ContentProvider để lấy danh bạ.

### Tóm tắt công thức mở rộng

Khi có một công nghệ mới (Ví dụ Bluetooth, NFC, Camera, Firebase Cloud Messaging), bạn chỉ cần đặt 2 câu hỏi:

1. **Nó kích hoạt hành động hay nó cung cấp dữ liệu?**
* Kích hoạt (FCM Push Notification) -> Đứng ngoài cùng, gọi **Use Case**.
* Cung cấp (Bluetooth, Camera, Sensor) -> Thuộc Data layer, bị bọc bởi **Repository Interface**.


2. **Nó có chứa Android SDK không?**
* Có -> Tuyệt đối không được nhét vào thư mục `:domain`.