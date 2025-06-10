import tkinter as tk
from tkinter import ttk, messagebox, filedialog
from tkcalendar import DateEntry
import csv
import os
from datetime import datetime
import subprocess
import platform
import sys

FILE_NAME = "bookings.csv"
HISTORY_FILE = "booking_history.csv"
CONFIG_FILE = "config.txt"

class TableBookingApp:
    def __init__(self, root):
        self.root = root
        self.scale_factor_font = 1.0
        self.root.bind("<Configure>", self.on_resize)
        self.root = root
        self.root.title("Бронювання столів")
        screen_width = self.root.winfo_screenwidth()
        screen_height = self.root.winfo_screenheight()
        width = int(screen_width * 0.95)
        height = int(screen_height * 0.95)
        self.root.geometry(f"{width}x{height}")
        # self.root.state("zoomed")  # замінено на автоматичне масштабування

        print("✅ Table Booking App started")

        style = ttk.Style()
        scale_factor = min(screen_width / 1366, screen_height / 768)
        base_font_size = int(14 * scale_factor)
        base_entry_size = int(13 * scale_factor)
        base_tree_font = int(13 * scale_factor)

        style.configure("TButton", font=("Arial", base_font_size), padding=6)
        style.configure("Edit.TButton", font=("Arial", base_font_size), padding=6, background="#d0f0c0")
        style.configure("TLabel", font=("Arial", base_font_size))
        style.configure("Treeview", rowheight=int(30 * scale_factor), font=("Arial", base_tree_font))
        style.configure("TEntry", font=("Arial", base_entry_size))

        self.keyboard_process = None
        self.bookings = []
        self.filtered_bookings = []
        self.sort_column = None
        self.sort_reverse = False
        self.base_path = ""
        self.selected_index = None
        

        self.setup_file_paths()
        self.load_bookings()
        
        self.root.protocol("WM_DELETE_WINDOW", self.on_close)
        self.auto_save()

        main_frame = ttk.Frame(self.root, padding=10)
        main_frame.pack(fill=tk.BOTH, expand=True)

        # Ввід бронювання
        entry_frame = ttk.Frame(main_frame)
        entry_frame.pack(fill=tk.X, pady=5)
        
        for i in range(10):
            entry_frame.columnconfigure(i, weight=1)

        label_name = ttk.Label(entry_frame, text="Ім'я клієнта:")
        label_name.grid(row=0, column=0, padx=5, pady=5)
        self.name_entry = ttk.Entry(entry_frame)
        self.name_entry.configure(width=max(10, int(self.root.winfo_width() / 70)))
        self.name_entry
        self.name_entry.grid(row=0, column=1, padx=5, pady=5)

        label_table = ttk.Label(entry_frame, text="Стіл №:")
        label_table.grid(row=0, column=2, padx=5, pady=5)
        self.table_entry = ttk.Combobox(entry_frame, state="normal")
        self.table_entry.configure(width=max(6, int(self.root.winfo_width() / 120)))
        self.table_entry
        self.table_entry.grid(row=0, column=3, padx=5, pady=5)
        self.load_table_list()
        self.table_entry.bind('<KeyRelease>', lambda e: self.table_entry.event_generate('<Down>'))

        label_date = ttk.Label(entry_frame, text="Дата:")
        label_date.grid(row=0, column=4, padx=5, pady=5)
        self.date_entry = DateEntry(entry_frame, date_pattern='dd-mm-yyyy')
        self.date_entry.configure(width=max(10, int(self.root.winfo_width() / 110)))
        self.date_entry.grid(row=0, column=5, padx=5, pady=5)

        label_phone = ttk.Label(entry_frame, text="Телефон:")
        label_phone.grid(row=0, column=6, padx=5, pady=5)
        self.phone_entry = ttk.Entry(entry_frame)
        self.phone_entry.configure(width=max(10, int(self.root.winfo_width() / 80)))
        self.phone_entry.grid(row=0, column=7, padx=5, pady=5)

        label_notes = ttk.Label(entry_frame, text="Примітки:")
        label_notes.grid(row=0, column=8, padx=5, pady=5)
        self.notes_entry = ttk.Entry(entry_frame)
        self.notes_entry.configure(width=max(15, int(self.root.winfo_width() / 60)))
        self.notes_entry.grid(row=0, column=9, padx=5, pady=5)

        self.submit_button = ttk.Button(entry_frame, text="Додати", command=self.add_or_update_booking)
        self.submit_button.grid(row=1, column=0, padx=int(10 * scale_factor), pady=int(5 * scale_factor))
        ttk.Button(entry_frame, text="Налаштувати столи", command=self.configure_table_list).grid(row=1, column=3, padx=int(10 * scale_factor), pady=int(5 * scale_factor))
        ttk.Button(entry_frame, text="Видалити", command=self.delete_booking).grid(row=1, column=1, padx=int(10 * scale_factor), pady=int(5 * scale_factor))
        ttk.Button(entry_frame, text="Очистити вибір", command=self.clear_selection).grid(row=1, column=2, padx=int(10 * scale_factor), pady=int(5 * scale_factor))

        # Таблиця
        tree_frame = ttk.Frame(main_frame)
        tree_frame.pack(fill=tk.BOTH, expand=True, pady=10, padx=10)

        self.tree = ttk.Treeview(tree_frame, columns=("Ім'я", "Стіл", "Дата", "Телефон", "Примітки"), show="headings")
        self.tree.heading("Ім'я", text="Ім'я", command=lambda: self.sort_by_column("Ім'я"))
        self.tree.heading("Стіл", text="Стіл", command=lambda: self.sort_by_column("Стіл"))
        self.tree.heading("Дата", text="Дата", command=lambda: self.sort_by_column("Дата"))
        self.tree.heading("Телефон", text="Телефон", command=lambda: self.sort_by_column("Телефон"))
        self.tree.heading("Примітки", text="Примітки", command=lambda: self.sort_by_column("Примітки"))
        self.tree.pack(fill=tk.BOTH, expand=True, side="left")
        self.tree.bind("<ButtonRelease-1>", self.on_row_select)

        tree_scroll = ttk.Scrollbar(tree_frame, orient="vertical", command=self.tree.yview)
        self.tree.configure(yscrollcommand=tree_scroll.set)
        tree_scroll.pack(side="right", fill="y")

        # Фільтри по кожному стовпцю
        filter_frame = ttk.Frame(main_frame)
        filter_frame.pack(fill=tk.X, padx=10, pady=5)
        

        label_filter_name = ttk.Label(filter_frame, text="Фільтр Ім'я:")
        label_filter_name.grid(row=0, column=0, padx=5)
        self.filter_name = ttk.Entry(filter_frame, width=20)
        self.filter_name.bind('<KeyRelease>', self.delayed_filter)
        self.filter_name.grid(row=0, column=1, padx=5)

        label_filter_table = ttk.Label(filter_frame, text="Фільтр Стіл:")
        label_filter_table.grid(row=0, column=2, padx=5)
        self.filter_table = ttk.Entry(filter_frame, width=10)
        self.filter_table.bind('<KeyRelease>', self.delayed_filter)
        self.filter_table.grid(row=0, column=3, padx=5)

        label_filter_date = ttk.Label(filter_frame, text="Фільтр Дата:")
        label_filter_date.grid(row=0, column=4, padx=5)
        self.filter_date = ttk.Entry(filter_frame, width=12)
        self.filter_date.bind("<Button-1>", self.open_calendar_popup)
        self.filter_date.grid(row=0, column=5, padx=5)
        
        # не встановлюємо початкове значення  # початково не заповнене
        # self.filter_date._top_cal.configure(takefocus=1)
        # self.filter_date._top_cal.transient(self.root)
        
        # ручне застосування фільтра, без автозаповнення          # дозволяє ручне введення
        
        self.filter_date.grid(row=0, column=5, padx=5)

        ttk.Button(filter_frame, text="Застосувати фільтр", command=self.apply_filters).grid(row=0, column=6, padx=int(5 * scale_factor))
        ttk.Button(filter_frame, text="Очистити фільтри", command=self.clear_filters).grid(row=0, column=7, padx=int(5 * scale_factor))

        self.refresh_tree()

        label_filter_phone = ttk.Label(filter_frame, text="Фільтр Телефон:")
        label_filter_phone.grid(row=1, column=0, padx=5)
        self.filter_phone = ttk.Entry(filter_frame, width=20)
        self.filter_phone.bind('<KeyRelease>', self.delayed_filter)
        self.filter_phone.grid(row=1, column=1, padx=5)

        label_filter_notes = ttk.Label(filter_frame, text="Фільтр Примітки:")
        label_filter_notes.grid(row=1, column=2, padx=5)
        self.filter_notes = ttk.Entry(filter_frame, width=30)
        self.filter_notes.bind('<KeyRelease>', self.delayed_filter)
        self.filter_notes.grid(row=1, column=3, padx=5)

    def on_row_select(self, event):
        selected = self.tree.focus()
        if selected:
            values = self.tree.item(selected, 'values')
            if values:
                self.name_entry.delete(0, tk.END)
                self.name_entry.insert(0, values[0])
                self.table_entry.delete(0, tk.END)
                self.table_entry.insert(0, values[1])
                self.date_entry.set_date(values[2])
                self.selected_item_id = selected
                self.selected_index = self.tree.index(selected)
                self.submit_button.config(text="Зберегти зміни", style="Edit.TButton")
                self.phone_entry.delete(0, tk.END)
                self.phone_entry.insert(0, values[3])
                self.notes_entry.delete(0, tk.END)
                self.notes_entry.insert(0, values[4])

        
    def save_table_list(self):
        table_list_file = os.path.join(self.base_path, "tables.csv")
        with open(table_list_file, 'w', newline='', encoding='utf-8') as f:
            writer = csv.writer(f)
            writer.writerow(self.table_entry["values"])

    def configure_table_list(self):
        top = tk.Toplevel(self.root)
        top.title("Налаштування столів")
        top.geometry("300x300")

        table_list_file = os.path.join(self.base_path, "tables.csv")

        ttk.Label(top, text="Введіть номери столів через кому:").pack(pady=10)
        entry = ttk.Entry(top, width=30)
        entry.pack(pady=10)
        entry.insert(0, ", ".join(self.table_entry["values"]))

        def save_new_values():
            values = [v.strip() for v in entry.get().split(",") if v.strip()]
            self.table_entry["values"] = values
            with open(table_list_file, 'w', newline='', encoding='utf-8') as f:
                writer = csv.writer(f)
                writer.writerow(values)
            top.destroy()

        ttk.Button(top, text="Зберегти", command=save_new_values).pack(pady=10)

    def load_table_list(self):
        table_list_file = os.path.join(self.base_path, "tables.csv")
        if os.path.exists(table_list_file):
            with open(table_list_file, newline='', encoding='utf-8') as f:
                reader = csv.reader(f)
                values = next(reader, [])
                self.table_entry["values"] = values

    def auto_save(self):
        self.save_bookings()
        self.root.after(60000, self.auto_save)

    def clear_filters(self):
        self.filter_name.delete(0, tk.END)
        self.filter_table.delete(0, tk.END)
        self.filter_date.delete(0, tk.END)
        self.refresh_tree()

    def apply_filters(self):
        name = self.filter_name.get().strip().lower()
        table = self.filter_table.get().strip()
        date = self.filter_date.get()
        phone = self.filter_phone.get().strip().lower()
        notes = self.filter_notes.get().strip().lower()
        if not date or date.strip() == '':
            date = None

        self.tree.delete(*self.tree.get_children())
        for row in self.bookings:
            try:
                formatted_date = datetime.strptime(row[2], '%Y-%m-%d').strftime('%d-%m-%Y')
            except ValueError:
                formatted_date = row[2]
            if name and name not in row[0].lower():
                continue
            if table and table != row[1]:
                continue
            try:
                if date:
                    filter_dt = datetime.strptime(date, '%d-%m-%Y').date()
                    row_dt = datetime.strptime(row[2], '%Y-%m-%d').date()
                    if filter_dt != row_dt:
                        continue
            except ValueError:
                # некоректна дата у фільтрі — пропускаємо
                continue
                continue
            if phone and phone not in row[3].lower():
                continue
            if notes and notes not in row[4].lower():
                continue
            self.tree.insert("", "end", values=(row[0], row[1], formatted_date, row[3], row[4]))

    def open_calendar_popup(self, event=None):
        top = tk.Toplevel(self.root)
        top.wm_title("Оберіть дату")
        top.geometry(f"250x220+{self.filter_date.winfo_rootx()}+{self.filter_date.winfo_rooty() - 220}")
        cal = DateEntry(top, date_pattern='dd-mm-yyyy')
        cal.pack(padx=10, pady=10)
        def select_and_close():
            self.filter_date.delete(0, tk.END)
            self.filter_date.insert(0, cal.get())
            top.destroy()
            self.apply_filters()
        ttk.Button(top, text="Ок", command=select_and_close).pack(pady=5)
        ttk.Button(top, text="[X] Скасувати", command=top.destroy).pack(pady=5)


        self.filter_name.delete(0, tk.END)
        self.filter_table.delete(0, tk.END)
        self.filter_date.delete(0, tk.END)
        self.refresh_tree()

    def delayed_filter(self, event=None):
        if hasattr(self, '_filter_after_id') and self._filter_after_id:
            self.root.after_cancel(self._filter_after_id)
        self._filter_after_id = self.root.after(500, self.apply_filters)

    def refresh_tree(self):
        self.tree.delete(*self.tree.get_children())
        for row in self.bookings:
            formatted_date = datetime.strptime(row[2], '%Y-%m-%d').strftime('%d-%m-%Y')
            self.tree.insert("", "end", values=(row[0], row[1], formatted_date, row[3], row[4]))

    def add_or_update_booking(self):
        name = self.name_entry.get().strip()
        table = self.table_entry.get().strip()
        date_display = self.date_entry.get_date().strftime('%d-%m-%Y')
        date_save = self.date_entry.get_date().strftime('%Y-%m-%d')
        phone = self.phone_entry.get().strip()
        notes = self.notes_entry.get().strip()

        if not name or not table or not date_display:
            messagebox.showwarning("Помилка", "Усі поля обов’язкові")
            return

        new_entry = [name, table, date_save, phone, notes]

        if self.selected_index is not None:
            for i, booking in enumerate(self.bookings):
                if i != self.selected_index and booking[1] == table and booking[2] == date_save:
                    messagebox.showerror("Зайнято", f"Стіл №{table} вже заброньований на {date_display}.")
                    return

            if hasattr(self, 'selected_item_id'):
                row_id = self.selected_item_id
                self.tree.item(row_id, values=(name, table, date_display, phone, notes))
                self.bookings[self.selected_index] = new_entry
            self.selected_index = None
        else:
            for booking in self.bookings:
                if booking[1] == table and booking[2] == date_save:
                    messagebox.showerror("Зайнято", f"Стіл №{table} вже заброньований на {date_display}.")
                    return
            self.bookings.append(new_entry)

        self.refresh_tree()
        self.save_bookings()

        self.name_entry.delete(0, tk.END)
        self.table_entry.delete(0, tk.END)
        self.phone_entry.delete(0, tk.END)
        self.notes_entry.delete(0, tk.END)
        self.date_entry.set_date(datetime.today())
        self.submit_button.config(text="Додати", style="TButton")
        

    def clear_selection(self):
        self.selected_index = None
        self.name_entry.delete(0, tk.END)
        self.table_entry.delete(0, tk.END)
        self.table_entry.delete(0, tk.END)
        self.phone_entry.delete(0, tk.END)
        self.notes_entry.delete(0, tk.END)
        self.date_entry.set_date(datetime.today())
        self.submit_button.config(text="Додати", style="TButton")

    def delete_booking(self):
        selected = self.tree.selection()
        if not selected:
            messagebox.showinfo("Інформація", "Оберіть запис для видалення")
            return
        if not messagebox.askyesno("Підтвердження", "Ви впевнені, що хочете видалити запис?"):
            return
        index = self.tree.index(selected[0])
        del self.bookings[index]
        self.refresh_tree()
        self.save_bookings()

    def launch_osk(self, event=None):
        if platform.system() == "Windows":
            if not self.keyboard_process:
                try:
                    self.keyboard_process = subprocess.Popen("osk.exe")
                except Exception as e:
                    print("❌ Не вдалося відкрити екранну клавіатуру:", e)
        elif platform.system() == "Linux":
            try:
                self.keyboard_process = subprocess.Popen(["onboard"])
            except FileNotFoundError:
                print("❌ 'onboard' не знайдено. Встановіть через: sudo apt install onboard")

    def close_osk(self, event=None):
        if self.keyboard_process:
            self.keyboard_process.terminate()
            self.keyboard_process = None

    def setup_file_paths(self):
        path = ""
        if os.path.exists(CONFIG_FILE):
            with open(CONFIG_FILE, "r", encoding="utf-8") as f:
                path = f.read().strip()
        if not path or not os.path.isdir(path):
            path = filedialog.askdirectory(title="Оберіть мережеву папку для зберігання файлів")
        if not path:
            messagebox.showerror("Помилка", "Не обрано директорію для збереження даних. Програма завершить роботу.")
            self.root.destroy()
        else:
            self.base_path = path
            print("📂 BASE PATH:", self.base_path)
            with open(CONFIG_FILE, "w", encoding="utf-8") as f:
                f.write(self.base_path)
            global FILE_NAME, HISTORY_FILE
            FILE_NAME = os.path.join(self.base_path, "bookings.csv")
            HISTORY_FILE = os.path.join(self.base_path, "booking_history.csv")

    def load_bookings(self):
        self.create_backup()

        if not os.path.exists(FILE_NAME):
            with open(FILE_NAME, 'w', newline='', encoding="utf-8") as f:
                writer = csv.writer(f)
                writer.writerow(["Ім'я", "Стіл", "Дата", "Телефон", "Примітки"])
        with open(FILE_NAME, newline='', encoding="utf-8") as f:
            reader = csv.reader(f)
            next(reader, None)
            self.bookings = [row if len(row) == 5 else row + ["", ""] for row in reader]

    def save_bookings(self):
        with open(FILE_NAME, 'w', newline='', encoding="utf-8") as f:
            writer = csv.writer(f)
            writer.writerow(["Ім'я", "Стіл", "Дата", "Телефон", "Примітки"])
            writer.writerows(self.bookings)

    def create_backup(self):
        if os.path.exists(FILE_NAME):
            backup_dir = os.path.dirname(FILE_NAME)
            timestamp = datetime.now().strftime('%Y-%m-%d_%H-%M-%S')
            backup_path = os.path.join(backup_dir, f"bookings_backup_{timestamp}.csv")
            with open(FILE_NAME, 'r', encoding='utf-8') as original, open(backup_path, 'w', encoding='utf-8') as backup:
                backup.write(original.read())

    def on_resize(self, event):
        width = self.root.winfo_width()
        base_width = 1366
        scale = max(0.7, min(1.5, width / base_width))
        if abs(self.scale_factor_font - scale) > 0.05:
            self.scale_factor_font = scale
            style = ttk.Style()
            style.configure("TButton", font=("Arial", int(14 * scale)))
            style.configure("Edit.TButton", font=("Arial", int(14 * scale)))
            style.configure("TLabel", font=("Arial", int(14 * scale)))
            style.configure("TEntry", font=("Arial", int(13 * scale)))
            style.configure("Treeview", font=("Arial", int(13 * scale)), rowheight=int(30 * scale))

    def on_close(self):
        self.save_bookings()
        self.save_table_list()
        self.root.destroy()

    def sort_by_column(self, col):
        col_index = {"Ім'я": 0, "Стіл": 1, "Дата": 2}[col]
        self.sort_reverse = not self.sort_reverse if self.sort_column == col else False
        self.sort_column = col
        self.bookings.sort(key=lambda x: x[col_index], reverse=self.sort_reverse)
        self.refresh_tree()

if __name__ == "__main__":
    import socket
    try:
        single_instance_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        single_instance_socket.bind(("127.0.0.1", 65432))  # порт можна змінити при потребі
    except OSError:
        import tkinter as tk
        from tkinter import messagebox
        root = tk.Tk()
        root.withdraw()
        messagebox.showerror("Помилка", "Програма вже запущена!")
        sys.exit()

    print("🚀 Запуск TableBooking...")
    root = tk.Tk()
    app = TableBookingApp(root)
    root.mainloop()
