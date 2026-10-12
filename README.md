# 🚀 Yandex.Disk to Google Drive Cloud Migrator

Прямой перенос архивов и фото из **Яндекс.Диска** в **Google Диск** без скачивания на компьютер через виртуальную машину Google Colab.

---

## ⚡ Преимущества
- **0 байт на диске ПК:** память компьютера не расходуется.
- **Высокая скорость:** перенос идет напрямую между дата-центрами.
- **Безопасность:** авторизация происходит только в вашей изолированной сессии Google.

---

## 🛠 Как запустить перенос

1. Создайте пароль приложения в [Яндекс ID -> Безопасность -> Пароли приложений](https://id.yandex.ru/security/app-passwords) (выберите тип: **Файлы WebDAV**).
2. Откройте [Google Colab](https://colab.research.google.com/) и создайте новый блокнот.
3. Вставьте и запустите код:

```python
from google.colab import drive
import os, getpass

drive.mount('/content/drive')
target_dir = '/content/drive/MyDrive/Yandex_Backup'
os.makedirs(target_dir, exist_ok=True)

!curl -s [https://rclone.org/install.sh](https://rclone.org/install.sh) | bash > /dev/null

yandex_user = input('Логин Яндекс: ')
yandex_pass = getpass.getpass('Пароль приложения WebDAV: ')

config = f"""
[yandex]
type = yandex
user = {yandex_user}
pass = {yandex_pass}
"""

os.makedirs('/root/.config/rclone', exist_ok=True)
with open('/root/.config/rclone/rclone.conf', 'w') as f:
    f.write(config)

!rclone copy -P --transfers=8 --checkers=16 yandex:/ "{target_dir}"

🤝 Помощь с переносом под ключ
Если нужно перенести большие объемы данных без настройки:
Telegram: @anvlasov1
