# Copy Fail / Dirty Frag Mitigation for Yandex Managed Kubernetes

Автоматическое применение митигации для уязвимостей `CVE-2026-31431`, `CVE-2026-43284` и `CVE-2026-43500` в Linux kernel на всех worker-нодах Yandex Managed Kubernetes кластера.

## Описание уязвимости

**Идентификатор CVE (CVE ID):** `CVE-2026-43284`, `CVE-2026-43500`

**Ссылка на CVE:** https://nvd.nist.gov/vuln/detail/CVE-2026-43284

**Исходный отчет:**
- Dirty Frag (PoC и write-up): https://github.com/V4bel/dirtyfrag
- Copy Fail 2: Electric Boogaloo (PoC xfrm-ESP): https://github.com/0xdeadbeefnetwork/Copy_Fail2-Electric_Boogaloo
- Рассылка oss-security: https://www.openwall.com/lists/oss-security/2026/05/07/8

**Краткое описание:**

Dirty Frag - это класс логических уязвимостей в ядре Linux, позволяющий непривилегированному локальному пользователю получить права суперпользователя (`root`). Эксплуатация объединяет два независимых page-cache write-примитива в подсистемах `xfrm-ESP` и `RxRPC`, каждый из которых самодостаточен для повышения привилегий.

`Copy Fail 2: Electric Boogaloo` - независимый PoC, эксплуатирующий `xfrm-ESP`-примитив (`CVE-2026-43284`). По классу уязвимости он аналогичен исходному `Copy Fail` (`CVE-2026-31431`), поэтому этот DaemonSet сохраняет митигацию и для исходного `AF_ALG`-сценария, и для новых вариантов `Dirty Frag`.

**Атака:**
- не требует удалённого доступа - только непривилегированный локальный аккаунт
- является детерминистическим логическим багом без race condition - успешна с первой попытки
- не приводит к `kernel panic` при неудачной эксплуатации
- может быть использована как примитив побега из контейнера на хост, поскольку `page cache` общий для всего узла

Корневая причина обоих вариантов одна: при использовании `splice()` / `MSG_SPLICE_PAGES` ядро помещает страницы `page cache` напрямую во фрагменты сокетных буферов (`skb`). Подсистемы `xfrm-ESP` и `RxRPC` выполняют дешифрование in-place над такими фрагментами, не проверяя, являются ли они приватными. В результате атакующий получает контролируемую запись в `page cache` любого читаемого файла.

**Затронутые технологии:**
- Ядро Linux, подсистема `net/ipv4/esp4.c` / `net/ipv6/esp6.c` (`xfrm-ESP`)
- Ядро Linux, подсистема `net/rxrpc/rxkad.c` (`RxRPC` / `RxKAD`)
- Системные вызовы `splice()` / `vmsplice()` в связке с UDP-сокетами (`ESP-in-UDP`) и `AF_RXRPC`
- Отдельно сохраняется митигация исходного `Copy Fail` (`CVE-2026-31431`) через блокировку `AF_ALG` (`algif_aead`)

Уязвимость напрямую не затрагивает `AF_ALG` (`algif_aead`) как часть `Dirty Frag` - это отдельная уязвимость `Copy Fail` (`CVE-2026-31431`). Также напрямую не затрагиваются `dm-crypt` / `LUKS`, `kTLS`, `in-kernel TLS` и `IPsec` в режиме tunnel без UDP-инкапсуляции.

**Вектор атаки и уровень опасности согласно CVSS v.3.1:**

Базовая оценка: на момент публикации не присвоена.

По характеру уязвимость аналогична `Copy Fail` (`CVE-2026-31431`, `7.8 HIGH`, `CVSS:3.1/AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H`) - это локальное повышение привилегий без race condition.

## Что делает этот фикс

DaemonSet автоматически на каждой worker ноде кластера:

1. **Проверяет доступность `AF_ALG`** - выполняет быстрый тест для исходного сценария `Copy Fail`
2. **Блокирует уязвимые модули** - создает `/etc/modprobe.d/blacklist-lpe.conf` с правилами для `algif_aead`, `esp4`, `esp6` и `rxrpc`
3. **Выгружает модули** - выполняет `rmmod` для `algif_aead`, `esp4`, `esp6` и `rxrpc`, если они загружены
4. **Сбрасывает `page cache` и верифицирует конфигурацию** - очищает кэши и проверяет наличие конфигурационного файла
5. **Мониторит состояние** - каждый час проверяет наличие конфигурации и повторно выгружает модули при необходимости

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
Copy Fail / Dirty Frag mitigation for Yandex Managed K8s
Node: demo-ru-central1-a-1
Date: Thu May 08 14:00:00 UTC 2026
=========================================

Step 1: Checking vulnerability before fix...
❌ System is VULNERABLE - AF_ALG AEAD interface is accessible

Step 2: Creating modprobe configuration...
✓ Created /etc/modprobe.d/blacklist-lpe.conf

Step 3: Unloading vulnerable modules...
  ✓ algif_aead unloaded
  ✓ esp4 not loaded
  ✓ esp6 not loaded
  ✓ rxrpc not loaded

Step 3.5: Dropping system caches...
✓ System caches cleared

Step 4: Verifying the fix...
✓ Configuration file exists:
install algif_aead /bin/false
install esp4 /bin/false
install esp6 /bin/false
install rxrpc /bin/false

Step 5: Testing if vulnerability is fixed...
✓ AF_ALG AEAD interface is properly blocked

=========================================
✓ Mitigation applied successfully
=========================================
```

## Проверка уязвимости вручную

Вы можете проверить наличие уязвимости на ноде вручную. Подключитесь к ноде по SSH и выполните:

```bash
# Проверить доступность исходного сценария Copy Fail через AF_ALG
python3 -c 'import socket; s = socket.socket(socket.AF_ALG, socket.SOCK_SEQPACKET, 0); s.bind(("aead","authencesn(hmac(sha256),cbc(aes))")); print("AF_ALG AEAD available - VULNERABLE")'

# Если выводит "AF_ALG AEAD available - VULNERABLE" - система уязвима
# Если выдает ошибку - система защищена
```

Проверить конфигурацию:

```bash
# Проверить наличие конфигурации блокировки
cat /etc/modprobe.d/blacklist-lpe.conf

# Ожидаемый вывод:
# install algif_aead /bin/false
# install esp4 /bin/false
# install esp6 /bin/false
# install rxrpc /bin/false
```

Проверить, что уязвимые модули не загружены:

```bash
lsmod | egrep 'algif_aead|esp4|esp6|rxrpc'
```

## Удаление фикса

Если необходимо удалить DaemonSet:

```bash
kubectl delete -f copy-fail-mitigation-daemonset.yaml
```

**Важно:** Удаление DaemonSet **не удалит** конфигурационные файлы с нод. Файл `/etc/modprobe.d/blacklist-lpe.conf` останется на месте и будет продолжать защищать систему.

Для полного удаления фикса с нод нужно подключиться к каждой ноде по SSH и вручную удалить файл:

```bash
rm /etc/modprobe.d/blacklist-lpe.conf
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
