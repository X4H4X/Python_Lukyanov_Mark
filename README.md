Разработка клиента для VPN


![Alt text](/build/app/112.png?raw=true "Optional Title")
![Alt text](/build/app/113.png?raw=true "Optional Title")




import customtkinter as ctk
import sqlite3
import hashlib
import subprocess
import socket
import os
import sys

Настройка темы
ctk.set_appearance_mode("dark")
ctk.set_default_color_theme("blue")

class VPNApp(ctk.CTk):
    def __init__(self):
        super().__init__()
        self.title("XXX")
        self.geometry("500x600")
        
        self.db_init()
        self.vpn_process = None
        self.current_user = None
        
        # Сначала показываем экран логина
        self.show_login_screen()

    def db_init(self):
        conn = sqlite3.connect('vpn_data.db')
        conn.execute('CREATE TABLE IF NOT EXISTS users (user TEXT PRIMARY KEY, pwd TEXT)')
        conn.execute('CREATE TABLE IF NOT EXISTS exclusions (user TEXT, site TEXT)')
        conn.commit()
        conn.close()

    def show_login_screen(self):
        self.clear_screen()
        ctk.CTkLabel(self, text="Авторизация", font=("Arial", 24)).pack(pady=20)
        
        self.u_entry = ctk.CTkEntry(self, placeholder_text="Логин")
        self.u_entry.pack(pady=10)
        
        self.p_entry = ctk.CTkEntry(self, placeholder_text="Пароль", show="*")
        self.p_entry.pack(pady=10)
        
        ctk.CTkButton(self, text="Войти", command=self.login).pack(pady=5)
        ctk.CTkButton(self, text="Регистрация", fg_color="transparent", command=self.register).pack()

    def login(self):
        user = self.u_entry.get()
        pwd = hashlib.sha256(self.p_entry.get().encode()).hexdigest()
        
        conn = sqlite3.connect('vpn_data.db')
        res = conn.execute('SELECT * FROM users WHERE user=? AND pwd=?', (user, pwd)).fetchone()
        if res:
            self.current_user = user
            self.show_main_screen()
        else:
            print("Ошибка входа")
        conn.close()

    def register(self):
        user = self.u_entry.get()
        pwd = hashlib.sha256(self.p_entry.get().encode()).hexdigest()
        try:
            conn = sqlite3.connect('vpn_data.db')
            conn.execute('INSERT INTO users VALUES (?,?)', (user, pwd))
            conn.commit()
            print("Успешная регистрация")
        except:
            print("Пользователь уже есть")

    def show_main_screen(self):
        self.clear_screen()
        ctk.CTkLabel(self, text=f"Привет, {self.current_user}", font=("Arial", 18)).pack(pady=10)
        
        # Выбор страны (нужны файлы .ovpn в папке configs)
        self.server_var = ctk.StringVar(value="USA")
        ctk.CTkOptionMenu(self, values=["USA", "Germany", "France"], variable=self.server_var).pack(pady=10)
        
        self.status_lbl = ctk.CTkLabel(self, text="Статус: Отключено", text_color="gray")
        self.status_lbl.pack(pady=5)
        
        self.conn_btn = ctk.CTkButton(self, text="Подключиться", fg_color="green", command=self.toggle_vpn)
        self.conn_btn.pack(pady=20)
        
        # Блок исключений
        ctk.CTkLabel(self, text="Исключить сайт (Split Tunneling):").pack()
        self.site_entry = ctk.CTkEntry(self, placeholder_text="example.com")
        self.site_entry.pack(pady=5)
        ctk.CTkButton(self, text="Добавить в исключения", command=self.add_exclusion).pack()

    def toggle_vpn(self):
        if not self.vpn_process:
            config = f"configs/{self.server_var.get()}.ovpn"
            if os.path.exists(config):
                # Запуск OpenVPN (требуются права админа)
                self.vpn_process = subprocess.Popen(['openvpn', '--config', config], 
                                                    shell=True, stdout=subprocess.PIPE)
                self.status_lbl.configure(text="Статус: ПОДКЛЮЧЕНО", text_color="green")
                self.conn_btn.configure(text="Отключить", fg_color="red")
        else:
            os.system("taskkill /f /im openvpn.exe") # Грубое закрытие для Windows
            self.vpn_process = None
            self.status_lbl.configure(text="Статус: Отключено", text_color="gray")
            self.conn_btn.configure(text="Подключиться", fg_color="green")

    def add_exclusion(self):
        site = self.site_entry.get()
        try:
            ip = socket.gethostbyname(site)
            # Команда для Windows: направляем трафик сайта через ваш реальный роутер (шлюз)
            # Нужно заранее знать IP шлюза, например 192.168.1.1
            os.system(f"route add {ip} mask 255.255.255.255 192.168.1.1")
            print(f"Сайт {site} ({ip}) добавлен в исключения")
        except:
            print("Не удалось разрешить IP сайта")

    def clear_screen(self):
        for widget in self.winfo_children():
            widget.destroy()

Код представляет собой заготовку для десктопного VPN-клиента на Python с использованием библиотек CustomTkinter (интерфейс) и sqlite3 (база данных).

1. Системные переменные (состояние приложения)
self.vpn_process: Хранит объект запущенного процесса OpenVPN. Если значение None — VPN выключен. Если там объект — значит, программа «помнит», какой процесс нужно контролировать или закрывать.
self.current_user: Строковая переменная, куда записывается имя пользователя после успешного входа. Используется для персонализации интерфейса (например, вывода приветствия).
conn: Объект соединения с базой данных SQLite (vpn_data.db). Через него программа «общается» с файлом БД.
res: Результат выполнения SQL-запроса (от англ. result). В коде используется для проверки, нашелся ли пользователь с таким логином и паролем в базе.

2. Элементы интерфейса (Виджеты)
self.u_entry и self.p_entry: Поля ввода (Entry) для логина и пароля на экране авторизации.
self.server_var: Специальная переменная Tkinter (StringVar), которая в реальном времени хранит текст выбранной страны из выпадающего списка.
self.status_lbl: Текстовая метка (Label), которая показывает текущее состояние: «Статус: Отключено» или «ПОДКЛЮЧЕНО».
self.conn_btn: Кнопка управления подключением. Программа меняет её цвет (зеленый/красный) и текст в зависимости от состояния VPN.
self.site_entry: Поле ввода, куда пользователь вписывает адрес сайта (например, google.com), который он хочет пустить в обход VPN.
