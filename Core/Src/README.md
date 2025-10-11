# Chuỗi hoạt động BLE cơ bản # 
Sơ đồ tổng quát:
```scss
[System Boot]
     ↓
MX_APPE_Init()
     ↓
APP_BLE_Init()
     ↓
BLE Stack init (M0+)
     ↓
Create GAP, GATT
     ↓
Start Advertising
     ↓
<Phone scans> → Connects
     ↓
Connection established
     ↓
Exchange data via Characteristics
     ↓
(Optionally disconnect / restart advertise)
```

## Tổng thế các lớp trong BLE stack của STM32WB55
Trên STM32WB55, firmware BLE được chia theo 4 tầng:
| Layer |Mục đích | Ví dụ hàm |
|-------|---------|-----------|
|Application Layer (CPU1) | Code của bạn (Advertise, GATT, Service, Xử lý event,...)|`APP_BLE_Init()` |
|Transport Layer (TL) CPU1 | Giao tiếp CPU1 <-> CPU2 qua IPCC | `appe_tl_init()`, `TL_Init()`|
|Wireless Stack (CPU2) | BLE stack chính thức của ST (GAP, GATT, L2CAP, HCI,..) | Chạy trong firmware `stm32wb5x_BLE_Stack_full_fw.bin`|
|RF Hardware Layer | Mạch RF vật lý, atten, clock,..| cấu hình bởi `MX_RF_Init()`| 

- Khi CPU1 (ứng dụng) bắt đầu chạy, nó phải thiết lập giao tiếp và khởi động CPU2 (BLE stack), sau đó mới được phép dùng
- Toàn bộ flow code trong Middleware STM32_WLAN:
```scss
main()
 └─ MX_APPE_Init()
     ├─ System_Init()
     ├─ SystemPower_Config()
     ├─ HW_TS_Init()
     └─ appe_TL_Init()
          ├─ TL_Init()
          ├─ TL_Enable()
          ├─ Gửi yêu cầu khởi động BLE CPU2
          └─ Khi CPU2 sẵn sàng → callback:
                APPE_SysUserEvtRx()
                    └─ APPE_SysEvtReadyProcessing()
                        ├─ APPD_EnableCPU2()
                        ├─ SHCI_C2_Config()
                        └─ APP_BLE_Init() (trong `file app_ble.h`)
                              ├─ Ble_Tl_Init()
                              ├─ SHCI_C2_BLE_Init()
                              ├─ Ble_Hci_Gap_Gatt_Init()
                              ├─ SVCCTL_Init()
                              ├─ DISAPP_Init(), HRSAPP_Init()
                              └─ Adv_Request(APP_BLE_FAST_ADV)
```

## Chi tiết từng bước
### Bước 1: Khởi động hệ thống (System Boot) 
- APPE (**"Application Entry"**) - là lớp trung gian giữa BLE stack và ứng dụng của bạn. Cụ thể hơn, nó là điểm khởi đầu của ứng dụng người dùng (User Application Entry Point) - nơi:
	- BLE stack được khởi tạo
	- Các dịch vụ BLE(GATT, GAP,..) được đăng ký
	- Các task, callback và event được kết nối
	- Và sau đó nhảy đến code của bạn (ví dụ `App_BLE_Init()` hoặc `App_UserContextInit()`)

- Trong `main.c` thường thấy:
```c
int main(void){
	HAL_Init();
	MX_APPE_Config();
	SystemClock_Config();
	MX_GPIO_Init();
	MX_RF_Init(); //Hoặc MX_IPCC_Init() tùy phiên bản
	MX_APPE_Init(); //Gọi Application Entry
}
```

- Hàm gọi sớm nhất sau `HAL_Init()` là `MX_APPE_Config()`
	- Mục đích: Cấu hình low-level HSE tuning (tần số RF chính xác cho BLE)
	- Đặt chế độ hoạt động của CPU2 (M0+)
	- Khởi tạo một số vùng bộ nhớ chia sẻ giữa CPU1 và CPU2
	
- Tiếp theo là hàm `MX_RF_Init()`. Mục đích:
	- Khởi tạo phần radio front-end (RF) để CPU2 có thể điều khiển module RF BLE
	- Dùng để cấu hình pin RF, đường bias, và đảm bảo clock RF đã bật
	
- Hàm `MX_IPPC_Init()` 
	- Đây là cơ chế giúp CPU1 và CPU2 trao đổi dữ liệu (BLE connected, event,..)
	
- Hàm `MX_RTC_Init()` được dùng cho: 
	- BLE sleep timing
	- Advertising interval/connection interval management 
	- Timestamp log (nếu cần) 
	
- Hàm `main()` gọi `MX_APPE_Init()` (file `app_entry.c`). Trong hàm này chứa các hàm con để khởi tạo hệ thống cho BLE:
	- `System_Init()`: Bật clock cho CPU2, IPCC, shared RAM
	- `SystemPower_Config()`: Cho phép CPU2 đi vào sleep khi idle để tiết kiệm điện 
	- `HW_TS_Init()`: Cung cấp Timer tick cho BLE event timing
	- `appe_TL_Init()`: Khởi tạo Transport Layer, cụ thể:
		- Cấu hình IPCC channel cho BLE 
		- Gọi `TL_Init()` để gắn callback nhận event từ CPU2
		- Gửi lệnh khởi động BLE firmware sang CPU2 
		- Đợi sự kiện SYSTEM_READY từ CPU2 


### Bước 2: Khởi tạo BLE stack 
- Khi CPU2 sẵn sàng bằng cách phản hồi lại SYSTEM_READY, bắt đầu gọi đến callback đến hàm `void APEE_SysUserEvtRx(void *pPayload)`
- Hàm `APPE_SysUserEvtRx()` - callback khi CPU2 phản hồi
	- CPU2 gửi lại các sự kiện hệ thống qua IPCC, được đóng gói trong `p_sys_event->subevtcode` của struct `TL_AsynEvt_t`
	- Nếu event là `SHCI_SUB_EVT_CODE_READY` -> Tức CPU2 đã sẵn sàng để dùng BLE
	- Hàm sẽ gọi đến hàm `APPE_SysEvtReadyProcessing()` để xử lý tiếp
	
- Hàm `APPE_SysEvtReadyProcessing()` có nhiệm vụ xác nhận rằng BLE stack đã sẵn sàng hay chưa:
	- Kiểm tra `sysevt_ready_rsp`:
		- Nếu `WIRELESS_FW_RUNNING` -> BLE stack trên CPU2 đã khởi động
		- Nếu là `FUS_FW_RUNNING` -> CPU2 đang ở chế độ Bootloader (Firmware Upgrade Service)
	- Khi BLE FW đang chạy: 
		- `APPD_EnableCPU2()` -> bật trace channel để debug log từ CPU2
		- `SHCI_C2_Config()` -> Gửi config về DeviceID, Revision, Mask event cần enable 
		- Cuối cùng gọi `APP_BLE_Init()` -> Bắt đầu khởi tạo BLE thực tế trên CPU2

- Chính tại đây, BLE stack mới chính thực được khởi tạo. Hàm `APP_BLE_Init()` làm 5 việc lớn:
	1. Gọi `Ble_Tl_Init()`:
		- Gắn các callback để CPU1 có thể nhận asynchronous BLE events (Connect, disconnect,..)
		- Trong hàm này sẽ thiết lập HCI (Host Controller Interface) trước khi có thể nhận data từ BLE. Đây là giao diện chuẩn - lớp trung gian giúp kết nối CPU1 (application) - Host và CPU2 (BLE stack) - Controller
		
	2. Gọi `SHCI_C2_BLE_Init()`:
		- Gửi gói `SHCI_C2_Ble_Init_Cmd_Packet_t` qua IPCC đến CPU2
		- Cấu hình số lượng GAT attribute, MTU, connection, trasmit power,...
		- Khi thành công -> BLE stack trên CPU2 khởi động hoàn toàn
		
	3. Gọi `Ble_Hci_Gap_Gatt_Init()`:
		- Khởi tạo các tầng BLE logic trên CPU1
			- HCI (Host Controller Interface)
			- GAP (Generic Access Profile): Hồ sơ xác định các thiết bị BLE khám phá (máy nào là máy chủ (Host), máy nào máy khách) và cách thức tương tác với nhau ở mức độ thấp. Chức năng chính:
				- Advertising (Quảng bá): Cho phép thiết bị phát tín hiệu đến các thiết bị khác có thể tìm thấy
				- Discovery (Khám phá): Giúp các thiết bị tìm kiếm và quét các BLE khác
				- Thiết lập kết nối giữa các thiết bị	 
			- GATT (Generic Attribute Profile): Hồ sơ quy định cấu trúc dữ liệu phân cấp (dịch vụ, đặc điểm) và cách trao đổi dữ liệu cụ thể sau khi kết nối đã được thiếp lập. 
			
	4. Khởi tạo service và ứng dụng BLE 
		- `SVCCTL_Init()` - Đăng ký GATT service handler
		- `DISAPP_Init()` - Device Information Service
		- `HSAPP_Init()` - Heart Rate Service (mẫu của ST)
	
	5. Quảng bá (Advertising)
		- `Adv_Request(APP_BLE_FAST_ADV)` - bắt đầu phát BLE beacon để cho phép kết nối
