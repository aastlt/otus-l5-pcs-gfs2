```markdown
# GFS2 Cluster Storage

## Описание проекта

Данный проект автоматизирует развертывание отказоустойчивого общего хранилища на базе GFS2 (Global File System 2) в среде VirtualBox. Проект использует iSCSI в качестве транспорта для общего блочного устройства и стек Pacemaker/Corosync/DLM для координации распределенных блокировок.

## Цель

- Развернуть инфраструктуру из 4 ВМ (1 Target, 3 Nodes) через Vagrant.
- Настроить iSCSI-таргет и подключить инициаторы.
- Развернуть кластерный LVM (lvmlockd) и файловую систему GFS2.
- Обеспечить одновременный доступ на чтение и запись с проверкой целостности данных.

## Архитектура

- **iscsi-target (192.168.56.10)**: ОС Debian 12, предоставляет диск размером 5 ГБ через LIO (targetcli).
- **gfs2-node-01..03 (192.168.56.11-13)**: Узлы кластера.
  - Стек: Pacemaker + Corosync (транспорт knet).
  - Блокировки: DLM (Distributed Lock Manager).
  - LVM: Clustered LVM (locking_type = 3).

## Структура проекта

```bash
.
├── Vagrantfile          # Описание инфраструктуры (2 GB RAM, 2 CPU на узел)
├── inventory            # Группировка хостов и переменные
├── site.yml             # Основной Playbook
├── ansible.cfg          # Настройки Ansible (форки, пути)
└── roles
    ├── iscsi-target     # Настройка LIO iSCSI Target
    └── gfs2-cluster     # Стек GFS2, DLM и Pacemaker
```

## Технические особенности

В ходе реализации на Debian 12 были внедрены следующие критические решения:

- **DLM без STONITH**: В лабораторной среде использован параметр `allow_stonith_disabled=true` (заменивший устаревший `enable_fencing`), что позволяет DLM работать без устройств отсечения.
- **ConfigFS**: Принудительное монтирование `/sys/kernel/config`, необходимое для инициализации DLM в ядрах 6.x.
- **Global LVM Activation**: Использование `vgchange -ay` на всех узлах после создания тома на мастере для "проявления" устройства в `/dev/mapper/`.
- **Идемпотентность**: Все задачи Ansible проверяют текущее состояние (`blkid`, `pcs status`, `constraint list`) перед выполнением, что исключает ошибки при повторном запуске.
- **APT Isolation**: Использование `policy-rc.d` для предотвращения неконтролируемого запуска сервисов во время установки пакетов.

## Запуск

### Развертывание ВМ:

```bash
vagrant up
```

Ansible запустится автоматически по завершении поднятия `gfs2-node-03`.

### Ручная проверка (если требуется):

```bash
ansible-playbook -i inventory site.yml
```

## Проверка работоспособности

После завершения работы Ansible, выполните следующие команды:

1. **Статус кластера**:

```bash
vagrant ssh gfs2-node-01 -c "sudo pcs status"
```

Ожидаемый результат: `dlm-clone` и `shared_fs-clone` должны быть в статусе `Started` на всех трёх узлах.

2. **Проверка монтирования**:

```bash
vagrant ssh gfs2-node-02 -c "df -h /mnt/shared"
```

3. **Тест совместного доступа**:

Скрипт проверяет видимость данных между узлами:

```bash
# Запись на первой ноде
vagrant ssh gfs2-node-01 -c "echo 'Hello from Node 1' | sudo tee /mnt/shared/test.txt"

# Чтение на третьей ноде
vagrant ssh gfs2-node-03 -c "cat /mnt/shared/test.txt"
```

## Удаление стенда

```bash
vagrant destroy -f
```

## Лицензия

MIT License

## Автор

[aastlt](https://github.com/aastlt)
```
