# Packer (HashiCorp) - Полная шпаргалка

## 📋 Содержание
1. [Введение](#введение)
2. [Базовый синтаксис HCL2](#базовый-синтаксис-hcl2)
3. [Структура Template](#структура-template)
4. [Переменные](#переменные)
5. [Локальные переменные](#локальные-переменные)
6. [Data Sources](#data-sources)
7. [Source блоки](#source-блоки)
8. [Build блоки](#build-блоки)
9. [Provisioners](#provisioners)
10. [Post-Processors](#post-processors)
11. [Функции](#функции)
12. [Best Practices](#best-practices)
13. [Примеры](#примеры)
14. [Команды CLI](#команды-cli)

---

## Введение

**Packer** - инструмент для автоматизации создания идентичных образов машин для различных платформ.

### Основные концепции

```
Template        - Конфигурационный файл (.pkr.hcl)
Builder         - Компонент создания образа (amazon-ebs, docker, etc.)
Provisioner     - Настройка образа (shell, ansible, etc.)
Post-Processor  - Обработка после создания
Artifact        - Результат работы (AMI, Docker image, etc.)
```

---

## Базовый синтаксис HCL2

### Структура файла

```hcl
# Комментарий
// Также комментарий
/* Многострочный
   комментарий */

packer {
  required_version = ">= 1.9.0"
  required_plugins {
    amazon = {
      version = ">= 1.2.0"
      source  = "github.com/hashicorp/amazon"
    }
  }
}

variable "region" {
  type    = string
  default = "us-east-1"
}

locals {
  timestamp = formatdate("YYYY-MM-DD-hhmm", timestamp())
}

source "amazon-ebs" "ubuntu" {
  ami_name      = "my-ami-${local.timestamp}"
  instance_type = "t2.micro"
  region        = var.region
  source_ami    = "ami-12345"
  ssh_username  = "ubuntu"
}

build {
  sources = ["source.amazon-ebs.ubuntu"]
  
  provisioner "shell" {
    inline = ["echo 'Hello Packer'"]
  }
}
```

### Правила именования

```hcl
# ✅ Правильно
variable "instance_type" {}
source "amazon-ebs" "web_server" {}
locals {
  app_name = "myapp"
}

# ❌ Неправильно
variable "2_instance" {}     # Начинается с цифры
variable "my var" {}         # Содержит пробелы
source "aws" "my-app" {}    # Используйте подчёркивания
```

### Интерполяция

```hcl
# HCL интерполяция ${...}
ami_name = "${var.app_name}-${var.version}"

# Packer template engine {{...}}
ami_name = "app-{{timestamp}}"
ami_name = "app-{{uuid}}"

# Комбинирование
ami_name = "${var.app_name}-{{timestamp}}"

# Packer template переменные:
# {{timestamp}}    - Unix timestamp
# {{uuid}}         - UUID v4
# {{build_name}}   - Имя build
# {{build_type}}   - Тип builder
# {{template_dir}} - Директория template
```

---

## Структура Template

### Минимальный template

```hcl
packer {
  required_version = ">= 1.9.0"
}

source "amazon-ebs" "example" {
  ami_name      = "packer-example-{{timestamp}}"
  instance_type = "t2.micro"
  region        = "us-east-1"
  source_ami    = "ami-0c55b159cbfafe1f0"
  ssh_username  = "ubuntu"
}

build {
  sources = ["source.amazon-ebs.example"]
}
```

### Полный template

```hcl
# 1. Packer блок
packer {
  required_version = ">= 1.9.0"
  required_plugins {
    amazon = {
      version = ">= 1.2.0"
      source  = "github.com/hashicorp/amazon"
    }
  }
}

# 2. Переменные
variable "region" {
  type    = string
  default = "us-east-1"
}

variable "instance_type" {
  type = string
}

# 3. Локальные переменные
locals {
  timestamp  = formatdate("YYYY-MM-DD-hhmm", timestamp())
  image_name = "${var.app_name}-${local.timestamp}"
  
  common_tags = {
    Project   = "MyApp"
    ManagedBy = "Packer"
  }
}

# 4. Data sources
data "amazon-ami" "ubuntu" {
  filters = {
    name = "ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"
  }
  most_recent = true
  owners      = ["099720109477"]
  region      = var.region
}

# 5. Sources
source "amazon-ebs" "web" {
  ami_name      = local.image_name
  instance_type = var.instance_type
  region        = var.region
  source_ami    = data.amazon-ami.ubuntu.id
  ssh_username  = "ubuntu"
  tags          = local.common_tags
}

# 6. Build
build {
  sources = ["source.amazon-ebs.web"]
  
  provisioner "shell" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y nginx"
    ]
  }
  
  post-processor "manifest" {
    output = "manifest.json"
  }
}
```

### Организация файлов

```
project/
├── template.pkr.hcl          # Главный template
├── variables.pkr.hcl         # Переменные
├── sources.pkr.hcl           # Source блоки
├── builds.pkr.hcl            # Build блоки
├── prod.pkrvars.hcl          # Prod значения
├── dev.pkrvars.hcl           # Dev значения
├── scripts/
│   ├── install.sh
│   └── cleanup.sh
└── files/
    └── config.conf
```

---

## Переменные

### Объявление

```hcl
# Базовая переменная
variable "region" {
  type    = string
  default = "us-east-1"
}

# С описанием
variable "instance_type" {
  type        = string
  description = "EC2 instance type"
  default     = "t2.micro"
}

# Обязательная (без default)
variable "ami_name" {
  type        = string
  description = "Name for AMI"
}

# С валидацией
variable "environment" {
  type = string
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Must be dev, staging, or prod"
  }
}

# Чувствительная
variable "api_key" {
  type      = string
  sensitive = true
}

# Список
variable "regions" {
  type    = list(string)
  default = ["us-east-1", "us-west-2"]
}

# Map
variable "tags" {
  type = map(string)
  default = {
    Environment = "prod"
  }
}

# Объект
variable "config" {
  type = object({
    name = string
    size = number
  })
  default = {
    name = "default"
    size = 20
  }
}

# Число
variable "disk_size" {
  type    = number
  default = 20
}

# Булево
variable "enable_monitoring" {
  type    = bool
  default = true
}
```

### Использование

```hcl
source "amazon-ebs" "ubuntu" {
  ami_name      = var.ami_name
  instance_type = var.instance_type
  region        = var.region
  
  tags = var.tags
}

# В строках
ami_name = "${var.app_name}-${var.version}"

# В условиях
instance_type = var.environment == "prod" ? "t2.large" : "t2.micro"
```

### Передача значений

```bash
# 1. Командная строка
packer build -var="region=us-west-2" template.pkr.hcl

# 2. Файлы переменных
# prod.pkrvars.hcl
region        = "us-east-1"
instance_type = "t2.large"

packer build -var-file="prod.pkrvars.hcl" template.pkr.hcl

# 3. Переменные окружения
export PKR_VAR_region="us-west-2"
packer build template.pkr.hcl

# 4. Интерактивный ввод
packer build template.pkr.hcl
# Packer запросит значения
```

---

## Локальные переменные

```hcl
locals {
  # Простые значения
  app_name = "myapp"
  
  # Timestamp
  timestamp = formatdate("YYYY-MM-DD-hhmm", timestamp())
  
  # Вычисляемые
  image_name = "${local.app_name}-${local.timestamp}"
  
  # Условные
  instance_type = var.environment == "prod" ? "t2.large" : "t2.micro"
  
  # Общие теги
  common_tags = {
    Application = local.app_name
    BuildDate   = local.timestamp
    ManagedBy   = "Packer"
  }
  
  # Списки
  availability_zones = ["${var.region}a", "${var.region}b"]
  
  # Maps
  region_amis = {
    us-east-1 = "ami-12345"
    us-west-2 = "ami-67890"
  }
  
  # Парсинг
  config = jsondecode(file("config.json"))
  
  # Слияние
  all_tags = merge(
    local.common_tags,
    var.custom_tags
  )
}

# Использование
source "amazon-ebs" "ubuntu" {
  ami_name      = local.image_name
  instance_type = local.instance_type
  tags          = local.all_tags
}
```

---

## Data Sources

### Amazon AMI

```hcl
data "amazon-ami" "ubuntu" {
  filters = {
    name                = "ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"
    root-device-type    = "ebs"
    virtualization-type = "hvm"
  }
  most_recent = true
  owners      = ["099720109477"]
  region      = var.region
}

# Использование
source "amazon-ebs" "web" {
  source_ami = data.amazon-ami.ubuntu.id
  # ...
}

# Доступные атрибуты:
# data.amazon-ami.ubuntu.id
# data.amazon-ami.ubuntu.name
# data.amazon-ami.ubuntu.creation_date
```

### Amazon Secrets Manager

```hcl
data "amazon-secretsmanager" "db_password" {
  name   = "prod/database/password"
  region = var.region
}

# Использование
provisioner "shell" {
  environment_vars = [
    "DB_PASSWORD=${data.amazon-secretsmanager.db_password.secret_string}"
  ]
  inline = ["echo $DB_PASSWORD"]
}

# JSON секреты
data "amazon-secretsmanager" "api_keys" {
  name   = "prod/api/keys"
  region = var.region
}

locals {
  api_keys = jsondecode(data.amazon-secretsmanager.api_keys.secret_string)
}
```

### HTTP

```hcl
data "http" "my_ip" {
  url = "https://api.ipify.org?format=json"
}

locals {
  my_public_ip = jsondecode(data.http.my_ip.body).ip
}

# Ограничение SSH доступа
source "amazon-ebs" "web" {
  temporary_security_group_source_cidrs = ["${local.my_public_ip}/32"]
}
```

### Git Commit

```hcl
data "git-commit" "current" {
  path = "${path.root}/.git"
}

locals {
  git_commit = data.git-commit.current.hash
  git_branch = data.git-commit.current.branch
}

source "amazon-ebs" "web" {
  ami_name = "app-${substr(local.git_commit, 0, 7)}-{{timestamp}}"
  
  tags = {
    GitCommit = local.git_commit
    GitBranch = local.git_branch
  }
}
```

---

## Source блоки

### Amazon EBS

```hcl
source "amazon-ebs" "ubuntu" {
  # Обязательные
  ami_name      = "my-ami-{{timestamp}}"
  instance_type = "t2.micro"
  region        = "us-east-1"
  source_ami    = "ami-12345"
  ssh_username  = "ubuntu"
  
  # Поиск AMI
  source_ami_filter {
    filters = {
      name                = "ubuntu/images/hvm-ssd/ubuntu-focal-*"
      root-device-type    = "ebs"
      virtualization-type = "hvm"
    }
    most_recent = true
    owners      = ["099720109477"]
  }
  
  # SSH
  ssh_timeout  = "10m"
  ssh_interface = "public_ip"
  
  # VPC
  vpc_id    = "vpc-12345"
  subnet_id = "subnet-67890"
  
  # Security Groups
  security_group_ids = ["sg-12345"]
  
  # EBS
  ebs_optimized = true
  
  launch_block_device_mappings {
    device_name           = "/dev/sda1"
    volume_size           = 20
    volume_type           = "gp3"
    encrypted             = true
    delete_on_termination = true
  }
  
  # AMI
  ami_description = "My Ubuntu Image"
  ami_regions     = ["us-east-1", "us-west-2"]
  
  # Теги
  tags = {
    Name = "MyAMI"
    OS   = "Ubuntu"
  }
  
  run_tags = {
    Name = "Packer Builder"
  }
  
  # IAM
  iam_instance_profile = "packer-role"
  
  # Spot instance
  spot_price = "auto"
}
```

### Azure ARM

```hcl
source "azure-arm" "ubuntu" {
  # Аутентификация
  client_id       = var.azure_client_id
  client_secret   = var.azure_client_secret
  subscription_id = var.azure_subscription_id
  tenant_id       = var.azure_tenant_id
  
  # Image
  managed_image_name                = "my-image-{{timestamp}}"
  managed_image_resource_group_name = "images-rg"
  
  # Source
  image_publisher = "Canonical"
  image_offer     = "UbuntuServer"
  image_sku       = "18.04-LTS"
  
  # VM
  location = "East US"
  vm_size  = "Standard_DS2_v2"
  
  # OS
  os_type = "Linux"
  
  # Tags
  azure_tags = {
    Environment = "Production"
  }
}
```

### Google Compute

```hcl
source "googlecompute" "ubuntu" {
  # Аутентификация
  account_file = "account.json"
  project_id   = "my-project"
  zone         = "us-central1-a"
  
  # Image
  image_name   = "my-image-{{timestamp}}"
  image_family = "ubuntu-2004-lts"
  
  # Source
  source_image        = "ubuntu-2004-focal-v20210119"
  source_image_family = "ubuntu-2004-lts"
  
  # Machine
  machine_type = "n1-standard-1"
  disk_size    = 20
  disk_type    = "pd-ssd"
  
  # SSH
  ssh_username = "packer"
}
```

### Docker

```hcl
source "docker" "ubuntu" {
  image  = "ubuntu:20.04"
  commit = true
  
  # Changes (Dockerfile-подобные команды)
  changes = [
    "EXPOSE 80",
    "WORKDIR /app",
    "ENV NODE_ENV production",
    "CMD [\"npm\", \"start\"]"
  ]
  
  # Volumes
  volumes = {
    "/host/path" = "/container/path"
  }
  
  # Environment
  environment_vars = [
    "DEBIAN_FRONTEND=noninteractive"
  ]
}
```

### VirtualBox ISO

```hcl
source "virtualbox-iso" "ubuntu" {
  # ISO
  iso_url      = "https://releases.ubuntu.com/20.04/ubuntu-20.04.iso"
  iso_checksum = "sha256:..."
  
  # VM
  guest_os_type = "Ubuntu_64"
  vm_name       = "ubuntu-20.04"
  
  # Hardware
  cpus      = 2
  memory    = 2048
  disk_size = 20000
  
  # HTTP для preseed
  http_directory = "http"
  
  # Boot
  boot_command = [
    "<esc><wait>",
    "linux /install/vmlinuz auto=true url=http://{{ .HTTPIP }}:{{ .HTTPPort }}/preseed.cfg<enter>"
  ]
  boot_wait = "5s"
  
  # SSH
  ssh_username = "packer"
  ssh_password = "packer"
  ssh_timeout  = "30m"
  
  # Shutdown
  shutdown_command = "echo 'packer' | sudo -S shutdown -P now"
  
  # VirtualBox
  vboxmanage = [
    ["modifyvm", "{{.Name}}", "--memory", "2048"],
    ["modifyvm", "{{.Name}}", "--cpus", "2"]
  ]
}
```

### Null (для тестирования)

```hcl
source "null" "test" {
  communicator = "ssh"
  ssh_host     = "existing-server.com"
  ssh_username = "ubuntu"
  ssh_password = "password"
}
```

---

## Build блоки

### Базовая структура

```hcl
build {
  name        = "production-build"
  description = "Production web server"
  
  sources = [
    "source.amazon-ebs.ubuntu",
    "source.azure-arm.ubuntu"
  ]
  
  provisioner "shell" {
    inline = ["echo 'Building'"]
  }
  
  post-processor "manifest" {
    output = "manifest.json"
  }
}
```

### Source overrides

```hcl
build {
  sources = ["source.amazon-ebs.ubuntu"]
  
  source "source.amazon-ebs.ubuntu" {
    ami_name = "custom-name-{{timestamp}}"
  }
  
  provisioner "shell" {
    inline = ["echo 'Building'"]
  }
}
```

### Множественные билды

```hcl
build {
  name = "aws-build"
  sources = ["source.amazon-ebs.ubuntu"]
  provisioner "shell" {
    inline = ["echo 'AWS'"]
  }
}

build {
  name = "azure-build"
  sources = ["source.azure-arm.ubuntu"]
  provisioner "shell" {
    inline = ["echo 'Azure'"]
  }
}
```

### HCP Packer Registry

```hcl
build {
  name = "learn-packer"
  
  hcp_packer_registry {
    bucket_name = "my-app"
    description = "Web server images"
    
    bucket_labels = {
      "application" = "web"
    }
    
    build_labels = {
      "git-commit" = "${env("GIT_COMMIT")}"
    }
  }
  
  sources = ["source.amazon-ebs.ubuntu"]
}
```

---

## Provisioners

### Shell

```hcl
# Inline команды
provisioner "shell" {
  inline = [
    "sudo apt-get update",
    "sudo apt-get install -y nginx"
  ]
}

# Из файла
provisioner "shell" {
  script = "scripts/install.sh"
}

# Множественные скрипты
provisioner "shell" {
  scripts = [
    "scripts/update.sh",
    "scripts/install.sh"
  ]
}

# С переменными окружения
provisioner "shell" {
  environment_vars = [
    "APP_ENV=production",
    "DB_HOST=${var.db_host}"
  ]
  inline = ["echo $APP_ENV"]
}

# Expect disconnect (для reboot)
provisioner "shell" {
  inline            = ["sudo reboot"]
  expect_disconnect = true
  pause_after       = "30s"
}

# Условное выполнение
provisioner "shell" {
  only = ["amazon-ebs.ubuntu"]
  inline = ["echo 'AWS only'"]
}

provisioner "shell" {
  except = ["docker.alpine"]
  inline = ["echo 'All except docker'"]
}
```

### File

```hcl
# Копирование файла
provisioner "file" {
  source      = "configs/app.conf"
  destination = "/tmp/app.conf"
}

# Копирование директории
provisioner "file" {
  source      = "configs/"
  destination = "/tmp/configs"
}

# Генерация содержимого
provisioner "file" {
  content = templatefile("templates/config.tpl", {
    db_host = var.db_host
  })
  destination = "/tmp/config.yaml"
}

# Download
provisioner "file" {
  source      = "/var/log/app.log"
  destination = "logs/app.log"
  direction   = "download"
}
```

### Ansible

```hcl
provisioner "ansible" {
  playbook_file = "ansible/playbook.yml"
  
  extra_arguments = [
    "--extra-vars",
    "app_env=production"
  ]
  
  ansible_env_vars = [
    "ANSIBLE_HOST_KEY_CHECKING=False"
  ]
  
  galaxy_file = "requirements.yml"
  user        = "ubuntu"
}

# Ansible Local (на целевой машине)
provisioner "ansible-local" {
  playbook_file   = "playbook.yml"
  install_ansible = true
}
```

### PowerShell (Windows)

```hcl
provisioner "powershell" {
  inline = [
    "Write-Host 'Installing IIS'",
    "Install-WindowsFeature Web-Server"
  ]
}

provisioner "powershell" {
  script = "scripts/install.ps1"
}

provisioner "powershell" {
  elevated_user     = "Administrator"
  elevated_password = build.Password
  inline            = ["Write-Host 'Elevated'"]
}
```

### Windows Restart

```hcl
provisioner "windows-restart" {
  restart_timeout = "10m"
}
```

### Breakpoint (отладка)

```hcl
provisioner "breakpoint" {
  note = "Check nginx: ssh ubuntu@${build.Host}"
}
```

---

## Post-Processors

### Manifest

```hcl
post-processor "manifest" {
  output     = "manifest.json"
  strip_path = true
  custom_data = {
    build_time = timestamp()
    git_commit = env("GIT_COMMIT")
  }
}
```

### Docker

```hcl
post-processor "docker-import" {
  repository = "myapp"
  tag        = "latest"
}

post-processor "docker-tag" {
  repository = "registry.com/myapp"
  tags       = ["latest", "1.0.0"]
}

post-processor "docker-push" {
  login          = true
  login_username = var.docker_username
  login_password = var.docker_password
}
```

### Compress

```hcl
post-processor "compress" {
  output            = "builds/{{.BuildName}}.tar.gz"
  compression_level = 9
}
```

### Checksum

```hcl
post-processor "checksum" {
  checksum_types = ["md5", "sha256"]
  output         = "{{.BuildName}}_{{.ChecksumType}}.checksum"
}
```

### Shell Local

```hcl
post-processor "shell-local" {
  inline = [
    "echo 'AMI ID: ${build.ID}' >> builds.log"
  ]
}

post-processor "shell-local" {
  script = "scripts/notify.sh"
  environment_vars = [
    "AMI_ID=${build.ID}"
  ]
}
```

### Цепочка

```hcl
build {
  sources = ["source.docker.ubuntu"]
  
  # Последовательная цепочка
  post-processors {
    post-processor "docker-tag" {
      repository = "myapp"
      tags       = ["latest"]
    }
    post-processor "docker-push" {}
  }
  
  # Параллельная обработка
  post-processors {
    post-processor "compress" {
      output = "image.tar.gz"
    }
  }
}
```

---

## Функции

### Строковые

```hcl
locals {
  # upper/lower
  upper_name = upper("hello")  # "HELLO"
  lower_name = lower("HELLO")  # "hello"
  
  # format
  formatted = format("%s-%s", var.app, var.env)
  
  # join/split
  joined = join("-", ["web", "server"])  # "web-server"
  parts  = split("-", "web-server")      # ["web", "server"]
  
  # replace
  replaced = replace("hello-world", "-", "_")  # "hello_world"
  
  # trim/substr
  trimmed   = trim("  hello  ", " ")  # "hello"
  substring = substr("hello", 0, 3)    # "hel"
  
  # regex
  matched = regex("[0-9]+", "version-123")  # ["123"]
}
```

### Дата/время

```hcl
locals {
  # timestamp
  now = timestamp()  # "2024-01-01T12:00:00Z"
  
  # formatdate
  date = formatdate("YYYY-MM-DD", timestamp())
  time = formatdate("YYYY-MM-DD-hhmm", timestamp())
  
  # Форматы:
  # YYYY - год (2024)
  # MM   - месяц (01-12)
  # DD   - день (01-31)
  # hh   - часы (00-23)
  # mm   - минуты (00-59)
  # ss   - секунды (00-59)
}
```

### Коллекции

```hcl
locals {
  # concat
  combined = concat(["a"], ["b"])  # ["a", "b"]
  
  # contains
  has = contains(["a", "b"], "a")  # true
  
  # length
  count = length([1, 2, 3])  # 3
  
  # merge
  merged = merge({a = 1}, {b = 2})  # {a = 1, b = 2}
  
  # keys/values
  map_keys = keys({a = 1, b = 2})    # ["a", "b"]
  map_vals = values({a = 1, b = 2})  # [1, 2]
  
  # lookup
  value = lookup({a = 1}, "b", 0)  # 0
  
  # sort/reverse
  sorted   = sort(["c", "a", "b"])  # ["a", "b", "c"]
  reversed = reverse([1, 2, 3])     # [3, 2, 1]
  
  # distinct/flatten
  unique = distinct([1, 1, 2])           # [1, 2]
  flat   = flatten([[1, 2], [3, 4]])    # [1, 2, 3, 4]
  
  # zipmap
  zipped = zipmap(["a", "b"], [1, 2])  # {a = 1, b = 2}
}
```

### Кодирование

```hcl
locals {
  # base64
  encoded = base64encode("hello")
  decoded = base64decode("aGVsbG8=")
  
  # json
  json   = jsonencode({name = "John"})
  parsed = jsondecode("{\"name\":\"John\"}")
  
  # yaml
  yaml      = yamlencode({name = "John"})
  from_yaml = yamldecode("name: John")
  
  # url
  url_encoded = urlencode("hello world")
}
```

### Файловые

```hcl
locals {
  # file/fileexists
  content = file("${path.root}/config.txt")
  exists  = fileexists("${path.root}/config.txt")
  
  # templatefile
  rendered = templatefile("${path.root}/template.tpl", {
    app_name = var.app_name
  })
  
  # basename/dirname
  filename  = basename("/path/to/file.txt")  # "file.txt"
  directory = dirname("/path/to/file.txt")   # "/path/to"
  
  # Path переменные
  root_dir    = path.root  # Корень template
  current_dir = path.cwd   # Текущая директория
}
```

### Числовые

```hcl
locals {
  # min/max
  minimum = min(5, 12, 9)  # 5
  maximum = max(5, 12, 9)  # 12
  
  # abs/ceil/floor
  absolute = abs(-42)   # 42
  ceiling  = ceil(4.3)  # 5
  flooring = floor(4.8) # 4
  
  # pow
  power = pow(2, 8)  # 256
}
```

### Условные

```hcl
locals {
  # Тернарный оператор
  type = var.env == "prod" ? "large" : "small"
  
  # try (безопасный доступ)
  value = try(var.optional, "default")
  
  # can (проверка возможности)
  can_parse = can(tonumber("123"))  # true
  
  # coalesce (первое непустое)
  result = coalesce(var.opt1, var.opt2, "default")
}
```

### Типы

```hcl
locals {
  # Преобразование типов
  as_string = tostring(123)     # "123"
  as_number = tonumber("123")   # 123
  as_bool   = tobool("true")    # true
  as_list   = tolist(["a", "b"])
  as_set    = toset(["a", "b", "a"])  # ["a", "b"]
  as_map    = tomap({a = 1})
}
```

### Хеш

```hcl
locals {
  # Хеш функции
  hash_md5    = md5("hello")
  hash_sha256 = sha256("hello")
  hash_sha512 = sha512("hello")
  
  # base64sha256
  b64_hash = base64sha256("hello")
  
  # uuid
  unique_id = uuid()
}
```

### Сетевые

```hcl
locals {
  # cidrhost
  host_ip = cidrhost("10.0.0.0/24", 5)  # "10.0.0.5"
  
  # cidrnetmask
  netmask = cidrnetmask("10.0.0.0/24")  # "255.255.255.0"
  
  # cidrsubnet
  subnet = cidrsubnet("10.0.0.0/16", 8, 1)  # "10.0.1.0/24"
}
```

### Packer-специфичные

```hcl
locals {
  # env (переменные окружения)
  home_dir   = env("HOME")
  git_commit = env("GIT_COMMIT")
  
  # Проверка существования
  has_git = env("GIT_COMMIT") != ""
}

# Build переменные (в provisioners/post-processors)
# ${build.ID}                - ID образа (AMI ID, etc.)
# ${build.PackerBuildName}   - Имя build
# ${build.PackerBuilderType} - Тип builder
# ${build.SourceName}        - Имя source
# ${build.Host}              - Хост для подключения
# ${build.Port}              - Порт для подключения
# ${build.User}              - Пользователь
# ${build.Password}          - Пароль (для WinRM)
```

---

## Best Practices

### Именование образов

```hcl
locals {
  timestamp = formatdate("YYYY-MM-DD-hhmm", timestamp())
  
  # ✅ Включайте важную информацию
  image_name = join("-", [
    var.app_name,      # Название приложения
    var.environment,   # Окружение
    var.version,       # Версия
    local.timestamp    # Timestamp
  ])
  # Результат: "myapp-prod-1.2.3-2024-01-01-1200"
}

source "amazon-ebs" "ubuntu" {
  ami_name = local.image_name
  
  tags = {
    Name        = local.image_name
    Application = var.app_name
    Version     = var.version
    Environment = var.environment
    BuildDate   = local.timestamp
    GitCommit   = env("GIT_COMMIT")
  }
}
```

### Управление секретами

```hcl
# ❌ НИКОГДА не храните секреты в коде
variable "aws_secret" {
  default = "SECRET123"  # НЕ ДЕЛАЙТЕ ТАК!
}

# ✅ Используйте переменные окружения
variable "aws_access_key" {
  type      = string
  sensitive = true
}

# export PKR_VAR_aws_access_key="..."

# ✅ Или secrets manager
data "amazon-secretsmanager" "creds" {
  name   = "packer/aws-creds"
  region = "us-east-1"
}

# ✅ Или IAM roles (лучший вариант для AWS)
source "amazon-ebs" "ubuntu" {
  # Не указывайте ключи
  # Packer использует IAM role автоматически
  region = "us-east-1"
}
```

### Идемпотентность

```hcl
# ❌ Плохо
provisioner "shell" {
  inline = ["apt-get install nginx"]
}

# ✅ Хорошо
provisioner "shell" {
  inline = [
    "apt-get update",
    "apt-get install -y nginx || true"
  ]
}

# ✅ Используйте configuration management
provisioner "ansible" {
  playbook_file = "playbook.yml"
}
```

### Cleanup

```hcl
provisioner "shell" {
  inline = [
    "echo 'Cleanup...'",
    
    # APT
    "sudo apt-get clean",
    "sudo apt-get autoremove -y",
    "sudo rm -rf /var/lib/apt/lists/*",
    
    # Logs
    "sudo rm -rf /var/log/*.log",
    "sudo find /var/log -type f -name '*.gz' -delete",
    
    # Temp
    "sudo rm -rf /tmp/*",
    "sudo rm -rf /var/tmp/*",
    
    # SSH keys
    "sudo rm -f /root/.ssh/authorized_keys",
    "sudo rm -f /home/*/.ssh/authorized_keys",
    
    # History
    "sudo rm -f /root/.bash_history",
    "sudo rm -f /home/*/.bash_history",
    "history -c",
    
    # Machine ID
    "sudo truncate -s 0 /etc/machine-id",
    
    # Cloud-init
    "sudo cloud-init clean --logs --seed",
    
    # Zero free space
    "sudo dd if=/dev/zero of=/EMPTY bs=1M || true",
    "sudo rm -f /EMPTY",
    
    "sync"
  ]
}
```

### Валидация

```hcl
variable "version" {
  type = string
  
  validation {
    condition     = can(regex("^[0-9]+\\.[0-9]+\\.[0-9]+$", var.version))
    error_message = "Version must be semver (e.g., 1.2.3)"
  }
}

variable "instance_type" {
  type = string
  
  validation {
    condition = contains([
      "t2.micro", "t2.small", "t2.medium"
    ], var.instance_type)
    error_message = "Invalid instance type"
  }
}
```

### Оптимизация времени

```hcl
# 1. Более мощные instances
source "amazon-ebs" "ubuntu" {
  instance_type = "c5.2xlarge"
}

# 2. Объединяйте команды
provisioner "shell" {
  inline = [
    "apt-get update && apt-get install -y pkg1 pkg2 pkg3"
  ]
}

# 3. Параллелизация
# packer build -parallel-builds=4 .
```

### Multi-region

```hcl
# Копирование в несколько регионов
source "amazon-ebs" "ubuntu" {
  ami_name    = "myapp-{{timestamp}}"
  region      = "us-east-1"
  ami_regions = ["us-west-2", "eu-west-1"]
}
```

---

## Примеры

### Пример 1: AWS Ubuntu AMI

```hcl
packer {
  required_version = ">= 1.9.0"
  required_plugins {
    amazon = {
      version = ">= 1.2.0"
      source  = "github.com/hashicorp/amazon"
    }
  }
}

variable "region" {
  type    = string
  default = "us-east-1"
}

locals {
  timestamp = formatdate("YYYY-MM-DD-hhmm", timestamp())
}

data "amazon-ami" "ubuntu" {
  filters = {
    name = "ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"
  }
  most_recent = true
  owners      = ["099720109477"]
  region      = var.region
}

source "amazon-ebs" "ubuntu" {
  ami_name      = "nginx-${local.timestamp}"
  instance_type = "t2.micro"
  region        = var.region
  source_ami    = data.amazon-ami.ubuntu.id
  ssh_username  = "ubuntu"
  
  tags = {
    Name      = "Nginx Server"
    BuildDate = local.timestamp
  }
}

build {
  sources = ["source.amazon-ebs.ubuntu"]
  
  provisioner "shell" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y nginx",
      "sudo systemctl enable nginx"
    ]
  }
  
  provisioner "shell" {
    inline = [
      "sudo apt-get clean",
      "sudo rm -rf /var/lib/apt/lists/*"
    ]
  }
  
  post-processor "manifest" {
    output = "manifest.json"
  }
}
```

### Пример 2: Docker Image

```hcl
packer {
  required_version = ">= 1.9.0"
  required_plugins {
    docker = {
      version = ">= 1.0.0"
      source  = "github.com/hashicorp/docker"
    }
  }
}

variable "app_version" {
  type = string
}

source "docker" "node" {
  image  = "node:18-alpine"
  commit = true
  
  changes = [
    "EXPOSE 3000",
    "WORKDIR /app",
    "ENV NODE_ENV=production",
    "CMD [\"node\", \"server.js\"]"
  ]
}

build {
  sources = ["source.docker.node"]
  
  provisioner "file" {
    source      = "package.json"
    destination = "/app/package.json"
  }
  
  provisioner "shell" {
    inline = [
      "cd /app",
      "npm ci --only=production"
    ]
  }
  
  provisioner "file" {
    source      = "src/"
    destination = "/app/"
  }
  
  post-processors {
    post-processor "docker-tag" {
      repository = "myapp"
      tags       = ["latest", var.app_version]
    }
    
    post-processor "docker-push" {}
  }
}
```

### Пример 3: Multi-platform

```hcl
packer {
  required_version = ">= 1.9.0"
  required_plugins {
    amazon = {
      version = ">= 1.2.0"
      source  = "github.com/hashicorp/amazon"
    }
    azure = {
      version = ">= 1.4.0"
      source  = "github.com/hashicorp/azure"
    }
  }
}

variable "version" {
  type = string
}

locals {
  timestamp  = formatdate("YYYY-MM-DD-hhmm", timestamp())
  image_name = "webapp-${var.version}-${local.timestamp}"
  
  common_tags = {
    Version   = var.version
    BuildDate = local.timestamp
    ManagedBy = "Packer"
  }
}

# AWS
data "amazon-ami" "ubuntu" {
  filters = {
    name = "ubuntu/images/hvm-ssd/ubuntu-focal-*"
  }
  most_recent = true
  owners      = ["099720109477"]
  region      = "us-east-1"
}

source "amazon-ebs" "webapp" {
  ami_name      = local.image_name
  instance_type = "t2.micro"
  region        = "us-east-1"
  source_ami    = data.amazon-ami.ubuntu.id
  ssh_username  = "ubuntu"
  tags          = local.common_tags
}

# Azure
source "azure-arm" "webapp" {
  managed_image_name                = local.image_name
  managed_image_resource_group_name = "images-rg"
  os_type                           = "Linux"
  image_publisher                   = "Canonical"
  image_offer                       = "UbuntuServer"
  image_sku                         = "18.04-LTS"
  location                          = "East US"
  vm_size                           = "Standard_DS2_v2"
  azure_tags                        = local.common_tags
  
  client_id       = var.azure_client_id
  client_secret   = var.azure_client_secret
  subscription_id = var.azure_subscription_id
  tenant_id       = var.azure_tenant_id
}

build {
  sources = [
    "source.amazon-ebs.webapp",
    "source.azure-arm.webapp"
  ]
  
  provisioner "shell" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y nginx"
    ]
  }
  
  provisioner "shell" {
    script = "scripts/cleanup.sh"
  }
  
  post-processor "manifest" {
    output = "manifest.json"
  }
}
```

### Пример 4: Windows IIS

```hcl
packer {
  required_version = ">= 1.9.0"
  required_plugins {
    amazon = {
      version = ">= 1.2.0"
      source  = "github.com/hashicorp/amazon"
    }
  }
}

data "amazon-ami" "windows" {
  filters = {
    name = "Windows_Server-2022-English-Full-Base-*"
  }
  most_recent = true
  owners      = ["amazon"]
  region      = "us-east-1"
}

source "amazon-ebs" "windows" {
  ami_name      = "windows-iis-{{timestamp}}"
  instance_type = "t3.medium"
  region        = "us-east-1"
  source_ami    = data.amazon-ami.windows.id
  
  communicator   = "winrm"
  winrm_username = "Administrator"
  winrm_use_ssl  = true
  winrm_insecure = true
  
  user_data_file = "scripts/setup-winrm.ps1"
}

build {
  sources = ["source.amazon-ebs.windows"]
  
  provisioner "powershell" {
    inline = [
      "Install-WindowsFeature -Name Web-Server",
      "Install-WindowsFeature -Name Web-Asp-Net45"
    ]
  }
  
  provisioner "windows-restart" {}
  
  provisioner "powershell" {
    script = "scripts/configure-iis.ps1"
  }
  
  provisioner "powershell" {
    script = "scripts/cleanup.ps1"
  }
}
```

### Пример 5: Golden Image с тестами

```hcl
packer {
  required_version = ">= 1.9.0"
  required_plugins {
    amazon = {
      version = ">= 1.2.0"
      source  = "github.com/hashicorp/amazon"
    }
  }
}

variable "version" {
  type = string
  
  validation {
    condition     = can(regex("^[0-9]+\\.[0-9]+\\.[0-9]+$", var.version))
    error_message = "Version must be semver format"
  }
}

locals {
  timestamp  = formatdate("YYYY-MM-DD-hhmm", timestamp())
  image_name = "golden-${var.version}-${local.timestamp}"
}

data "amazon-ami" "ubuntu" {
  filters = {
    name = "ubuntu/images/hvm-ssd/ubuntu-focal-*"
  }
  most_recent = true
  owners      = ["099720109477"]
  region      = "us-east-1"
}

source "amazon-ebs" "golden" {
  ami_name      = local.image_name
  instance_type = "t2.micro"
  region        = "us-east-1"
  source_ami    = data.amazon-ami.ubuntu.id
  ssh_username  = "ubuntu"
  ami_regions   = ["us-west-2", "eu-west-1"]
  
  tags = {
    Name      = local.image_name
    Version   = var.version
    Type      = "GoldenImage"
    BuildDate = local.timestamp
    GitCommit = env("GIT_COMMIT")
  }
}

build {
  sources = ["source.amazon-ebs.golden"]
  
  # Updates
  provisioner "shell" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get upgrade -y"
    ]
  }
  
  # Common tools
  provisioner "shell" {
    script = "scripts/install-tools.sh"
  }
  
  # Security hardening
  provisioner "shell" {
    script = "scripts/hardening.sh"
  }
  
  # Validation
  provisioner "shell" {
    scripts = [
      "tests/validate-packages.sh",
      "tests/validate-security.sh"
    ]
  }
  
  # Cleanup
  provisioner "shell" {
    script = "scripts/cleanup.sh"
  }
  
  post-processor "manifest" {
    output = "manifest-${var.version}.json"
    custom_data = {
      version    = var.version
      build_date = local.timestamp
      git_commit = env("GIT_COMMIT")
    }
  }
  
  post-processor "shell-local" {
    inline = [
      "echo 'Golden Image: ${local.image_name}'",
      "echo 'AMI ID: ${build.ID}' >> golden-images.log"
    ]
  }
}
```

---

## Команды CLI

### Основные команды

```bash
# Инициализация
packer init .
packer init template.pkr.hcl

# Форматирование
packer fmt .
packer fmt -recursive .
packer fmt -check .

# Валидация
packer validate .
packer validate template.pkr.hcl
packer validate -var="region=us-west-2" .
packer validate -var-file="prod.pkrvars.hcl" .

# Инспектирование
packer inspect template.pkr.hcl

# Build
packer build template.pkr.hcl
packer build -var="version=1.0.0" .
packer build -var-file="prod.pkrvars.hcl" .

# Селективный build
packer build -only="amazon-ebs.*" .
packer build -except="docker.*" .

# Параллелизация
packer build -parallel-builds=4 .

# Force rebuild
packer build -force .

# Debug
packer build -debug .
PACKER_LOG=1 packer build .

# On-error
packer build -on-error=ask .
packer build -on-error=cleanup .
packer build -on-error=abort .

# Консоль
packer console template.pkr.hcl
# > var.region
# > local.timestamp

# Версия
packer version

# Помощь
packer --help
packer build --help
```

### Переменные окружения

```bash
# Логирование
export PACKER_LOG=1
export PACKER_LOG=TRACE
export PACKER_LOG_PATH="packer.log"

# Переменные
export PKR_VAR_region="us-west-2"
export PKR_VAR_instance_type="t2.small"

# Кеш
export PACKER_CACHE_DIR="$HOME/.packer/cache"

# Плагины
export PACKER_PLUGIN_PATH="$HOME/.packer.d/plugins"

# HCP Packer
export HCP_CLIENT_ID="..."
export HCP_CLIENT_SECRET="..."

# Breakpoint
export PACKER_BREAKPOINT_SKIP=1
```

### Отладка

```bash
# Debug режим
packer build -debug template.pkr.hcl

# Логирование
PACKER_LOG=1 packer build .

# Уровни логов
export PACKER_LOG=TRACE  # Самый детальный
export PACKER_LOG=DEBUG
export PACKER_LOG=INFO
export PACKER_LOG=WARN
export PACKER_LOG=ERROR

# В файл
PACKER_LOG=1 PACKER_LOG_PATH="packer.log" packer build .
```

### CI/CD

```yaml
# GitHub Actions
name: Packer Build
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Packer
        uses: hashicorp/setup-packer@main
      
      - name: Init
        run: packer init .
      
      - name: Validate
        run: packer validate .
      
      - name: Build
        run: packer build .
        env:
          PKR_VAR_aws_access_key: ${{ secrets.AWS_ACCESS_KEY }}
          PKR_VAR_aws_secret_key: ${{ secrets.AWS_SECRET_KEY }}
```

```groovy
// Jenkins
pipeline {
  agent any
  
  environment {
    PKR_VAR_version = "${env.BUILD_NUMBER}"
  }
  
  stages {
    stage('Init') {
      steps {
        sh 'packer init .'
      }
    }
    
    stage('Validate') {
      steps {
        sh 'packer validate .'
      }
    }
    
    stage('Build') {
      steps {
        sh 'packer build .'
      }
    }
  }
}
```

---

## Быстрая справка

### Структура Template

```hcl
packer { }          # Настройки
variable { }        # Переменные
locals { }          # Локальные значения
data { }            # Data sources
source { }          # Builders
build { }           # Процесс сборки
  provisioner { }   # Настройка
  post-processor {} # Обработка
```

### Типы данных

```hcl
string          # "text"
number          # 42
bool            # true/false
list(string)    # ["a", "b"]
map(string)     # {key = "value"}
object({...})   # Сложный объект
```

### Интерполяция

```hcl
${var.name}         # Переменная
${local.value}      # Локальное значение
${data.type.name}   # Data source
${env("VAR")}       # Переменная окружения
{{timestamp}}       # Packer функция
${build.ID}         # Build переменная
```

### Популярные функции

```hcl
# Строки
upper(), lower(), trim(), replace()
format(), split(), join()

# Коллекции
length(), concat(), merge()
keys(), values(), contains()

# Кодирование
jsonencode(), jsondecode()
base64encode(), base64decode()

# Дата
timestamp(), formatdate()

# Условия
try(), can(), coalesce()
```

### Builders

```hcl
amazon-ebs          # AWS AMI
azure-arm           # Azure Image
googlecompute       # GCE Image
docker              # Docker Image
virtualbox-iso      # VirtualBox
vmware-iso          # VMware
```

### Provisioners

```hcl
shell               # Shell/Bash
powershell          # PowerShell
file                # Копирование файлов
ansible             # Ansible
ansible-local       # Ansible на машине
windows-restart     # Перезагрузка Windows
breakpoint          # Остановка
```

### Post-Processors

```hcl
manifest            # Манифест
compress            # Сжатие
checksum            # Checksums
shell-local         # Локальные команды
docker-tag          # Docker tag
docker-push         # Docker push
```

### Команды

```bash
packer init .       # Инициализация
packer fmt .        # Форматирование
packer validate .   # Валидация
packer build .      # Создание образа
packer console .    # Консоль

# С параметрами
packer build -var="key=value" .
packer build -var-file="vars.pkrvars.hcl" .
packer build -only="amazon-ebs.*" .
packer build -debug .
```

### Best Practices

```hcl
# ✅ DO
- Используйте HCL2 формат
- Версионируйте образы
- Добавляйте теги с метаданными
- Используйте data sources
- Делайте provisioners идемпотентными
- Добавляйте cleanup
- Валидируйте переменные
- Используйте secrets manager
- Тестируйте образы
- Автоматизируйте через CI/CD

# ❌ DON'T
- Не храните секреты в коде
- Не используйте хардкод
- Не игнорируйте cleanup
- Не создавайте монолитные provisioners
```

---

## Заключение

Это полная шпаргалка по Packer, которая покрывает:

✅ Базовый и продвинутый синтаксис HCL2
✅ Структуру templates
✅ Переменные и функции
✅ Data sources
✅ Все типы блоков (source, build, provisioner, post-processor)
✅ Builders для популярных платформ
✅ Best practices
✅ Реальные примеры
✅ Команды CLI и отладку
✅ CI/CD интеграцию
✅ Быструю справку

Используйте как полноценный справочник для работы с Packer! 🚀