# 🚀 Yandex.Disk to Google Drive Cloud Migrator

Прямой перенос архивов и файлов из **Яндекс Диска** в **Google Диск** через виртуальную машину Google Colab — без скачивания на локальный компьютер.

---

## ⚠️ Важные требования и ограничения
- **Подписка Яндекс 360:** доступ к Яндекс Диску по протоколу WebDAV предоставляется только пользователям с активным тарифом Яндекс 360.
- **Архитектура переноса:** локальный накопитель не задействуется, трафик идет транзитом: `Яндекс Диск → виртуальная машина Colab → Google Диск`.
- **Безопасность данных:** используется режим `rclone copy` (исходные файлы на Яндексе не удаляются). Если связь оборвется, повторный запуск продолжит перенос без дублирования уже скачанных файлов.

---

## ⚡ Преимущества
- **0 байт на диске ПК:** свободное место на компьютере не расходуется.
- **Прямой облачный трафик:** перенос идет на сетевых мощностях Google Colab.
- **Идемпотентность:** при перезапуске уже переданные файлы автоматически пропускаются.

---

## 🛠 Как запустить перенос

1. Убедитесь, что на аккаунте активен тариф **Яндекс 360**.
2. Создайте пароль приложения: перейдите в [Яндекс ID → Безопасность → Пароли приложений](https://id.yandex.ru/security/app-passwords) и выберите тип **Файлы (WebDAV)**.
3. Откройте [Google Colab](https://colab.research.google.com/), создайте новый блокнот и вставьте единый блок кода:

```python
from google.colab import drive
import os, getpass, subprocess

# 1. Монтируем Google Диск
drive.mount('/content/drive')
target_dir = '/content/drive/MyDrive/Yandex_Backup'
os.makedirs(target_dir, exist_ok=True)

# 2. Устанавливаем rclone
!curl -s [https://rclone.org/install.sh](https://rclone.org/install.sh) | bash > /dev/null

# 3. Ввод учетных данных
yandex_user = input("Логин Яндекс (без @yandex.ru): ").strip()
yandex_pass = getpass.getpass("Пароль приложения WebDAV: ").strip()

# 4. Создание защищенной конфигурации rclone
subprocess.run([
    "rclone", "config", "create", "yandex_backup", "webdav",
    "url=[https://webdav.yandex.ru](https://webdav.yandex.ru)",
    "vendor=other",
    f"user={yandex_user}",
    f"pass={yandex_pass}"
], check=True)

# 5. Выбор папки (Enter для переноса всего диска)
folder_name = input("Какую папку перенести? (Enter для ВСЕГО диска): ").strip()
source = f"yandex_backup:{folder_name}" if folder_name else "yandex_backup:/"

print(f"\n🚀 Запуск процесса переноса из [{source}] в [{target_dir}]...")
print("⏳ Пожалуйста, не закрывайте вкладку блокнота во время копирования.\n")

# 6. Надежное копирование с защитой от rate limit Яндекса
cmd = [
    "rclone", "copy", source, target_dir,
    "--exclude", ".Trash/**",
    "--exclude", "Корзина/**",
    "--transfers", "2",
    "--checkers", "1",
    "--tpslimit", "2",
    "--retries", "5",
    "--low-level-retries", "10",
    "--buffer-size", "32M",
    "-v", "--stats", "10s"
]

subprocess.run(cmd, check=True)

print("\n🎉 Перенос успешно завершен! Все файлы сохранены в Google Drive.")
```

---

> ### 📦 **Нужна помощь с переносом под ключ?**
> 
> Если нужно перенести **сотни гигабайт или терабайты данных**, сохранить сложную структуру каталогов или нет желания возиться с окружением:
> 
> - **Безопасно:** запуск производится в вашей личной сессии без передачи паролей третьим лицам.
> - **Контроль целостности:** проверка завершения передачи каждого файла и решение возможных сетевых сбоев.
> 
> 👉 **Написать в Telegram:** [@anvlasov1](https://t.me/anvlasov1) — помогу настроить и провести миграцию.
