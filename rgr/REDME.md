Розрахунково графічна робота

Виконав: студент Середа Олександр група 546

1. Підготовка GCP

Перше що треба зробити, це створити новий проект у GCP, віртуальну машину(ВМ), на якій в подальшому будемо працювати, та service account. Щоб автоматизувати ВМ, будемо використовувати Terraform, що розглядали у попередніх лабораторних роботах. Тож зробимо три основні .tf файли, за допомогою яких буде створена ВМ та всі налаштування:

main.tf:

```
terraform {
	required\_providers {
	google = {
	source  = "hashicorp/google"
	version = "4.51.0"
	}
    }
}
provider "google" {

	credentials = file(var.credentials\_file)
	project = var.project
	region  = var.region
	zone    = var.zone
}

resource "google\_compute\_network" "vpc\_network" {
	name = "rgr"
}

resource "google\_compute\_instance" "vm\_instance" {
	name         = var.machine\_name
	machine\_type = "e2-medium" # Don't need much for small server
	tags         = ["jenkins"]
	boot\_disk {
	initialize\_params {
	image = "ubuntu-os-cloud/ubuntu-minimal-2210-kinetic-amd64-v20230425"
	}
     }

	network\_interface {
	network = google\_compute\_network.vpc\_network.name
	access\_config {
	}

	}

}

resource "google\_compute\_firewall" "rules" {
	project       = var.project
	name          = "user"
	network       = google\_compute\_network.vpc\_network.self\_link
	source\_ranges = ["0.0.0.0/0"]
	allow {
	protocol = "tcp"
	ports    = ["20", "22", "80", "8080", "1000-2000"]
	}
	source\_tags = ["vpc"]
	target\_tags = ["jenkins"]
}
```

variables.tf

```
variable "project" {
	default = "laba3-382520"
}
variable "credentials\_file" {
	default = "laba3-382520-e42b62366afb.json"
}
variable "region" {
	default = "us-central1"
}
variable "zone" {
	default = "us-central1-c"
}
variable "machine\_name" {
	default = "laba3-1"
}
```

outputs.tf

```
output "ip\_intra" {
	value = google\_compute\_instance.vm\_instance.network\_interface.0.network\_ip
}
output "ip\_extra" {
	value = google\_compute\_instance.vm\_instance.network\_interface.0.access\_config.0.nat\_ip
}
```
Пояснимо, що саме записано вище у файлах. Звісно, було створено ВМ мінімально можливої конфігурації в us-central1-c зоні, з пераціною системою є ubuntu cloud. Також треба вірно сконфігурувати вбудований у GCP брандмауер, щоб він мав змогу надати доступ по SSH, потім доступ до Jenkins, а також дозволив роботу на порту сервера. Це можна зробити шляхом налаштування ресурсу google\_compute\_firewall. Створимо ВМ і мережі за відомою командою: 

```
terraform apply
```

Результат на рисунку 1:

![](Aspose.Words.16455021-3ae6-4294-819c-c283ba22e21b.001.png)Рисунок 1 – Створена ВМ з усім налаштуванням

1. Встановлення та створення аккаунту у середовищі Jenkins 

Перейдемо до ВМ та почнемо працювати, а саме, спочатку оновимо базу даних менеджеру пакетів apt:

```
sudo apt update
```

Далі встановлюємо текстовий редактор nano:

```
sudo apt install nano
```

Та звісно встановлюємо git:

```
sudo apt install git
```

Перейдемо до встановлення Jenkins. По-перше встановимо Java 11 версії:

```
sudo apt install openjdk-11-jre
```

Далі послідовно записуємо такі командні рядки, щоб встановити Jenkins:

```
curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee \/usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \https://pkg.jenkins.io/debian binary/ | sudo tee \/etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt-get update

sudo apt-get install jenkins
```

Після чого робимо перезавантаження та пишемо наступну команду: 

```
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Отримуємо пароль від Jenkins, після чого переходимо за  External IP адресою і портом 8080. Переходимо на сайт Jenkins та одразу ж нам пропонують ввести персональний пароль, який ми отримали раніше, тож введемо його та пройдемось по першому налаштуванню свого нового аккаунту. Повернемося до ВМ та зробимо ще деякі важливі речі. Для того, щоб Jenkins міг виконувати дії від імені root, треба зробити наступне в самій ВМ: 

```
cd /etc

sudo nano sudoers
```

Потім знаходимо рядок: 

```
\# User privilege specification

root    ALL=(ALL:ALL) ALL
```

Та пишемо нижче цього рядка таку команду: 

```
jenkins ALL=NOPASSWD: ALL
```

Тобто, це дозволить виконувати команди типу 'sh "sudo..."'

1. Переходимо до автоматизації CI/CD 

В Jenkins тиснемо "Create a Job" та створюємо новий Pipeline. Надаємо імя та вказуємо, що це проект GitHub. Пишемо посилання на наш проект ось таким посыланням:

```
<https://github.com/Alexandr-546/RGR_cus>
```

Далі пишемо скрипт мовою Groovy: 

```
pipeline 
{
	agent any

	options

	{

	disableConcurrentBuilds()

	}

	stages 

	{

		stage('Download') 

		{

		steps 

		{

		echo "Checkout GO"

		checkout scmGit(

			branches: [[name: 'main']],

			userRemoteConfigs:[[url: '<https://github.com/Alexandr-546/RGR_cus>']])

			echo "Checkout OKOK"

		}

		}

		stage('Setup') 

		{

		steps {

		echo "Setup Section GO"

		sh 'chmod +rwx ${WORKSPACE}/1449/Linux/TerrariaServer.bin.x86\_64'

			echo "Setup OKOK"

		}

		}

		stage('Run') 

		{

		steps 

		{

		echo "Run GO"

		sh "sudo ${WORKSPACE}/1449/Linux/TerrariaServer.bin.x86\_64 -config ${WORKSPACE}/1449/Linux/serverconfig.txt &"

		echo "Run OKOK"

		}

		}

	}

	post 

	{

		success

	{

		echo 'hellyeah'

	}

		failure

	{

	deleteDir()

	}

	}

}
```

Спочатку клонуємо репозиторій із сервером в робочу директорію, видаємо права на виконання, читання та запис файлу сервера, далі запускаємо сервер від імені root з читанням інформації із конфігурованого файлу, і вже після збірки буде виводитися повідомлення. Якщо збірка не мала успіху, тоді тека вичищається. Також важливо проставити галочки у пунктах: GitHub hook trigger for GITScm polling та SCM Changes. Тож запустимо зборку та подивимось, чи все гаразд з проектом. 

![](Aspose.Words.16455021-3ae6-4294-819c-c283ba22e21b.002.png)

Як можна побачити, що не з першого разу все зібралося, але все ж таки воно завантажилось, налаштувалося, запустилося. 

Перевіримо наш сайт, який заздалегідь завантажили на GitHub. 

![](Aspose.Words.16455021-3ae6-4294-819c-c283ba22e21b.003.png)

Так все працює. 

Висновок 

У даній РГР було створено ВМ за допомогою Terraform, що значно прискорює та автоматизує майже весь процес. Далі, на ВМ було створено Jenkins, за допомогою якого, можна робити зборку нашого проекту, який буде розташований у нашому GitHub. 



