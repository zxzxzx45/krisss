import tkinter as tk
from tkinter import ttk, messagebox
import random
import json
import os
from datetime import datetime

# Константы для символов
DIGITS = "0123456789"
LOWERCASE = "abcdefghijklmnopqrstuvwxyz"
UPPERCASE = "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
LETTERS = LOWERCASE + UPPERCASE
SPECIALS = "!@#$%^&*()_+-=[]{}|;:,.<>?/~"

# Файл для сохранения истории
HISTORY_FILE = "password_history.json"

class PasswordGeneratorApp:
    def __init__(self, root):
        self.root = root
        self.root.title("Random Password Generator")
        self.root.geometry("750x500")
        self.root.resizable(True, True)

        # Переменные для настроек
        self.password_length = tk.IntVar(value=12)
        self.use_digits = tk.BooleanVar(value=True)
        self.use_letters = tk.BooleanVar(value=True)
        self.use_specials = tk.BooleanVar(value=False)

        # История паролей (список словарей)
        self.history = []

        # Загрузка истории из файла
        self.load_history()

        # Создание интерфейса
        self.create_widgets()

        # Обновить отображение истории
        self.update_history_display()

    def create_widgets(self):
        # Главный фрейм для настроек
        settings_frame = ttk.LabelFrame(self.root, text="Настройки пароля", padding=10)
        settings_frame.pack(fill=tk.X, padx=10, pady=5)

        # Ползунок длины пароля
        ttk.Label(settings_frame, text="Длина пароля:").grid(row=0, column=0, sticky=tk.W, pady=5)
        self.length_scale = ttk.Scale(settings_frame, from_=1, to=50, orient=tk.HORIZONTAL,
                                      variable=self.password_length, command=self.update_length_label)
        self.length_scale.grid(row=0, column=1, sticky=tk.EW, padx=5, pady=5)
        self.length_label = ttk.Label(settings_frame, text=str(self.password_length.get()))
        self.length_label.grid(row=0, column=2, padx=5, pady=5)

        # Чекбоксы
        ttk.Checkbutton(settings_frame, text="Цифры (0-9)", variable=self.use_digits).grid(row=1, column=0, sticky=tk.W, pady=2)
        ttk.Checkbutton(settings_frame, text="Буквы (A-Z, a-z)", variable=self.use_letters).grid(row=1, column=1, sticky=tk.W, pady=2)
        ttk.Checkbutton(settings_frame, text="Спецсимволы", variable=self.use_specials).grid(row=1, column=2, sticky=tk.W, pady=2)

        # Кнопка генерации
        self.generate_btn = ttk.Button(settings_frame, text="Сгенерировать пароль", command=self.on_generate)
        self.generate_btn.grid(row=2, column=0, columnspan=3, pady=10)

        # Кнопка очистки истории
        self.clear_btn = ttk.Button(settings_frame, text="Очистить историю", command=self.clear_history)
        self.clear_btn.grid(row=3, column=0, columnspan=3, pady=5)

        # Фрейм для таблицы истории
        history_frame = ttk.LabelFrame(self.root, text="История паролей", padding=10)
        history_frame.pack(fill=tk.BOTH, expand=True, padx=10, pady=5)

        # Таблица (Treeview)
        columns = ("datetime", "password", "length")
        self.tree = ttk.Treeview(history_frame, columns=columns, show="headings")
        self.tree.heading("datetime", text="Дата и время")
        self.tree.heading("password", text="Пароль")
        self.tree.heading("length", text="Длина")
        self.tree.column("datetime", width=150)
        self.tree.column("password", width=350)
        self.tree.column("length", width=60)

        # Добавляем скроллинг
        scrollbar = ttk.Scrollbar(history_frame, orient=tk.VERTICAL, command=self.tree.yview)
        self.tree.configure(yscrollcommand=scrollbar.set)
        self.tree.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)
        scrollbar.pack(side=tk.RIGHT, fill=tk.Y)

        # Двойной клик для копирования пароля
        self.tree.bind("<Double-1>", self.copy_password)

        # Настройка растягивания колонок
        settings_frame.columnconfigure(1, weight=1)

    def update_length_label(self, event=None):
        """Обновляет текстовую метку длины пароля."""
        self.length_label.config(text=str(self.password_length.get()))

    def get_character_pool(self):
        """Возвращает строку символов для генерации пароля на основе выбранных чекбоксов."""
        pool = ""
        if self.use_digits.get():
            pool += DIGITS
        if self.use_letters.get():
            pool += LETTERS
        if self.use_specials.get():
            pool += SPECIALS
        return pool

    def generate_password(self, length):
        """Генерирует случайный пароль заданной длины."""
        pool = self.get_character_pool()
        if not pool:
            raise ValueError("Не выбран ни один тип символов!")
        # Используем random.choices для возможности повторения символов
        return ''.join(random.choices(pool, k=length))

    def on_generate(self):
        """Обработчик нажатия кнопки генерации."""
        length = self.password_length.get()
        # Проверка длины (дополнительная)
        if length < 1:
            messagebox.showerror("Ошибка", "Длина пароля должна быть не менее 1")
            return
        if length > 100:
            messagebox.showerror("Ошибка", "Длина пароля не должна превышать 100")
            return

        try:
            password = self.generate_password(length)
        except ValueError as e:
            messagebox.showerror("Ошибка", str(e))
            return

        # Добавляем запись в историю
        record = {
            "datetime": datetime.now().isoformat(),
            "password": password,
            "length": length
        }
        self.history.append(record)
        self.save_history()
        self.update_history_display()

        # Показываем сгенерированный пароль в отдельном окне (опционально)
        messagebox.showinfo("Сгенерированный пароль", f"Ваш новый пароль:\n{password}")

    def load_history(self):
        """Загружает историю из JSON-файла."""
        if os.path.exists(HISTORY_FILE):
            try:
                with open(HISTORY_FILE, "r", encoding="utf-8") as f:
                    self.history = json.load(f)
            except (json.JSONDecodeError, IOError):
                self.history = []
        else:
            self.history = []

    def save_history(self):
        """Сохраняет историю в JSON-файл."""
        with open(HISTORY_FILE, "w", encoding="utf-8") as f:
            json.dump(self.history, f, ensure_ascii=False, indent=2)

    def update_history_display(self):
        """Обновляет отображение таблицы истории."""
        # Очищаем таблицу
        for row in self.tree.get_children():
            self.tree.delete(row)

        # Сортируем историю по дате (от новых к старым)
        sorted_history = sorted(self.history, key=lambda x: x["datetime"], reverse=True)
        for record in sorted_history:
            # Форматируем дату для отображения
            try:
                dt = datetime.fromisoformat(record["datetime"])
                dt_str = dt.strftime("%Y-%m-%d %H:%M:%S")
            except:
                dt_str = record["datetime"]
            self.tree.insert("", tk.END, values=(dt_str, record["password"], record["length"]))

    def clear_history(self):
        """Очищает историю после подтверждения пользователя."""
        if messagebox.askyesno("Подтверждение", "Вы действительно хотите очистить всю историю паролей?"):
            self.history = []
            self.save_history()
            self.update_history_display()

    def copy_password(self, event):
        """Копирует пароль из выбранной строки таблицы в буфер обмена."""
        selected = self.tree.selection()
        if selected:
            item = self.tree.item(selected[0])
            password = item["values"][1]
            self.root.clipboard_clear()
            self.root.clipboard_append(password)
            messagebox.showinfo("Скопировано", "Пароль скопирован в буфер обмена!")

if __name__ == "__main__":
    root = tk.Tk()
    app = PasswordGeneratorApp(root)
    root.mainloop()
