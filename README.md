
# NGƯỜI VIỆT BAY KHÔNG CẦN CÁNH

## Giới thiệu

Hướng dẫn này mô tả cách tự động phát hiện các thiết bị trong cùng một mạng, sau đó:
- Thiết lập SSH không mật khẩu cho các thiết bị Linux.
- Push code từ máy chủ (server) lên các thiết bị trong mạng.
- Chạy script Python trên các thiết bị.
- Vô hiệu hóa firewall trên các thiết bị (hỗ trợ cả Linux và Windows).

Quá trình này giúp bạn tiết kiệm thời gian và công sức khi cần cập nhật, triển khai code, hoặc thực hiện các thao tác cấu hình trên nhiều thiết bị cùng lúc.

## Yêu cầu

1. **Python 3.x**: Đảm bảo máy chủ có Python 3.x được cài đặt.
2. **SSH không mật khẩu**: Máy chủ phải có khả năng kết nối đến các thiết bị Linux thông qua SSH không cần mật khẩu.
3. **WinRM (Windows Remote Management)**: Được bật trên các thiết bị Windows.
4. **Công cụ `nmap`**: Dùng để quét các thiết bị trong mạng. Cài đặt `nmap` bằng lệnh:
   ```bash
   sudo apt-get install nmap
   ```
5. **Bật PowerShell Remoting trên Windows**:
   Trên các thiết bị Windows, chạy lệnh sau để bật PowerShell Remoting:
   ```powershell
   Enable-PSRemoting -Force
   ```

## Các bước thực hiện

### Bước 1: Thiết lập SSH không mật khẩu cho Linux

Để đảm bảo quá trình tự động hóa diễn ra suôn sẻ, bạn cần thiết lập SSH không mật khẩu từ máy chủ đến tất cả các thiết bị Linux trong mạng.

1. **Tạo SSH key trên máy chủ**:
    - Chạy lệnh sau để tạo SSH key mới:
    ```bash
    ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
    ```
    - Khi được hỏi về vị trí lưu trữ key, bạn có thể nhấn Enter để chấp nhận vị trí mặc định.
    - Khi được hỏi về passphrase, bạn có thể để trống để SSH không yêu cầu mật khẩu khi kết nối.

2. **Sao chép SSH key đến các thiết bị client sau khi phát hiện**:
    - Quá trình này sẽ được tự động hóa trong script Python ở bước sau.

### Bước 2: Viết script Python

Dưới đây là script Python để phát hiện các thiết bị trong cùng một mạng, sau đó tự động push code, chạy code trên các thiết bị đó, và vô hiệu hóa firewall.

```python
import os
import subprocess
import re
import platform

# Đường dẫn tới thư mục chứa mã nguồn trên máy chủ
source_dir = "/home/godhunter7/Documents/project-tan-cac-chi-khoi-tren/index.html"

# Tên script cần chạy trên client
script_name = "your_script.py"

# Quét mạng để tìm tất cả các thiết bị
import os
import subprocess
import re
import socket

# Lấy IP của máy chủ đang chạy chương trình
def get_host_ip():
    s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    try:
        # Kết nối đến một địa chỉ bất kỳ (ở đây là Google DNS) để lấy IP nội bộ
        s.connect(("8.8.8.8", 80))
        host_ip = s.getsockname()[0]
    except Exception:
        host_ip = "127.0.0.1"
    finally:
        s.close()
    return host_ip

# Quét mạng để tìm tất cả các thiết bị và bỏ qua các IP không mong muốn
def scan_network():
    print("Đang quét mạng để tìm các thiết bị...")
    result = subprocess.run(["nmap", "-sn", "192.168.1.1/24"], capture_output=True, text=True)
    ips = re.findall(r'(\d+\.\d+\.\d+\.\d+)', result.stdout)

    # Lấy IP của máy chủ
    host_ip = get_host_ip()
    
    # Bỏ qua IP 192.168.1.1 và IP của máy chủ đang chạy chương trình
    ips = [ip for ip in ips if ip not in ["192.168.1.1", host_ip]]

    print(f"Đã tìm thấy {len(ips)} thiết bị trong mạng (bỏ qua 192.168.1.1 và {host_ip}).")
    return ips

if __name__ == "__main__":
    ips = scan_network()
    print(f"Các IP được quét: {ips}")


# Kiểm tra kết nối SSH tới client Linux
def check_ssh_connection(client_ip, username="your_default_user"):
    try:
        subprocess.run(["ssh", f"{username}@{client_ip}", "exit"], check=True)
        return True
    except subprocess.CalledProcessError:
        return False

# Kiểm tra kết nối WinRM tới client Windows
def check_winrm_connection(client_ip, username="Administrator"):
    try:
        winrm_command = f"winrs -r:{client_ip} -u:{username} -p:your_password 'exit'"
        subprocess.run(winrm_command, shell=True, check=True)
        return True
    except subprocess.CalledProcessError:
        return False

# Kiểm tra kết nối tới thiết bị Android qua adb
def check_android_connection(client_ip):
    try:
        adb_command = f"adb connect {client_ip}:38517"
        subprocess.run(adb_command, shell=True, check=True)
        return True
    except subprocess.CalledProcessError:
        return False

# Copy SSH key sang client Linux
def copy_ssh_key(client_ip, username="your_default_user"):
    print(f"Đang sao chép SSH key tới {client_ip}...")
    subprocess.run(["ssh-copy-id", f"{username}@{client_ip}"], check=True)

# Đồng bộ mã nguồn với client Linux
def sync_code(client_ip, username="your_default_user"):
    print(f"Đang đồng bộ mã nguồn với {client_ip}...")
    rsync_command = [
        "rsync", "-avz", "--exclude='.git'", source_dir,
        f"{username}@{client_ip}:/home/{username}/project/"
    ]
    subprocess.run(rsync_command, check=True)

# Chạy script trên client Linux
def run_script_linux(client_ip, username="your_default_user"):
    print(f"Đang chạy script trên {client_ip} (Linux)...")
    ssh_command = [
        "ssh", f"{username}@{client_ip}",
        f"cd /home/{username}/project/ && python3 {script_name}"
    ]
    subprocess.run(ssh_command, check=True)

# Vô hiệu hóa firewall trên client Linux
def disable_firewall_linux(client_ip, username="your_default_user"):
    print(f"Đang vô hiệu hóa firewall trên {client_ip} (Linux)...")
    ssh_command = [
        "ssh", f"{username}@{client_ip}",
        "sudo ufw disable || sudo iptables -F"
    ]
    subprocess.run(ssh_command, check=True)

# Vô hiệu hóa firewall trên client Windows
def disable_firewall_windows(client_ip, username="Administrator"):
    print(f"Đang vô hiệu hóa firewall trên {client_ip} (Windows)...")
    winrm_command = f"winrs -r:{client_ip} -u:{username} -p:your_password 'netsh advfirewall set allprofiles state off'"
    subprocess.run(winrm_command, shell=True, check=True)

# Thực hiện push code và chạy script trên Android
def deploy_to_android(client_ip):
    try:
        print(f"Đang triển khai tới thiết bị Android tại {client_ip}...")
        adb_push_command = f"adb -s {client_ip}:5555 push /path/to/your/app /sdcard/"
        subprocess.run(adb_push_command, shell=True, check=True)
        adb_run_command = f"adb -s {client_ip}:5555 shell 'sh /sdcard/your_script.sh'"
        subprocess.run(adb_run_command, shell=True, check=True)
    except subprocess.CalledProcessError as e:
        print(f"Lỗi khi triển khai tới thiết bị Android: {e}")

# Xác định hệ điều hành của client
def get_os_type(client_ip):
    print(f"Đang xác định hệ điều hành của {client_ip}...")
    try:
        # Kiểm tra kết nối SSH (Linux)
        ssh_command = f"ssh {client_ip} 'uname -s'"
        result = subprocess.run(ssh_command, shell=True, capture_output=True, text=True)
        if "Linux" in result.stdout:
            return "Linux"
    except subprocess.CalledProcessError:
        pass

    try:
        # Kiểm tra kết nối WinRM (Windows)
        winrm_command = f"winrs -r:{client_ip} -u:Administrator -p:your_password 'ver'"
        result = subprocess.run(winrm_command, shell=True, capture_output=True, text=True)
        if "Microsoft" in result.stdout or "Windows" in result.stdout:
            return "Windows"
    except subprocess.CalledProcessError:
        pass

    # Kiểm tra kết nối Android
    if check_android_connection(client_ip):
        return "Android"

    return "Unknown"

# Thực hiện push code và vô hiệu hóa firewall trên tất cả các thiết bị
def deploy_to_network():
    ips = scan_network()
    errors = []
    
    for ip in ips:
        os_type = get_os_type(ip)
        if os_type == "Linux":
            if check_ssh_connection(ip):
                try:
                    copy_ssh_key(ip)
                    sync_code(ip)
                    run_script_linux(ip)
                    disable_firewall_linux(ip)
                except subprocess.CalledProcessError as e:
                    error_message = f"Lỗi xảy ra khi xử lý {ip} (Linux): {e}"
                    print(error_message)
                    errors.append(error_message)
            else:
                error_message = f"Không thể kết nối SSH tới {ip} (Linux)."
                print(error_message)
                errors.append(error_message)
        elif os_type == "Windows":
            if check_winrm_connection(ip):
                try:
                    disable_firewall_windows(ip)
                except subprocess.CalledProcessError as e:
                    error_message = f"Lỗi xảy ra khi xử lý {ip} (Windows): {e}"
                    print(error_message)
                    errors.append(error_message)
            else:
                error_message = f"Không thể kết nối WinRM tới {ip} (Windows)."
                print(error_message)
                errors.append(error_message)
        elif os_type == "Android":
            deploy_to_android(ip)
        else:
            error_message = f"Không thể xác định hệ điều hành của {ip}, bỏ qua..."
            print(error_message)
            errors.append(error_message)

    print("Hoàn thành quá trình cho tất cả các thiết bị.")
    
    if errors:
        print("\nCác lỗi đã xảy ra:")
        for error in errors:
            print(error)

if __name__ == "__main__":
    deploy_to_network()

```

### Bước 3: Chạy script

1. **Lưu script** với tên, ví dụ `deploy_network.py`.
2. **Chạy script** bằng lệnh sau:
   ```bash
   python3 deploy_network.py
   ```

### Bước 4: Tùy chỉnh và bảo trì

- **Tùy chỉnh dải địa chỉ IP**: Trong phần quét mạng (`scan_network`), địa chỉ IP `192.168.1.0/24` có thể thay đổi tùy theo dải IP của mạng bạn sử dụng.
- **Đặt username mặc định**: Script mặc định sử dụng `your_default_user` cho Linux và `Administrator` cho Windows. Bạn có thể thay đổi giá trị này hoặc cập nhật khi cần.
- **Giám sát lỗi**: Script có cơ chế xử lý lỗi cơ bản, in ra lỗi nếu có. Bạn có thể mở rộng để ghi log hoặc thực hiện các biện pháp khắc phục lỗi.

### Chú ý

- **Bảo mật**: Đảm bảo rằng quá trình copy SSH key và vô hiệu hóa firewall được bảo mật và chỉ áp dụng cho các thiết bị mà bạn có quyền kiểm soát.
- **Firewall và quyền truy cập**: Kiểm tra rằng các thiết bị trong mạng không bị chặn truy cập SSH/WinRM và bạn có quyền truy cập cần thiết.
- **An toàn bảo mật**: Vô hiệu hóa firewall có thể khiến các thiết bị dễ bị tấn công. Hãy chắc chắn rằng bạn chỉ thực hiện điều này trong môi trường an toàn.

---

Lưu ý: Script này chỉ là một giải pháp mẫu và có thể cần được điều chỉnh theo yêu cầu cụ thể của hệ thống và môi trường mạng của bạn.



