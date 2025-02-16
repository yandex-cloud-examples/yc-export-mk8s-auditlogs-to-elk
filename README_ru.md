## Yandex.Cloud: анализ логов безопасности K8s в ELK, включая аудит-логи, policy engine и Falco 

![image](https://user-images.githubusercontent.com/85429798/137449451-eaa3a4ec-5a79-4fc5-8e7e-bd222c78b714.png)

![Dashboard](https://user-images.githubusercontent.com/85429798/130331405-26a909ae-0171-47b2-93a2-c656632d262c.png)

<img width="1403" alt="1" src="https://user-images.githubusercontent.com/85429798/133788731-3c410508-3539-4ba0-b873-85ae55d58b87.png">

![2](https://user-images.githubusercontent.com/85429798/133788762-75152c1a-ad93-4291-999d-7fc0739d2438.png)

# Версия

**Версия 2.0**
- Чейнджлог:
    - Изменен метод развертывания. Виртуальные машины больше не выполняют роль рабочих движков, вместо них используется Kubernetes. Благодарим Hilbert Team за содействие в реализации.
    <a href="https://kubernetes.io/">
    <img src="https://storage.yandexcloud.net/9863c845-4d2b-4a09-a7dc-84118e8b892a-ht-logo/HT%20Logo.png"
         alt="Kubernetes logo" title="Kubernetes" height="115" width="115" />
</a></br>
- Docker-образы:
    - `cr.yandex/sol/k8s-events-siem-worker:2.0.0`.

**Версия 2.0**
- Чейнджлог:
    - Добавлена поддержка автоустановки Kyverno с политиками в режиме audit. 
- Docker-образы:
    - `cr.yandex/sol/k8s-events-siem-worker:1.1.0`.

# Содержание

- [Описание](#description)
- [Ссылка на решение для сбора, мониторинга и анализа аудит-логов в Yandex Managed Service for Elasticsearch (ELK)](#link-to-solution-"Collecting-monitoring-and-analyzing-audit-logs-in-Yandex-Managed-Service-for-Elasticsearch-(ELK)")
- [Общая схема](#generic-diagram)
- [Описание Terraform](#terraform-description)
- [Процесс обновления контента](#content-update-process)
- [Опциональные действия, выполняемые вручную](#optional-manual-actions)


## Описание

Возможности решения, предоставляемые по умолчанию:
☑️ Сбор [аудит-логов K8s](https://kubernetes.io/docs/tasks/debug-application-cluster/audit/) в [Managed ELK SIEM](https://yandex.cloud/en/docs/managed-elasticsearch/).
- ☑️ Установка [Falco](https://falco.org/) и сбор его [алертов](https://falco.org/docs/alerts/) в [Managed ELK SIEM](https://yandex.cloud/en/docs/managed-elasticsearch/).
- ☑️ Установка [Kyverno](https://kyverno.io/) с политиками [Pod Security Standards (Restricted)](https://kyverno.io/policies/?policytypes=Pod%2520Security%2520Standards%2520%28Restricted%29) в режиме audit и сбор его [алертов (Policy Reports)](https://kyverno.io/docs/policy-reports/) с помощью [Policy Reporter](https://github.com/kyverno/policy-reporter).
- ☑️ Импорт объектов Security Content, включая дашборды, правила детектирования и пр. (cм. в разделе _Security Content_) в [SIEM-систему Managed ELK](https://yandex.cloud/en/docs/managed-elasticsearch/) для анализа и реагирования на события информационной безопасности. 
- ☑️ Импорт объектов Security Content для [OPA Gatekeeper](https://open-policy-agent.github.io/gatekeeper/website/docs/) (в режиме enforce). Сам OPA Gatekeeper можно пи необходимости установить вручную.
- ☑️ Создание индексов в двух репликах и установка базовой политики rollover (создание новых индексов каждые 30 дней или по достижении 50 ГБ) для обеспечения высокой доступности данных и настройки снимков данных в S3 (см. [рекомендации](../export-auditlogs-to-ELK_main/CONFIGURE-HA.md)). 

## Ссылка на решение для сбора, мониторинга и анализа аудит-логов в Yandex Managed Service for Elasticsearch (ELK)

[Решение для сбора, мониторинга и анализа аудит-логов в Yandex Managed Service for Elasticsearch (ELK)](https://github.com/yandex-cloud/yc-solution-library-for-security/tree/master/auditlogs/export-auditlogs-to-ELK_main) содержит информацию об установке Yandex Managed Service for Elasticsearch (ELK) и сбору в него логов Audit Trails.


## Общая схема 

![image](https://user-images.githubusercontent.com/85429798/164211865-5f95498a-3778-47a9-bb82-cb43110836c4.png)

## Описание импортируемых объектов ELK (Security Content)
Подробное описание объектов представлено [здесь](https://github.com/yandex-cloud/yc-solution-library-for-security/blob/master/auditlogs/export-auditlogs-to-ELK_main/papers/Описание%20объектов.pdf).

## Описание Terraform 

Данное решение основано на модуле Terraform, который:
- Принимает следующие входные параметры: 
    - `folder_id` — идентификатор каталога, в котором находится кластер.
    - `cloud_id` — идентификатор облака, в котором находится кластер.
    - `cluster_name` — имя кластера Kubernetes.
    - `elastic_server` — FQDN инсталляции ELK.
    - `elastic_pw` и `elastic_user` — учетные данные пользователя ELK для импорта событий.
    - `service_account_id` — идентификатор сервисного аккаунта с правом записи в бакет и ролью `ymq.admin`.
    - `log_bucket_name` — имя бакета для хранения логов, создаваемого модулем.
    - `auditlog_enabled` — включает или отключает отправку аудит-логов K8s в ELK. Принимает значение `true` или `false`.
    - `falco_enabled` — включает или отключает отправку алертов Falco в ELK. Принимает значение `true` или `false`.
    - `kyverno_enabled` — включает или отключает отправку алертов Kyverno в ELK. Принимает значение `true` или `false`.
- Поддерживает: 
    - Создание статического ключа для сервисного аккаунта.
    - Создание функции и триггера для записи логов кластера в S3.
    - Установку Falco и преднастроенного `falcosidekick` для отправки логов в S3.
    - Установку Kyverno и преднастроенного [Policy Reporter](https://github.com/kyverno/policy-reporter) для отправки логов в S3.
    - Создание очередей YMQ с именами файлов логов в S3.
    - Создание функций для передачи имен файлов из S3 в YMQ.
    - Создание триггеров для взаимодействия очередей и функций.
    - Развертывание в Kubernetes с контейнерами воркеров для импорта событий из S3 в ELK.

#### Предварительные условия

- :white_check_mark: Кластер Managed K8s.
- :white_check_mark: Managed ELK.
- :white_check_mark: Сервисный аккаунт с правом записи в бакет и ролью `ymq.admin`.


#### Пример вызова модулей

Пример вызова модулей см. в [/examples/README.md](https://github.com/yandex-cloud/yc-solution-library-for-security/blob/master/auditlogs/export-auditlogs-to-ELK_k8s/examples/README.md).


## Процесс обновления контента

Рекомендуем подписаться на данный репозиторий для получения уведомлений об обновлениях.

В части обновления контента необходимо убедиться, что вы используете последнюю доступную версию образа:
`cr.yandex/sol/k8s-events-siem-worker:latest`

Обновление контейнера можно выполнить следующим образом:
Пересоздайте развертывания в K8s с помощью Terraform, изменив переменную окружения `worker_docker_image` в `tfvars` и выполнив команду `terraform apply`.

## Опциональные действия, выполняемые вручную

#### Установка OPA Gatekeeper с помощью Helm

Если предпочтительнее использовать OPA Gatekeeper вместо Kyverno, то при вызове модуля установите параметр `kyverno_enabled` в значение `false` и выполните установку вручную:
- Установите OPA Gatekeeper [с помощью Helm](https://open-policy-agent.github.io/gatekeeper/website/docs/install/#deploying-via-helm).
- Из [gatekeeper-library](https://github.com/open-policy-agent/gatekeeper-library/tree/master/library/pod-security-policy) выберите и установите требуемый шаблон ограничения и само ограничение.
- Ознакомьтесь с примером установки [здесь](https://github.com/open-policy-agent/gatekeeper-library#usage).

## Рекомендации по настройке политики хранения (retention), ротации индексов (rollover) и создания снимков (snapshots)

Ознакомиться с рекомендациями по настройке политики хранения, ротации индексов и создания снимков можно [здесь](../export-auditlogs-to-ELK_main/CONFIGURE-HA.md).
