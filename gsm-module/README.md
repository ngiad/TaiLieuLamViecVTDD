# Hướng Dẫn Sử Dụng GSM Module với Node.js

## Mục Lục
- [Phần 1: Hướng Dẫn Lệnh Trên GSM Module](#phần-1-hướng-dẫn-lệnh-trên-gsm-module)
- [Phần 2: Thao Tác Code Với Node.js](#phần-2-thao-tác-code-với-nodejs)

## Phần 1: Hướng Dẫn Lệnh Trên GSM Module

### 1. Lệnh cơ bản để kiểm tra và quản lý IMEI

#### Đọc IMEI
```
AT+CGSN
```
Trả về số IMEI của module.

#### Ghi IMEI (nếu được hỗ trợ)
```
AT+EGMR=1,7,"NEW_IMEI"
```
Thay thế NEW_IMEI bằng số IMEI mới (15 chữ số).

#### Kiểm tra trạng thái
Lệnh AT để kiểm tra module hoạt động:
```
AT
```
Trả về OK nếu module hoạt động.

#### Khởi động lại module
```
AT+CFUN=1,1
```
Khởi động lại module GSM để áp dụng thay đổi.

### 2. Lệnh quản lý SMS

#### Kiểm tra danh sách tin nhắn
```
AT+CMGL="ALL"
```
Hiển thị danh sách tất cả tin nhắn.

#### Đọc tin nhắn cụ thể
```
AT+CMGR=INDEX
```
Thay thế INDEX bằng chỉ số tin nhắn cần đọc.

#### Xóa tin nhắn
```
AT+CMGD=INDEX
```
Xóa tin nhắn tại chỉ số INDEX.

#### Gửi SMS
```
AT+CMGS="PHONE_NUMBER"
```
Sau đó, nhập nội dung tin nhắn, kết thúc bằng ký tự Ctrl+Z hoặc 0x1A.

### 3. Lệnh quản lý cuộc gọi

#### Thực hiện cuộc gọi
```
ATDPHONE_NUMBER;
```

#### Ngắt cuộc gọi
```
ATH
```

#### Trả lời cuộc gọi
```
ATA
```

### 4. Lệnh giao tiếp dữ liệu

#### Kiểm tra mạng
```
AT+COPS?
```
Kiểm tra nhà mạng hiện tại.

#### Kiểm tra tín hiệu
```
AT+CSQ
```
Trả về cường độ tín hiệu.

#### Kiểm tra trạng thái kết nối dữ liệu
```
AT+CGATT?
```
1 nghĩa là kết nối, 0 nghĩa là chưa kết nối.

## Phần 2: Thao Tác Code Với Node.js

### 1. Cài đặt môi trường

#### Cài đặt thư viện SerialPort
```bash
npm install serialport
```

#### Cài đặt MongoDB (nếu cần)
```bash
npm install mongoose
```

### 2. Khởi tạo kết nối với GSM module

```javascript
const SerialPort = require('serialport');
const Readline = require('@serialport/parser-readline');

// Kết nối cổng COM
const port = new SerialPort('COM3', { baudRate: 9600 });
const parser = port.pipe(new Readline({ delimiter: '\r\n' }));

// Khi mở cổng thành công
port.on('open', () => {
  console.log('Serial Port Opened');
});

// Nhận dữ liệu từ module
parser.on('data', (data) => {
  console.log('Received:', data.trim());
});

// Gửi lệnh tới module
function sendCommand(command) {
  port.write(`${command}\r`, (err) => {
    if (err) {
      console.error('Error:', err.message);
    } else {
      console.log('Command sent:', command);
    }
  });
}
```

### 3. Quản lý IMEI

#### Đọc IMEI
```javascript
function readIMEI() {
  sendCommand('AT+CGSN');
}
```

#### Ghi IMEI
```javascript
function writeIMEI(newIMEI) {
  sendCommand(`AT+EGMR=1,7,"${newIMEI}"`);
}
```

#### Kiểm tra IMEI trong MongoDB
```javascript
const mongoose = require('mongoose');

// Kết nối MongoDB
mongoose.connect('mongodb://localhost:27017/gsmDB', { 
  useNewUrlParser: true, 
  useUnifiedTopology: true 
});

// Schema lưu IMEI
const IMEISchema = new mongoose.Schema({
  imei: { type: String, unique: true },
  createdAt: { type: Date, default: Date.now },
});

const IMEIModel = mongoose.model('IMEI', IMEISchema);

// Lưu IMEI vào database
function saveIMEI(imei) {
  const newIMEI = new IMEIModel({ imei });
  newIMEI.save()
    .then(() => console.log('IMEI saved:', imei))
    .catch((err) => console.error('Error saving IMEI:', err.message));
}

// Kiểm tra IMEI đã tồn tại hay chưa
function checkIMEI(imei) {
  IMEIModel.findOne({ imei })
    .then((result) => {
      if (result) console.log('IMEI exists:', imei);
      else console.log('IMEI not found:', imei);
    })
    .catch((err) => console.error('Error checking IMEI:', err.message));
}
```

### 4. Quản lý SMS

#### Gửi tin nhắn
```javascript
function sendSMS(phoneNumber, message) {
  sendCommand(`AT+CMGS="${phoneNumber}"`);
  setTimeout(() => {
    port.write(`${message}\x1A`, (err) => {
      if (err) console.error('Error sending SMS:', err.message);
      else console.log('SMS sent to:', phoneNumber);
    });
  }, 1000);
}
```

#### Đọc tin nhắn
```javascript
function readSMS(index) {
  sendCommand(`AT+CMGR=${index}`);
}
```

#### Xóa tin nhắn
```javascript
function deleteSMS(index) {
  sendCommand(`AT+CMGD=${index}`);
}
```

### 5. Quản lý nhiều GSM module

```javascript
const ports = ['COM3', 'COM4', 'COM5']; // Danh sách cổng COM

ports.forEach((portName) => {
  const port = new SerialPort(portName, { baudRate: 9600 });
  const parser = port.pipe(new Readline({ delimiter: '\r\n' }));

  port.on('open', () => console.log(`Port ${portName} Opened`));

  parser.on('data', (data) => {
    console.log(`Data from ${portName}:`, data.trim());
  });

  port.write('AT\r', (err) => {
    if (err) console.error(`Error on ${portName}:`, err.message);
    else console.log(`Command sent to ${portName}`);
  });
});
```

### 6. Lưu ý khi thao tác

- Xử lý lỗi: Kiểm tra và xử lý các trường hợp module không phản hồi.
- Khởi động lại module: Gửi lệnh AT+CFUN=1,1 sau khi ghi IMEI.
- Bảo mật dữ liệu: Đảm bảo dữ liệu IMEI và tin nhắn không bị rò rỉ.
