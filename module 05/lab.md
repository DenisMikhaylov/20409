
# Лабораторная работа: Развёртывание двухузлового отказоустойчивого кластера Hyper‑V на Windows Server 2025

## Цель работы
Научиться создавать две виртуальные машины на основе разностного диска, настраивать их как узлы кластера, включать вложенную виртуализацию, устанавливать роль Hyper‑V и строить отказоустойчивый кластер в домене `adatum.com`.

## Исходные данные
- **Физический хост Hyper‑V** с ОС Windows Server 2025.
- **Базовый диск** `base2025.vhdx`
- **Внешний виртуальный коммутатор** с именем `External Network` 
- **Контроллер домена** `adatum.com` доступен в сети 
- Учётная запись с правами администратора домена.

---

## Ход работы

### Этап 1. Создание разностных дисков для виртуальных машин
1. Откройте **PowerShell** от имени администратора на физическом хосте.
2. Перейдите в папку, где будете хранить виртуальные машины.
3. Создайте разностные диски для `Lon-HOST11` и `Lon-HOST22`:

```powershell
New-VHD -Path "C:\VMs\Lon-HOST11\Lon-HOST11.vhdx" -ParentPath "C:\vm\base2025.vhdx" -Differencing
New-VHD -Path "C:\VMs\Lon-HOST22\Lon-HOST22.vhdx" -ParentPath "C:\vm\base2025.vhdx" -Differencing
```

> **Проверка:**  
> В папках `Lon-HOST11` и `Lon-HOST22` должны появиться файлы `.vhdx` размером около 4–10 МБ – это разностные диски, ссылающиеся на базовый образ.

---

### Этап 2. Создание виртуальных машин
1. Создайте ВМ `Lon-HOST11`:

```powershell
New-VM -Name "Lon-HOST11" -MemoryStartupBytes 8GB -BootDevice VHD -VHDPath "C:\VMs\Lon-HOST11\Lon-HOST11.vhdx" -Path "C:\VMs\Lon-HOST11" -Generation 2 -SwitchName "External Network"
```

2. Настройте количество процессоров :

```powershell
Set-VMProcessor -VMName "Lon-HOST11" -Count 4
```

3. Аналогично создайте `Lon-HOST22` :

```powershell
New-VM -Name "Lon-HOST22" -MemoryStartupBytes 8GB -BootDevice VHD -VHDPath "C:\VMs\Lon-HOST22\Lon-HOST22.vhdx" -Path "C:\VMs\Lon-HOST22" -Generation 2 -SwitchName "External Network"
Set-VMProcessor -VMName "Lon-HOST22" -Count 4
```

> **Проверка:**  
> В диспетчере Hyper‑V появятся две новые виртуальные машины. Запустите их и убедитесь, что загружается Windows Server 2025.

---

### Этап 3. Базовая настройка операционной системы на каждой ВМ
#### 3.1. Установка имени компьютера и статического IP‑адреса
**На `Lon-HOST11`**:
- Войдите под локальной учётной записью администратора (пароль от базового образа).
- Откройте **Свойства системы** → **Имя компьютера** → **Изменить** → укажите имя `Lon-HOST11`.
- Назначьте статический IP‑адрес, например `172.16.0.40/24`, шлюз `172.16.0.1`, DNS – 172.16.0.10.

**На `Lon-HOST22`**:
- Повторите те же действия, но имя `Lon-HOST22`, IP `172.16.0.41/24`.

#### 3.2. Присоединение к домену `adatum.com`
- На каждом сервере откройте **Свойства системы** → **Изменить** → выберите **Домен** и введите `adatum.com`.
- Укажите учётные данные администратора домена.
- Перезагрузите оба сервера по запросу.

> **Проверка:**  
> После перезагрузки войдите под учётной записью `adatum\Administrator`. Выполните в PowerShell:
> ```powershell
> whoami
> ```
> Должно отобразиться `adatum\Administrator`.

---

### Этап 4. Включение вложенной виртуализации
**Внимание:** ВМ должны быть выключены!

1. Остановите обе ВМ:
```powershell
Stop-VM -Name "Lon-HOST11", "Lon-HOST22"
```

2. Включите расширения виртуализации для процессоров:
```powershell
Set-VMProcessor -VMName "Lon-HOST11" -ExposeVirtualizationExtensions $true
Set-VMProcessor -VMName "Lon-HOST22" -ExposeVirtualizationExtensions $true
```

3. Разрешите подмену MAC‑адресов для сетевых адаптеров (необходимо для работы вложенных коммутаторов):
```powershell
Get-VMNetworkAdapter -VMName "Lon-HOST11" | Set-VMNetworkAdapter -MacAddressSpoofing On
Get-VMNetworkAdapter -VMName "Lon-HOST22" | Set-VMNetworkAdapter -MacAddressSpoofing On
```

> **Проверка:**  
> Запустите ВМ и внутри них выполните `systeminfo | find "Hyper-V"` – должны увидеть, что требования Hyper‑V выполнены.

---

### Этап 5. Установка ролей Hyper‑V и отказоустойчивого кластера
**На каждом узле** (под учётной записью администратора домена) выполните следующие шаги:

1. Откройте PowerShell от имени администратора.
2. Установите необходимые компоненты с автоматической перезагрузкой:

```powershell
Install-WindowsFeature -Name Hyper-V, Failover-Clustering, Hyper-V-PowerShell, RSAT-Clustering-PowerShell, NetworkATC -IncludeManagementTools -Restart
```

> **Пояснение:**  
> - `Hyper-V` – сама роль виртуализации.  
> - `Failover-Clustering` – компонент кластеризации.  
> - `Hyper-V-PowerShell` – модуль для управления Hyper‑V через PowerShell.  
> - `RSAT-Clustering-PowerShell` – средства удалённого администрирования кластера.  
> - `NetworkATC` – автоматическая настройка сетей (новая возможность в WS2025).  
> Ключ `-Restart` перезагрузит сервер по окончании установки.

3. После перезагрузки проверьте успешность установки:
```powershell
Get-WindowsFeature -Name Hyper-V, Failover-Clustering | Select-Object Name, InstallState
```
В столбце `InstallState` должно быть `Installed`.

---

### Этап 6. Проверка готовности и создание кластера
Выполняйте шаги на любом из узлов (например, на `Lon-HOST11`).

1. Запустите проверку совместимости оборудования и настроек:
```powershell
Test-Cluster -Node "Lon-HOST11", "Lon-HOST22" -ReportName "C:\ClusterValidationReport.html"
```

2. Откройте созданный отчёт `C:\ClusterValidationReport.html` и убедитесь, что все тесты прошли с зелёным статусом (ошибок быть не должно). Если есть ошибки – исправьте их (обычно это проблемы с сетью, хранилищем или службой времени).

3. Создайте кластер с именем, например, `HyperVCluster`, и назначьте ему статический IP‑адрес в вашей сети :
```powershell
New-Cluster -Name HyperVCluster -Node "Lon-HOST11", "Lon-HOST22" -StaticAddress 172.16.0.60
```

> **Проверка:**  
> Откройте оснастку **Диспетчер отказоустойчивости кластеров** (Failover Cluster Manager) – кластер `HyperVCluster` должен отображаться с двумя узлами в активном состоянии.

---

### Этап 7. Настройка кворума (Quorum)
Для двухузлового кластера обязательно нужен **наблюдатель** (witness). Выберите один из вариантов.

**Вариант А – облачный наблюдатель (Cloud Witness)** – если есть подписка Azure.  
Выполните:
```powershell
Set-ClusterQuorum -CloudWitness -AccountName "storageaccountname" -AccessKey "ваш_ключ"
```

**Вариант Б – файловый наблюдатель (File Share Witness)** – если есть общая папка.  
Создайте на контроллере домена папку `C:\ClusterWitness`, расшарьте её с полным доступом для компьютеров кластера. Затем выполните:
```powershell
Set-ClusterQuorum -FileShareWitness \\adatum.com\ClusterWitness
```

**Вариант В – дисковый наблюдатель** – если у вас есть общий диск (iSCSI или SAN).  
Создайте диск для наблюдателя и выполните:
```powershell
Set-ClusterQuorum -DiskQuorum <DiskResource>
```

> **Проверка:**  
> В диспетчере кластеров перейдите в раздел **Кворум** – должно быть зелёное состояние и указан выбранный тип наблюдателя.

---

### Этап 8. Финальная проверка и тестирование
1. Убедитесь, что оба узла кластера работают корректно:
```powershell
Get-ClusterNode -Cluster HyperVCluster
```

2. Попробуйте создать высокодоступную виртуальную машину на кластере (можно прямо в оснастке Failover Cluster Manager).

3. Протестируйте отказоустойчивость – переместите роль с одного узла на другой вручную или выключите один узел и проверьте, что ВМ автоматически перезапускается на втором узле.

