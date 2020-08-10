# **Начало**

Выполняя данное задание, я познакомился  с инструментами, `Vagrant` и `Packer`, получил базовые навыки работы с системами контроля версий (`Github`). 
Создал образы виртуальных машин, один из которых поместил в репозиторий `Vagrant Cloud`.

Работа проводилась на машине, под управлением Windows ОС
Для выполнения работы я использовал инструменты:

- **VirtualBox** - среда виртуализации, позволяет создавать и выполнять виртуальные машины;
- **Vagrant** - ПО для создания и конфигурирования виртуальной среды. В данном случае в качестве среды виртуализации используется *VirtualBox*;
- **Packer** - ПО для создания образов виртуальных машин;
- **Git** - система контроля версий

И аккаунты:

- **GitHub** - https://github.com/
- **Vagrant Cloud** - https://app.vagrantup.com

# **Сам процесс**

### **установка Vagrant**
Перейдя по адресу https://www.vagrantup.com/downloads.html выбираем версию для Windows 2.2.9.
После скачивания запускаем полученный файл и устанавливаем Vagrant.

### **Установка Packer**
Перейдя, по ссылке https://www.packer.io/downloads.html выбираем версию для Windows 1.6.1.
После скачивания распаковываем архив в папку.(у меня C:\Packer ).

### **Установка VirtualBox**
Перейдя, по ссылке https://www.virtualbox.org выбираем версию для Windows VirtualBox 6.1.12

# **Создание с помощью Packer образа системы**
Через командную строку, заходим в директорию `packer`
cd C:\Packer

с помощью блокнота создаём файл CentOS с расширением .json 

```{
  "builders": [
    {
      "boot_command": [
        "<tab> text ks=http://{{ .HTTPIP }}:{{ .HTTPPort }}/vagrant.ks<enter><wait>"
      ],
      "boot_wait": "10s",
      "disk_size": "10240",
      "export_opts": [
        "--manifest",
        "--vsys",
        "0",
        "--description",
        "{{user `artifact_description`}}",
        "--version",
        "{{user `artifact_version`}}"
      ],
      "http_directory": "http",
      "iso_checksum": "sha256:659691c28a0e672558b003d223f83938f254b39875ee7559d1a4a14c79173193",
      "iso_url": "http://mirror.yandex.ru/centos/7.8.2003/isos/x86_64/CentOS-7-x86_64-Minimal-2003.iso",
      "name": "{{user `image_name`}}",
      "output_directory": "builds",
      "shutdown_command": "sudo -S /sbin/halt -h -p",
      "shutdown_timeout": "5m",
      "ssh_password": "vagrant",
      "ssh_port": 22,
      "ssh_pty": true,
      "ssh_timeout": "20m",
      "ssh_username": "vagrant",
      "type": "virtualbox-iso",
      "vboxmanage": [
        [
          "modifyvm",
          "{{.Name}}",
          "--memory",
          "1024"
        ],
        [
          "modifyvm",
          "{{.Name}}",
          "--cpus",
          "2"
        ]
      ],
      "vm_name": "packer-centos-vm"
    }
  ],
  "post-processors": [
    {
      "compression_level": "7",
      "output": "centos-{{user `artifact_version`}}-kernel-5-x86_64-Minimal.box",
      "type": "vagrant"
    }
  ],
  "provisioners": [
    {
      "execute_command": "{{.Vars}} sudo -S -E bash '{{.Path}}'",
      "expect_disconnect": true,
      "override": {
        "{{user `image_name`}}": {
          "scripts": [
            "scripts/stage-1-kernel-update.sh",
            "scripts/stage-2-clean.sh"
          ]
        }
      },
      "pause_before": "20s",
      "start_retry_timeout": "1m",
      "type": "shell"
    }
  ],
  "variables": {
    "artifact_description": "CentOS 7.8 with kernel 5.x",
    "artifact_version": "7.8.2003",
    "image_name": "centos-7-8"
  }
}

Данный скрипт будет брать образ CentOS из репозитория на Yandex
http://mirror.yandex.ru/centos/7.8.2003/isos/x86_64/CentOS-7-x86_64-Minimal-2003.iso

В переменные (`variables`) заводим с версию, названием проекта и имя образа (artifact):
```
    "artifact_description": "CentOS 7.8 with kernel 5.x",
    "artifact_version": "7.8.2003",
    "image_name": "centos-7-8"
```

В секции `builders` задаётся исходный образ, для создания своего в виде ссылки и контрольной суммы. Параметры подключения к создаваемой виртуальной машине.

```
   "iso_checksum": "sha256:659691c28a0e672558b003d223f83938f254b39875ee7559d1a4a14c79173193",
   "iso_url": "http://mirror.yandex.ru/centos/7.8.2003/isos/x86_64/CentOS-7-x86_64-Minimal-2003.iso",
```
В секции `post-processors` указываем имя файла, куда будет сохранен образ, в случае успешной сборки

```
    "output": "centos-{{user `artifact_version`}}-kernel-5-x86_64-Minimal.box"
```

В секции `provisioners` указываем каким образом и какие действия необходимо произвести для настройки виртуальой машины. 
Настройка системы выполняется несколькими скриптами, заданными в секции `scripts`. Данные скрипты находятся в папке С:\packer\scripts

```
    "scripts" : 
      [
        "scripts/stage-1-kernel-update.sh",
        "scripts/stage-2-clean.sh"
      ]
```
Скрипты будут выполнены в порядке указания. Первый скрипт включает себя набор команд, которые обновляют ядро. 
Второй скрипт занимается подготовкой системы к упаковке в образ. Она заключается в очистке директорий с логами, временными файлами, кешами. 
Это позволяет уменьшить результирующий образ. 

### ** Создание образа **
Для создания образа системы переходим в директорию `C:\packer` и в ней выполняем команду:

```
packer build centos.json

Исходя файла `config.json` будет скачан исходный iso-образ CentOS, установлен на виртуальную машину в автоматическом режиме, 
обновлено ядро и осуществлен экспорт в указанный нами файл. 
Результатом работы `packer`, появился файл centos-7.8.2003-kernel-5-x86_64-Minimal.box

### **vagrant init (тестирование)**
Проведем тестирование созданного образа. Выполним его импорт в `vagrant`:

```
vagrant box add --name centos-7-8 centos-7.8.2003-kernel-5-x86_64-Minimal.box
```
vagrant box list
centos-7-8            (virtualbox, 0)
```

Он будет называться `centos-7-8`, данное имя мы задали при помощи параметра `name` при импорте.

# **помещение образа в Vagrant cloud**

Чтобы поделимся образом, его нужно зальем его в Vagrant Cloud. Можно залить через web-интерфейс, но так же `vagrant` позволяет это проделать через CLI.
Логинимся в `vagrant cloud`, указывая e-mail, пароль
```
vagrant cloud auth login
Vagrant Cloud username or email: <user_email>
Password (will be hidden): 
You are now logged in.
```
Теперь публикуем полученный бокс:
```
vagrant cloud publish --release <username>/centos-7-8 1.0 virtualbox centos-7.8.2003-kernel-5-x86_64-Minimal.box
```
Здесь:
 - `cloud publish` - загрузить образ в облако;
 - `release` - указывает на необходимость публикации образа после загрузки;
 - `<username>/centos-7-8` - `username`, указаный при публикации и имя образа;
 - `1.0` - версия образа;
 - `virtualbox` - провайдер;
 - `centos-7.8.2003-kernel-5-x86_64-Minimal.box` - имя файла загружаемого образа;

После успешной загрузки получаем сообщение:

```
Complete! Published <username>/centos-7-8
tag:             <username>/centos-7-8
username:        <username>
name:            centos-7-8
private:         false
...
providers:       virtualbox
```

**Жирный текст (bold)** В результате я склонировал с базового репозитория CentOS, создал свой образ с обновленным ядром и опубликовал его в `vagrant cloud`.
