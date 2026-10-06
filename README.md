<h1 align="center">simplewall</h1>

-------

<p align="center">
	<img src="/images/simplewall.png?cache" />
</p>
# simplewall: сборка из исходников и аудит безопасности

Сборка фаервола [simplewall](https://github.com/henrypp/simplewall) (автор [henrypp](https://github.com/henrypp)) из открытых исходников с проверкой кода на безопасность.

> Это неофициальная сборка. Автор simplewall к ней отношения не имеет. Официальные релизы, подписанные автором, публикуются на странице [Releases](https://github.com/henrypp/simplewall/releases).

## Коротко

| | |
|---|---|
| Что собрано | simplewall, коммит [`d6fa5dfa`](https://github.com/henrypp/simplewall/commit/d6fa5dfa) от 08.02.2025 (в окне «О программе» версия 3.8.5) |
| Библиотека | [routine](https://github.com/henrypp/routine), коммит [`ff8f811`](https://github.com/henrypp/routine/commit/ff8f811) от 07.01.2025 |
| Платформа | Windows 10/11, x64 |
| Изменения в коде | нет, только флаг сборки `PlatformToolset=v145` |
| Вредоносный код | не найден |
| Найденные слабости | небезопасное автообновление (см. ниже), **отключите проверку обновлений** |

## Почему не последняя версия

simplewall состоит из двух репозиториев: самой программы и вспомогательной библиотеки `routine`. Автор продолжает выкладывать код simplewall, но обновления `routine` не публикует: последнее изменение кода в ней датировано 07.01.2025.

Поэтому:

- текущий `master` и все релизы начиная с **3.8.6** не собираются. Получается 665 ошибок компиляции, в публичной `routine` не хватает 33 функций;
- релиз **3.8.5** тоже не собирается ни с одной опубликованной версией `routine`;
- **`d6fa5dfa`** — последний коммит simplewall перед несовместимым изменением API (`9641e5b4`, 11.02.2025). Это самый свежий код, который можно собрать только из открытых исходников. Он новее релиза 3.8.5.

Недостающие функции `routine` намеренно не дописывались: иначе это был бы уже не код автора.

## Аудит безопасности

Проверялись simplewall (`master` и `d6fa5dfa`) и `routine` (`ff8f811`), около 28 тыс. строк на C.

### Что проверено, проблем нет

- **Сеть.** Единственные сетевые запросы — проверка и загрузка обновлений с `github.com` / `raw.githubusercontent.com`. Телеметрии, аналитики и скрытых адресов нет. Все остальные URL в коде — ссылки в комментариях, на сайт автора или на страницы доната.
- **Запуск процессов.** Только по действию пользователя: открыть ссылку, показать файл в проводнике, открыть regedit, перезапустить программу с правами администратора.
- **Внедрение и обфускация.** Нет записи в чужие процессы (`WriteProcessMemory`, `CreateRemoteThread`), хуков, драйверов ядра, зашифрованного или закодированного кода. Фильтрация работает через штатный Windows Filtering Platform.
- **Встроенные данные.** Файл `bin/profile_internal.bin` вшивается в exe как ресурс. Это сжатый LZNT1 XML с заголовком `SWC1` и SHA-256. После распаковки он побайтно совпадает с открытым `bin/profile_internal.xml`, хэш в заголовке тоже сходится.
- **Изменения в системе.** Автозапуск и «запуск без UAC» (через планировщик задач) включаются только вручную в настройках.
- **Сборка.** В `simplewall.vcxproj` нет pre/post-build команд, выполняется только компиляция.

### Найденная слабость: небезопасное автообновление

Код в `routine/src/routine.c` (загрузка) и `routine/src/rapp.c`, функция `_r_update_install` (установка).

1. Если проверка TLS-сертификата сервера не прошла (`ERROR_WINHTTP_SECURE_FAILURE`), клиент повторяет запрос с флагами `SECURITY_FLAG_IGNORE_UNKNOWN_CA | SECURITY_FLAG_IGNORE_CERT_WRONG_USAGE`. Он принимает сертификат от недоверенного центра сертификации, в том числе самоподписанный.
2. Скачанный установщик запускается с правами администратора (`/u /S`) **без проверки цифровой подписи и хэша**.

**Последствия.** В чужой сети (публичный Wi-Fi, скомпрометированный роутер) злоумышленник может подменить обновление и выполнить свой код с правами администратора. Для этого пользователь должен нажать «установить» в окне обновления.

**Что делать.** Отключите *Settings → Periodically check for updates* и обновляйтесь вручную: пересоберите из исходников или скачайте официальный установщик со страницы Releases и проверьте его подпись.

## Сборка

### Требования

- Windows 10/11 x64.
- [Visual Studio Build Tools 2026](https://visualstudio.microsoft.com/downloads/) с нагрузкой **Desktop development with C++** (MSVC v145, Windows SDK 10.0.26100).
  Подойдёт и VS 2022 (toolset `v143`), тогда в команде сборки укажите `v143`.
- Git.

### 1. Исходники

Библиотека `routine` должна лежать **рядом** с папкой simplewall: проект ищет её по пути `..\routine`. Сабмодули с относительными путями git сам не подтягивает.

```powershell
git clone https://github.com/henrypp/simplewall.git
git clone https://github.com/henrypp/routine.git
git -C simplewall checkout d6fa5dfa
git -C routine checkout ff8f811
```

### 2. Пакет C++/WinRT

Проекту нужен NuGet-пакет `Microsoft.Windows.CppWinRT 2.0.230706.1`. Если `msbuild -t:restore` не находит его (на чистой машине в `%APPDATA%\NuGet\NuGet.Config` часто нет источников), скачайте пакет напрямую с nuget.org:

```powershell
$v   = "2.0.230706.1"
$dst = "simplewall\packages\Microsoft.Windows.CppWinRT.$v"
$pkg = "$env:TEMP\cppwinrt.$v.nupkg"
Invoke-WebRequest "https://api.nuget.org/v3-flatcontainer/microsoft.windows.cppwinrt/$v/microsoft.windows.cppwinrt.$v.nupkg" -OutFile $pkg -UseBasicParsing
Add-Type -A System.IO.Compression.FileSystem
New-Item -ItemType Directory -Force $dst | Out-Null
[IO.Compression.ZipFile]::ExtractToDirectory($pkg, (Resolve-Path $dst).Path)

# cppwinrt.exe должен быть подписан Microsoft Corporation
Get-AuthenticodeSignature "$dst\bin\cppwinrt.exe" | Select-Object Status, SignerCertificate
```

### 3. Компиляция

```powershell
$msbuild = "${env:ProgramFiles(x86)}\Microsoft Visual Studio\18\BuildTools\MSBuild\Current\Bin\MSBuild.exe"
& $msbuild simplewall\simplewall.sln -p:Configuration=Release -p:Platform=x64 -p:PlatformToolset=v145 -m
```

Результат: `simplewall\bin\64\simplewall.exe`. Сборка проходит без ошибок и предупреждений.

Сборка не побайтно воспроизводима: в exe попадают метки времени, поэтому хэш у каждого будет свой.

## Установка и использование

Установка не нужна. Для работы достаточно одного файла `simplewall.exe`: базовые правила вшиты внутрь, дополнительных DLL нет.

| Файл | Нужен? |
|---|---|
| `simplewall.exe` | да |
| `simplewall.lng` | только для неанглийского интерфейса, кладётся рядом с exe |
| `simplewall.pdb` | нет, это отладочные символы |

**Где хранятся настройки.** По умолчанию в `%APPDATA%\Henry++\simplewall\` (`simplewall.ini`, `profile.xml`). Если создать рядом с exe пустой файл `portable.dat`, настройки будут храниться в папке программы.

### Рекомендации при первом запуске

1. **Не ставьте галочку «Disable Windows Firewall».** simplewall и брандмауэр Windows работают через один механизм WFP и не мешают друг другу. Если simplewall упадёт или будет удалён, брандмауэр Windows продолжит защищать от входящих подключений.
2. **Для первого включения выберите «Temporary rules».** Тогда при ошибке в настройке правила сбросятся после перезагрузки. Когда всё заработает, включите «Permanent rules».
3. **Отключите проверку обновлений** (см. раздел про аудит).
4. **Включите список блокировки.** В меню **Blocklist** включите *Microsoft spying and telemetry*. *Microsoft update* не блокируйте, иначе перестанет работать Windows Update.
5. **Перед удалением exe нажмите «Disable filters».** Постоянные правила продолжают действовать и без программы.

### Системные процессы Windows, которые просят доступ в сеть

| Процесс | Что это | Рекомендация |
|---|---|---|
| `svchost.exe` | службы Windows, включая Windows Update и DNS | разрешить |
| `MoUsoCoreWorker.exe` | Update Orchestrator, управляет проверкой и установкой обновлений | разрешить, если нужны обновления |
| `WaaSMedicAgent.exe` | Windows Update Medic, чинит сломанный Центр обновления | лучше разрешить; без него обновления работают, но без автопочинки |
| `RUXIMICS.exe` | Reusable UX Interaction Manager (Update Health Tools): напоминания о конце поддержки Windows 10 и предложения Windows 11 / ESU | блокировать |
| `DeviceCensus.exe` | телеметрия: сбор сведений о компьютере для Microsoft | блокировать |
| Windows Default Lock Screen (`Microsoft.LockApp`) | картинки Spotlight, советы и реклама на экране блокировки | блокировать, если Spotlight не нужен |

simplewall фильтрует по приложению, а не по адресу: если вы разрешили одну программу, другие, которые обращаются к тому же серверу, доступа не получают.

## Лицензия

simplewall и routine распространяются по лицензии [GNU GPL v3](https://github.com/henrypp/simplewall/blob/master/LICENSE). Если вы публикуете собранный exe, указывайте ссылки на исходный код и коммиты, из которых он собран (см. таблицу «Коротко»).

Поддержать автора можно через ссылки в [README проекта](https://github.com/henrypp/simplewall#readme).

