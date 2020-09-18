# Практикум: развертывание отказоустойчивого приложения в Managed Service for Kubernetes® и MDB

# Требуемое окружение

В рамках практической работы окружение подготовлено в виде образа ВМ. Из этого образа будет создана виртуальная машина,
на которой и будет выполняться работа. Для самостоятельного выполнения работы придется настроить окружение самостоятельно.

Для выполнения практической работы потребуются следующие утилиты:
* yc (https://cloud.yandex.ru/docs/cli/quickstart);
* terraform (https://learn.hashicorp.com/tutorials/terraform/install-cli);
* docker (https://docs.docker.com/get-docker/);
* kubectl (https://kubernetes.io/docs/tasks/tools/install-kubectl/);
* envsubst;
* git;
* ssh;
* curl.

Также необходимо:
* Зарегистрироваться в Яндекс.Облаке (https://cloud.yandex.ru/);
* Создать облако и каталог, в котором будут создаваться ресурсы. 

# Подготовка окружения

## Настройка доступа по ssh
1. Убедитесь, что у вас есть ssh-ключ или сгенерируйте новый:
```
ssh-keygen -t rsa -b 2048 # генерация нового ssh-ключа
cat ~/.ssh/id_rsa.pub
```
Более подробная инструкция по ссылке: https://cloud.yandex.ru/docs/compute/operations/vm-connect/ssh#creating-ssh-keys

2. Создайте виртуальную машину в Консоли из образа `base-image`:
* В каталоге нажмите кнопку `Создать ресурс`, выберите ресурс `Виртуальная машина`, введите имя `lab-vm`;
`Удалять вместе с ВМ`, выберите наполнение из образа. Выберите образ `base-image`;
* В разделе `Доступ` введите логин для доступа к ВМ. Скопируйте SSH-ключ из файла `~/.ssh/id_rsa.pub`.

3. Дождитесь создания ВМ (должен быть статус `Running`);

4. Зайдите на созданную ВМ по ssh;

5. Дальнейшие команды будут выполняться на созданной ВМ.

## Настройка консольной утилиты yc для работы с Яндекс Облаком
1. Выполните в терминале `yc init`;
2. Перейдите по полученной ссылке вида `https://oauth.yandex.ru/authorize?response_type=token&client_id=1a...`; 
3. Полученную строку OAuth-token запишите, она нам потребуется несколько раз;
4. Используйте OAuth token в терминале для продолжения;
5. Выберите `cloud` и `folder`, выберите `compute zone` ru-central1-b;

6. Сохраните в переменные окружения значения `OAuth-token` и `folder`: 
```
export YC_FOLDER=$(yc config get folder-id)
export YC_TOKEN=$(yc config get token)

printenv YC_FOLDER
printenv YC_TOKEN

echo "export YC_FOLDER=$(yc config get folder-id)" >> ~/.bashrc
echo "export YC_TOKEN=$(yc config get token)" >> ~/.bashrc
```

7. Проверьте, что yc настроен корректно: выведите список виртуальных машин в каталоге с помощью `yc`:
```
yc compute instance list
```

# Развертывание кластера PostgreSQL с помошью Terraform

## Скачивание terraform-спецификации

1. На ВМ загружен репозиторий с файлами для практической работы. Скачайте последнюю версию репозитория и сохраните 
путь к репозиторию в переменной окружения `REPO` (установите свой путь, если директория отличается от примера):
```
export REPO=/home/common/yandex-scale-2020-lab-k8s
echo "export REPO=/home/common/yandex-scale-2020-lab-k8s" >> ~/.bashrc
cd $REPO
git pull
```

## Инициализация Terraform и развертывание инфраструктуры

1. Перейдите в каталог с описанием инфраструктуры приложения и инициализируйте Terraform:
```
cd $REPO/terraform/
terraform init
```
2. Разверните инфраструктуру с помощью Terraform:
```
terraform apply -var yc_folder=$YC_FOLDER -var yc_token=$YC_TOKEN -var user=$USER
```

3. Развертывание кластера займет какое-то время. Оставьте открытым терминал с запущенным Terraform 
и перейдите к следующему шагу. Позже мы проверим, что кластер создался.

# Первоначальное развертывание k8s в Облаке
## Создание сервисных аккаунтов
2. Создайте сервисный аккаунт `k8s-cluster-manager` с ролью `editor`.
1. Создайте сервисный аккаунт `k8s-image-puller` с ролью `container-registry.images.puller`;

## Создание кластера
1. Создайте ресурс `Кластер Kubernetes` с именем lab-k8s:
* Укажите `k8s-cluster-manager` в качестве сервисного аккаунта для ресурсов;
* Укажите `k8s-image-puller` в качестве сервисного аккаунта для узлов;
* Дождитесь создания кластера.

## Создание группы узлов
1. Создайте группу узлов с именем `lab-k8s-group`:
* Доступ внутрь ВМ нам не потребуется, поэтому в разделе `Доступ` можно ввести любой текст.

## Проверка состояния кластера
```
yc managed-kubernetes cluster list
export K8S_ID=<ID>
yc managed-kubernetes cluster --id=$K8S_ID get
yc managed-kubernetes cluster --id=$K8S_ID list-node-groups
```

## Добавление учетных данных в конфигурационный файл kubectl
```
yc managed-kubernetes cluster get-credentials --id=$K8S_ID --external
```
Конфигурация будет записана в файл `~/.kube/config`, проверим его содержимое:
```
cat ~/.kube/config
```

# Проверка созданных ресурсов

## Проверка кластера PostgreSQL

1. Вернитесь к терминалу с запущенным Terraform. По завершении работы в секции `Outputs` можно будет увидеть данные о кластере.

1. Сохраните данные о кластере в переменные окружения:
```
export DATABASE_URI=<database_uri>
echo "export DATABASE_URI=$DATABASE_URI" >> ~/.bashrc
export DATABASE_HOSTS=<database_hosts>
echo "export DATABASE_URI=$DATABASE_HOSTS" >> ~/.bashrc
```

2. Проверьте результат работы Terraform:
```
yc managed-postgresql cluster list
yc managed-postgresql cluster get --id=<PG_ID>
```

# Подготовка docker-образов
## Создание Container Registry
1. Создайте registry: `yc container registry create --name lab-registry`;
2. Проверьте его наличие через терминал: `yc container registry list`;
3. Настройте аутентификацию в docker:
```
yc container registry configure-docker
cat ~/.docker/config.json
```
Файл `config.json` содержит вашу конфигурацию после авторизации:
```
{
  "credHelpers": {
    "container-registry.cloud.yandex.net": "yc",
    "cr.cloud.yandex.net": "yc",
    "cr.yandex": "yc"
  }
```

## Сборка и загрузка docker-образов в Container Registry 
1. Переходим в исходники приложения:
```
cd $REPO/app
```

2. Собираем образ приложения: 
```
yc container registry list
export REGISTRY_ID=<ID>
sudo docker build . --tag cr.yandex/$REGISTRY_ID/lab-demo:v1
```

3. Загружаем образ
```
sudo docker push cr.yandex/$REGISTRY_ID/lab-demo:v1
```

## Просмотр списка docker-образов

Получить список Docker-образов в реестре можно командой `yc container image list`
В результате в таблице будет ID образа, дата его создания, месторасположение и т.д.
Получить информацию о конкретном Docker-образе можно командой `yc container image get <IMAGE_ID>`

# Разворачивание приложения и Load Balancer в k8s
1. Посмотрите файлы конфигурации:
```
cd $REPO/k8s-files
cat lab-demo.yaml.tpl
cat load-balancer.yaml
```

2. В файле `lab-demo.yaml.tpl` замените значение переменных:
Переменные `DATABASE_URI` и `DATABASE_HOSTS` получены в результате работы terraform на четвертом шаге.
`REGISTRY_ID` получали ранее с помощью команды `yc container registry list`.
```
envsubst \$REGISTRY_ID,\$DATABASE_URI,\$DATABASE_HOSTS <lab-demo.yaml.tpl > lab-demo.yaml
```

4. Разверните ресурсы:
```
kubectl apply -f lab-demo.yaml
kubectl describe deployment lab-demo
```

```
kubectl apply -f load-balancer.yaml
kubectl describe service lab-demo
```

5. Как только `service load-balancer` полноценно развернется и получит внешний URL (поле `LoadBalancer Ingress`),
проверим работоспособность сервиса в браузере.

# Изменения в архитектуре сервиса для обеспечения отказойстойчивости

После выполнения всех шагов у нас есть:
* MDB PostgreSQL с одним хостом
* Managed Kubernetes с зональным мастером (1 хост) и группой узлов из одного узла
* Одна реплика приложения, запущенная в k8s

# Отказоустойчивый MDB PostgreSQL

В MDB можно увеличить количество хостов в кластере без остановки работы.
1. Зайдите во вкладку `Managed Service for PostgreSQL` в UI, выберите созданный кластер, во вкладке `Хосты` добавьте хост;
2. Выберите зону доступности, отличную от той, которая используется для первого хоста;
3. Убедитесь, что у хостов в кластере достаточно ресурсов, чтобы обрабатывать возвросшую нагрузку при падении одного
или нескольких хостов.

# Отказоустойчивый Managed Kubernetes

Необходимо сделать региональный мастера, состоящий из трех хостов. Тип мастера уже созданного кластера
поменять нельзя, поэтому надо создать новый кластер.
1. Зайдите во вкладку `Managed Service for Kubernetes` и создайте новый кластер;
2. Выберите тип мастера `Региональный`;
3. Создайте группу узлов с 3 узлами;
4. Убедитесь, что у хостов в группе узлов достаточно ресурсов, чтобы обрабатывать возвросшую нагрузку при падении одного
или нескольких хостов.

# Отказоустойчивое приложение

При деплое приложения в k8s мы указывали одну реплику, надо увеличить количество реплик до трех. Также уже создан
балансировщик нагрузки, который будет распределять нагрузку по всем трем репликам.

# Удаление ресурсов

## Удаление ресурсов из кластера k8s
```
kubectl delete -f load-balancer.yaml
kubectl delete -f lab-demo.yaml
```

## Удаление кластера k8s
```
yc managed-kubernetes cluster list
yc managed-kubernetes cluster delete lab-k8s
yc iam service-account delete k8s-cluster-manager
yc iam service-account delete k8s-image-puller
yc managed-kubernetes cluster list
```

## Удаление базы данных в Terraform
```
cd $REPO/terraform
terraform destroy -var yc_folder=$YC_FOLDER -var yc_token=$YC_TOKEN -var user=$USER
yc managed-postgresql cluster list
```

## Удаление реестра и docker-образов
В консоли зайдите во вкладку `Container Registry`, выберите реестр `lab-registry` и удалите в нем все образы.
После этого вернитесь к списку реестров, нажмите на раскрывающееся меню для `lab-registry` и удалите реестр.

## Удаление ВМ с настроенным окружением 
В консоли зайдите во вкладку `Compute`, нажмите на раскрывающееся меню для ВМ `lab-vm` и удалите эту ВМ.