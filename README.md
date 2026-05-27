# NT548 Lab 02 - CloudFormation

Repository này chứa mã nguồn cho bài lab 02 triển khai hạ tầng AWS bằng CloudFormation và kiểm thử template bằng `cfn-lint` + `taskcat`.

## 1. Nội dung bài lab

Template chính nằm tại:

```bash
templates/infrastructure.yaml
```

Template tạo các tài nguyên sau:

- VPC `10.0.0.0/16`
- Public subnet `10.0.1.0/24`
- Internet Gateway
- Public route table và default route `0.0.0.0/0`
- Security Group cho phép SSH port `22` và HTTP port `80`
- EC2 instance `t2.micro`
- Output `InstancePublicIP` để lấy public IP của EC2

Các file quan trọng:

```text
.
├── buildspec.yml
├── .taskcat.yml
├── templates/
│   └── infrastructure.yaml
└── README.md
```

## 2. Yêu cầu môi trường

Cần chuẩn bị:

- Tài khoản AWS có quyền tạo CloudFormation stack, VPC, subnet, route table, security group và EC2.
- AWS CLI v2.
- Python 3.9.
- `pip`.
- Git.
- EC2 Key Pair tên `nt548-key` trong region `ap-southeast-1`.

Region đang dùng trong bài:

```text
ap-southeast-1
```

Tên key pair đang khai báo trong `.taskcat.yml`:

```text
nt548-key
```

Nếu dùng key pair tên khác, cần sửa tham số `KeyName` trong `.taskcat.yml` hoặc truyền tham số khác khi triển khai CloudFormation.

## 3. Cài đặt môi trường

### 3.1. Cài AWS CLI

Kiểm tra AWS CLI:

```bash
aws --version
```

Nếu chưa có AWS CLI, cài AWS CLI v2 theo tài liệu chính thức của AWS cho hệ điều hành đang dùng.

Sau khi cài xong, cấu hình credentials:

```bash
aws configure
```

Nhập các giá trị:

```text
AWS Access Key ID
AWS Secret Access Key
Default region name: ap-southeast-1
Default output format: json
```

Kiểm tra tài khoản AWS hiện tại:

```bash
aws sts get-caller-identity
```

Nếu lệnh trả về `Account`, `UserId` và `Arn` là cấu hình AWS CLI đã hoạt động.

### 3.2. Tạo EC2 Key Pair

Template yêu cầu tham số `KeyName`. Với cấu hình hiện tại, key pair cần có tên `nt548-key`.

Kiểm tra key pair đã tồn tại chưa:

```bash
aws ec2 describe-key-pairs \
  --key-names nt548-key \
  --region ap-southeast-1
```

Nếu chưa có, tạo key pair:

```bash
aws ec2 create-key-pair \
  --key-name nt548-key \
  --region ap-southeast-1 \
  --query 'KeyMaterial' \
  --output text > nt548-key.pem
```

Phân quyền file key:

```bash
chmod 400 nt548-key.pem
```

Lưu ý: không commit file `.pem` lên Git.

### 3.3. Cài Python packages

Tạo virtual environment:

```bash
python3 -m venv .venv
```

Kích hoạt virtual environment trên Linux/macOS/WSL:

```bash
source .venv/bin/activate
```

Kích hoạt virtual environment trên Windows PowerShell:

```powershell
.venv\Scripts\Activate.ps1
```

Cài dependencies giống pipeline trong `buildspec.yml`:

```bash
python --version
pip install --upgrade "pip<25" "setuptools<81" wheel
pip install "PyYAML==5.4.1" --no-build-isolation
pip install "cfn-lint==0.87.10" "taskcat==0.9.58"
```

Kiểm tra công cụ:

```bash
cfn-lint --version
taskcat --version
```

## 4. Kiểm tra mã nguồn trước khi triển khai

Chạy `cfn-lint` để kiểm tra cú pháp và rule của CloudFormation template:

```bash
cfn-lint templates/infrastructure.yaml
```

Nếu không có output lỗi, template đã vượt qua bước lint.

## 5. Chạy kiểm thử triển khai bằng taskcat

`taskcat` sẽ đọc cấu hình trong `.taskcat.yml`, tạo CloudFormation stack thử nghiệm ở region `ap-southeast-1`, kiểm tra quá trình deploy, sau đó ghi kết quả vào thư mục output.

Chạy:

```bash
taskcat test run
```

Khi chạy thành công, kiểm tra các kết quả:

- Terminal hiển thị trạng thái test thành công.
- CloudFormation stack được tạo trong AWS Console.
- Thư mục output như `taskcat_outputs/` hoặc `.taskcat_outputs/` chứa log và file báo cáo.
- File báo cáo HTML có thể mở bằng trình duyệt để xem chi tiết.

Nếu taskcat báo lỗi `KeyName`, kiểm tra lại key pair `nt548-key` đã tồn tại trong region `ap-southeast-1`.

## 6. Triển khai thủ công bằng AWS CLI

Ngoài `taskcat`, có thể tự tạo CloudFormation stack bằng AWS CLI:

```bash
aws cloudformation create-stack \
  --stack-name nt548-lab02 \
  --template-body file://templates/infrastructure.yaml \
  --parameters ParameterKey=KeyName,ParameterValue=nt548-key \
  --region ap-southeast-1
```

Theo dõi trạng thái stack:

```bash
aws cloudformation describe-stacks \
  --stack-name nt548-lab02 \
  --region ap-southeast-1 \
  --query 'Stacks[0].StackStatus' \
  --output text
```

Khi trạng thái là `CREATE_COMPLETE`, lấy public IP của EC2:

```bash
aws cloudformation describe-stacks \
  --stack-name nt548-lab02 \
  --region ap-southeast-1 \
  --query 'Stacks[0].Outputs[?OutputKey==`InstancePublicIP`].OutputValue' \
  --output text
```

## 7. Kiểm tra kết quả triển khai

### 7.1. Kiểm tra trên AWS Console

Vào AWS Console và kiểm tra:

1. CloudFormation
   - Region: `ap-southeast-1`
   - Stack có trạng thái `CREATE_COMPLETE`
   - Tab `Resources` có các tài nguyên VPC, subnet, route table, security group và EC2
   - Tab `Outputs` có `InstancePublicIP`

2. EC2
   - Instance đang ở trạng thái `running`
   - Instance có public IPv4 address
   - Security Group cho phép inbound SSH port `22`

3. VPC
   - VPC có CIDR `10.0.0.0/16`
   - Public subnet có CIDR `10.0.1.0/24`
   - Route table có route `0.0.0.0/0` trỏ tới Internet Gateway

### 7.2. Kiểm tra bằng AWS CLI

Xem output của stack:

```bash
aws cloudformation describe-stacks \
  --stack-name nt548-lab02 \
  --region ap-southeast-1 \
  --query 'Stacks[0].Outputs'
```

Liệt kê tài nguyên trong stack:

```bash
aws cloudformation list-stack-resources \
  --stack-name nt548-lab02 \
  --region ap-southeast-1
```

Kiểm tra EC2 instance:

```bash
aws ec2 describe-instances \
  --filters "Name=tag:aws:cloudformation:stack-name,Values=nt548-lab02" \
  --region ap-southeast-1 \
  --query 'Reservations[].Instances[].{InstanceId:InstanceId,State:State.Name,PublicIp:PublicIpAddress,InstanceType:InstanceType}'
```

### 7.3. SSH vào EC2

Lấy public IP từ output `InstancePublicIP`, sau đó SSH:

```bash
ssh -i nt548-key.pem ec2-user@<InstancePublicIP>
```

Với AMI khác, username có thể là `ubuntu` thay vì `ec2-user`.

## 8. Chạy giống AWS CodeBuild

File `buildspec.yml` định nghĩa các bước CI/CD:

```text
install    -> cài Python packages
pre_build  -> chạy cfn-lint
build      -> chạy taskcat test run
artifacts  -> xuất template và .taskcat.yml
```

Có thể chạy lại các lệnh chính ở local:

```bash
pip install --upgrade "pip<25" "setuptools<81" wheel
pip install "PyYAML==5.4.1" --no-build-isolation
pip install "cfn-lint==0.87.10" "taskcat==0.9.58"
cfn-lint templates/infrastructure.yaml
taskcat test run
```

## 9. Xóa tài nguyên sau khi làm xong

Để tránh phát sinh chi phí, xóa stack sau khi hoàn thành bài lab.

Nếu triển khai bằng AWS CLI:

```bash
aws cloudformation delete-stack \
  --stack-name nt548-lab02 \
  --region ap-southeast-1
```

Theo dõi trạng thái xóa:

```bash
aws cloudformation describe-stacks \
  --stack-name nt548-lab02 \
  --region ap-southeast-1
```

Nếu dùng `taskcat`, kiểm tra CloudFormation Console và đảm bảo các stack test đã được xóa. Nếu còn stack test, xóa trực tiếp trong CloudFormation hoặc dùng AWS CLI với đúng tên stack.

## 10. Lỗi thường gặp

### Lỗi không tìm thấy key pair

Nguyên nhân: chưa tạo key pair `nt548-key` trong region `ap-southeast-1`.

Cách xử lý:

```bash
aws ec2 create-key-pair \
  --key-name nt548-key \
  --region ap-southeast-1 \
  --query 'KeyMaterial' \
  --output text > nt548-key.pem
chmod 400 nt548-key.pem
```

### Lỗi AWS credentials

Kiểm tra lại:

```bash
aws sts get-caller-identity
aws configure list
```

### Lỗi AMI không tồn tại

Template đang dùng AMI:

```text
ami-047126e50991d067b
```

AMI phải tồn tại trong region `ap-southeast-1`. Nếu AMI không còn hợp lệ, cần thay `ImageId` trong `templates/infrastructure.yaml` bằng AMI hợp lệ của region này.

### Lỗi giới hạn tài nguyên AWS

Nếu AWS báo vượt quota VPC hoặc EC2, kiểm tra và xóa các stack/tài nguyên cũ không dùng nữa.
