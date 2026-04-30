# CVE-2026-31431 Fix for Yandex Managed Kubernetes

Автоматическое исправление уязвимости CVE-2026-31431 в Linux kernel на всех worker нодах Yandex Managed Kubernetes кластера.

## Описание уязвимости

**Идентификатор CVE (CVE ID):** CVE-2026-31431

**Ссылка на CVE:** https://nvd.nist.gov/vuln/detail/CVE-2026-31431

**Исходный отчет:**
- Сайт уязвимости: https://copy.fail/
- Технический разбор Xint Code (Theori): https://xint.io/blog/copy-fail-linux-distributions
- PoC и issue tracker: https://github.com/theori-io/copy-fail-CVE-2026-31431

**Краткое описание:**

Copy Fail (CVE-2026-31431) — это логическая уязвимость в подсистеме криптографического API ядра Linux, позволяющая обычному пользователю системы получить права суперпользователя (root). PoC работает на всех основных дистрибутивах Linux, выпущенных с 2017 года и до момента выхода патча, без модификаций под конкретное ядро или дистрибутив.

**Атака:**
- не требует удалённого доступа — только непривилегированный локальный аккаунт
- не требует ни race window, ни kernel-specific смещений
- использует крипто-API ядра (AF_ALG), который включён в дефолтных конфигурациях практически всех мейнстрим-дистрибутивов
- может быть использована как примитив побега из контейнера на хост, поскольку page cache общий для всего узла

**Затронутые технологии:**
- Ядро Linux, подсистема crypto/ (модуль algif_aead)
- интерфейс пользовательского пространства AF_ALG
- системный вызов splice() в связке с сокетами AF_ALG

Уязвимость напрямую не затрагивает следующие пути использования крипто-API: dm-crypt / LUKS, kTLS, IPsec/XFRM, in-kernel TLS, OpenSSL/GnuTLS/NSS в дефолтных сборках, SSH, kernel keyring crypto — все они работают с in-kernel crypto API напрямую и не проходят через AF_ALG.

**Вектор атаки и уровень опасности согласно CVSS v.3.1:**

Базовая оценка: **7.8 HIGH**

Vector: `CVSS:3.1/AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H`

## Что делает этот фикс

DaemonSet автоматически на каждой worker ноде кластера:

1. **Проверяет уязвимость** - тестирует доступность AF_ALG AEAD интерфейса
2. **Блокирует модуль** - создает конфигурацию `/etc/modprobe.d/disable-algif.conf`
3. **Выгружает модуль** - выполняет `rmmod algif_aead` если модуль загружен
4. **Верифицирует фикс** - проверяет что уязвимость устранена
5. **Мониторит** - каждый час проверяет наличие конфигурации и восстанавливает при необходимости

## Быстрый старт

### 1. Скачать DaemonSet

```bash
wget https://raw.githubusercontent.com/yandex-cloud-examples/yc-mk8s-copy-fail-mitigation/main/copy-fail-mitigation-daemonset.yaml
```

Или клонировать репозиторий:

```bash
git clone https://github.com/yandex-cloud-examples/yc-mk8s-copy-fail-mitigation.git
cd yc-mk8s-copy-fail-mitigation
```

### 2. Применить фикс

```bash
kubectl apply -f copy-fail-mitigation-daemonset.yaml
```

### 3. Проверить статус применения

```bash
# Проверить статус DaemonSet
kubectl get daemonset -n kube-system cve-2026-31431-fix

# Посмотреть на скольких нодах применен фикс
kubectl get pods -n kube-system -l app=cve-2026-31431-fix -o wide
```

### 4. Просмотреть логи применения фикса

```bash
# Логи initContainer (применение фикса)
kubectl logs -n kube-system -l app=cve-2026-31431-fix -c apply-fix

# Логи основного контейнера (мониторинг)
kubectl logs -n kube-system -l app=cve-2026-31431-fix -c monitor
```

## Пример успешного применения

```
=========================================
CVE-2026-31431 Fix for Yandex Managed K8s
Node: demo-ru-central1-a-1
Date: Thu Apr 30 14:00:00 UTC 2026
=========================================

Step 1: Checking vulnerability before fix...
❌ System is VULNERABLE - AF_ALG AEAD interface is accessible

Step 2: Creating modprobe configuration...
✓ Created /etc/modprobe.d/disable-algif.conf

Step 3: Unloading algif_aead module if loaded...
✓ Module unloaded successfully

Step 4: Verifying the fix...
✓ Configuration file exists:
install algif_aead /bin/false

Step 5: Testing if vulnerability is fixed...
✓ AF_ALG AEAD interface is properly blocked

=========================================
✓ CVE-2026-31431 fix applied successfully
=========================================
```

## Проверка уязвимости вручную

Вы можете проверить наличие уязвимости на ноде вручную. Подключитесь к ноде по SSH и выполните:

```bash
# Проверить уязвимость
python3 -c 'import socket; s = socket.socket(socket.AF_ALG, socket.SOCK_SEQPACKET, 0); s.bind(("aead","authencesn(hmac(sha256),cbc(aes))")); print("AF_ALG AEAD available - VULNERABLE")'

# Если выводит "AF_ALG AEAD available - VULNERABLE" - система уязвима
# Если выдает ошибку - система защищена
```

Проверить конфигурацию:

```bash
# Проверить наличие конфигурации блокировки
cat /etc/modprobe.d/disable-algif.conf

# Ожидаемый вывод:
# install algif_aead /bin/false
```

## Удаление фикса

Если необходимо удалить DaemonSet:

```bash
kubectl delete -f copy-fail-mitigation-daemonset.yaml
```

**Важно:** Удаление DaemonSet **не удалит** конфигурационные файлы с нод. Файл `/etc/modprobe.d/disable-algif.conf` останется на месте и будет продолжать защищать систему.

Для полного удаления фикса с нод нужно подключиться к каждой ноде по SSH и вручную удалить файл:

```bash
rm /etc/modprobe.d/disable-algif.conf
```

## Технические детали

**Используемые разрешения:**
- `hostPID: true` - для доступа к процессам хоста через nsenter
- `privileged: true` - для записи в `/etc` и выгрузки модулей ядра
- Volume mount `/` - для доступа к файловой системе хоста

**Образ:** `ubuntu:22.04`

**Ресурсы:**
- Init container: 10m CPU / 64Mi RAM (requests), 200m CPU / 128Mi RAM (limits)
- Monitor container: 5m CPU / 32Mi RAM (requests), 50m CPU / 64Mi RAM (limits)

**Namespace:** `kube-system`

## Совместимость

- ✓ Yandex Managed Kubernetes
- ✓ Ubuntu 20.04
- ✓ Ubuntu 22.04
- ✓ Kubernetes 1.20+

## Лицензия

Apache License 2.0

См. [LICENSE](LICENSE) для подробностей.

## Поддержка

При возникновении проблем создайте issue в репозитории.
