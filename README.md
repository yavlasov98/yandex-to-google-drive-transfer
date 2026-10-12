# 🚀 Yandex Disk → Google Drive

Перенос файлов и папок с Яндекс Диска в Google Drive через виртуальную машину Google Colab. Архив не скачивается на компьютер: данные проходят через Colab и записываются в Google Drive.

## ⚠️ Требования и ограничения

- **Яндекс 360:** согласно [справке Яндекса](https://yandex.com/support/yandex-360/customers/disk/web/en/webdav), с 22 июня 2026 года доступ к WebDAV требует активного плана Яндекс 360.
- **Транзит через Colab:** данные идут по маршруту `Яндекс Диск → виртуальная машина Colab → Google Drive`. Это не прямая передача между облачными серверами.
- **Работа Colab:** среда может отключиться, а её доступность и время работы не гарантированы. На скорость влияют Colab, Яндекс и Google Drive.
- **Повторный запуск:** команда `copy` не удаляет исходные файлы. При повторе уже скопированные файлы обычно пропускаются; прерванный файл может копироваться заново.
- **Проверка результата:** перед удалением оригиналов самостоятельно проверьте файлы и папки в Google Drive. Этот код ничего не удаляет на Яндекс Диске.

## 🔐 Безопасность

- Используйте отдельный **пароль приложения WebDAV**, а не основной пароль Яндекса.
- Не отправляйте пароль приложения помощнику и не публикуйте его в блокноте, скриншотах или переписке.
- `getpass` скрывает пароль при вводе, а `rclone obscure` обратимо маскирует его в конфигурации. Это **не полноценное шифрование**.
- Подключение Google Drive разрешает коду блокнота доступ к вашему диску. Перед запуском просмотрите код.
- После переноса отзовите пароль приложения в настройках Яндекс ID.

## 🛠 Как запустить

1. Убедитесь, что на аккаунте Яндекса доступен WebDAV.
2. Создайте пароль приложения WebDAV в [настройках Яндекс ID](https://id.yandex.ru/security/app-passwords).
3. Откройте [Google Colab](https://colab.research.google.com/) и создайте новый блокнот.
4. Вставьте код ниже в одну ячейку и запустите её.
5. Разрешите доступ к Google Drive, введите логин Яндекса и пароль приложения.
6. После завершения проверьте папку `Yandex_Backup` на Google Диске.

> Для начала попробуйте перенести небольшую тестовую папку.

## 📦 Нужна помощь с переносом под ключ?
Если нужно перенести сотни гигабайт или терабайты данных, сохранить сложную структуру папок или просто не хочется вникать в консоль:
Безопасно: настройка и запуск производятся в вашей личной сессии без передачи паролей третьим лицам.
Под ключ: сопровождение процесса, контроль целостности и возобновление при обрывах.
👉 Написать в Telegram: @anvlasov1 — помогу быстро и без потери данных

## 💻 Код

```python
from google.colab import drive
from pathlib import Path
import getpass
import os
import subprocess

# 1. Подключаем Google Drive
drive.mount("/content/drive")

target_dir = Path("/content/drive/MyDrive/Yandex_Backup")
target_dir.mkdir(parents=True, exist_ok=True)

# 2. Устанавливаем rclone
subprocess.run(["apt-get", "update", "-qq"], check=True)
subprocess.run(
    ["apt-get", "install", "-y", "-qq", "rclone"],
    check=True
)

# 3. Вводим данные WebDAV
yandex_user = input("Логин Яндекса: ").strip()
yandex_password = getpass.getpass(
    "Пароль приложения WebDAV: "
).strip()

if not yandex_user or not yandex_password:
    raise ValueError("Нужно ввести логин и пароль приложения.")

# 4. Маскируем пароль для конфигурации rclone
result = subprocess.run(
    ["rclone", "obscure", yandex_password],
    check=True,
    capture_output=True,
    text=True
)
obscured_password = result.stdout.strip()

# Удаляем лишнюю ссылку на пароль из переменной
del yandex_password

# 5. Создаём временную конфигурацию WebDAV
config_path = Path("/content/rclone-migration.conf")
config_text = (
    "[yandex_webdav]\n"
    "type = webdav\n"
    "url = https://webdav.yandex.ru\n"
    "vendor = other\n"
    f"user = {yandex_user}\n"
    f"pass = {obscured_password}\n"
)
config_path.write_text(config_text, encoding="utf-8")
os.chmod(config_path, 0o600)

del obscured_password

# 6. Проверяем подключение и список папок
subprocess.run(
    ["rclone", "lsd", "yandex_webdav:", "--config", str(config_path)],
    check=True
)

# 7. Копируем файлы в Google Drive
subprocess.run(
    [
        "rclone", "copy",
        "yandex_webdav:",
        str(target_dir),
        "--config", str(config_path),
        "--progress",
        "--transfers", "4",
        "--checkers", "8",
        "--retries", "5",
        "--low-level-retries", "10",
    ],
    check=True
)

print(f"Копирование завершено. Проверьте папку: {target_dir}")
