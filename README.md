import tkinter as tk
from tkinter import messagebox
import json
from os.path import exists
import random

# Файл для сохранения данных пользователей
DATA_FILE = "data.json"

# Загрузка пользователей из файла, если файл существует, или создание пустого списка
def load_users():
    if exists(DATA_FILE):
        with open(DATA_FILE, "r") as file:
            return json.load(file)
    return []

# Сохранение пользователей в файл
def save_users():
    with open(DATA_FILE, "w") as file:
        json.dump(USERS, file, indent=4)

# Функция для добавления нового пользователя
def add_user():
    name = name_entry.get()
    if name:  # Проверка, что имя введено
        password = password_entry.get()
        if not password:
            messagebox.showwarning("Предупреждение", "Пожалуйста, введите пароль.")
            return
        if len(password) < 6:
            messagebox.showerror("Ошибка", "Пароль должен содержать не менее 6 символов.")
            return
        # Создание нового пользователя
        new_user = {"id": len(USERS) + 1, "name": name, "password": password}
        USERS.append(new_user)
        save_users()  # Сохранение пользователей в файл
        update_listbox()  # Обновление списка пользователей
        name_entry.delete(0, tk.END)
        password_entry.delete(0, tk.END)
        
        # Переключение на экран выбора сложности
        login_frame.pack_forget()  # Скрываем экран входа
        difficulty_frame.pack()  # Показываем экран выбора сложности
    else:
        messagebox.showwarning("Предупреждение", "Пожалуйста, введите имя.")

# Функция для обновления списка пользователей
def update_listbox():
    listbox.delete(0, tk.END)  # Очищаем текущий список
    for user in USERS:
        listbox.insert(tk.END, user["name"])  # Отображаем имена всех пользователей

# Функция для входа в систему
def login():
    selected_user = listbox.curselection()
    if not selected_user:  # Проверка, что пользователь выбран
        messagebox.showwarning("Предупреждение", "Пожалуйста, выберите пользователя.")
        return

    selected_name = listbox.get(selected_user[0])  # Получаем имя выбранного пользователя
    password = password_entry.get()
    user = next((u for u in USERS if u["name"] == selected_name and u["password"] == password), None)

    if user:  # Если логин и пароль совпадают
        messagebox.showinfo("Успех", f"Добро пожаловать, {user['name']}!")
        
        # Переключение на экран выбора сложности
        login_frame.pack_forget()  # Скрываем экран входа
        difficulty_frame.pack()  # Показываем экран выбора сложности
    else:
        messagebox.showerror("Ошибка", "Неправильный пароль.")

# Классы игры
class Card:
    def __init__(self, master, number, game, bg_color="#90CAF9"):
        self.master = master
        self.number = number
        self.game = game  # Добавляем параметр game
        self.button = tk.Button(master, text="", width=8, height=2, bg=bg_color, fg="black", font=("Helvetica", 24, "bold"), command=self.reveal)
        self.is_revealed = False

    def reveal(self):
        if not self.game.game_running or self.is_revealed or len(self.game.revealed_cards) == 2:
            return
        self.is_revealed = True
        self.button.config(text=str(self.number), bg="#81C784")
        self.game.reveal_card(self)

    def hide(self):
        self.is_revealed = False
        self.button.config(text="", bg="#90CAF9")

    def disable(self):
        self.button.config(state=tk.DISABLED)

class Game:
    def __init__(self, master, num_pairs=8, bg_color="#90CAF9"):
        self.master = master
        self.num_pairs = num_pairs
        self.cards = []
        self.revealed_cards = []
        self.game_running = False
        self.time_left = 30
        self.moves_left = 15  # Количество ходов
        self.master.configure(bg=bg_color)  # Устанавливаем фон для игрового экрана
        self.create_widgets()

    def create_cards(self):
        numbers = list(range(1, self.num_pairs + 1)) * 2
        random.shuffle(numbers)
        self.cards = [Card(self.master, num, self) for num in numbers]  # Передаем self (игру) в каждую карту
        for i, card in enumerate(self.cards):
            card.button.grid(row=i // 4 + 2, column=i % 4, padx=5, pady=5)

    def create_widgets(self):
        self.start_button = tk.Button(self.master, text="ЗАПУСК", command=self.start, bg="#FF7043", fg="white", font=("Arial", 14, "bold"))
        self.start_button.grid(row=0, column=0, columnspan=2, pady=10)

        self.reset_button = tk.Button(self.master, text="ИГРАТЬ СНОВА", command=self.reset_game, bg="#66BB6A", fg="white", font=("Arial", 14, "bold"))
        self.reset_button.grid(row=0, column=2, columnspan=2, pady=10)
        self.reset_button.config(state=tk.DISABLED)

        self.timer_label = tk.Label(self.master, text="ВРЕМЯ: 30 СЕКУНД", font=("Arial", 14), fg="#4CAF50")
        self.timer_label.grid(row=1, column=0, columnspan=2, pady=5)

        self.moves_label = tk.Label(self.master, text=f"КОЛИЧЕСТВО ХОДОВ: {self.moves_left}", font=("Arial", 14), fg="#FFEB3B")
        self.moves_label.grid(row=1, column=2, columnspan=2, pady=5)

        # Добавляем кнопку для выхода в главное меню
        self.main_menu_button = tk.Button(self.master, text="Главное меню", command=self.return_to_difficulty_menu, bg="#3F51B5", fg="white", font=("Arial", 14, "bold"))
        self.main_menu_button.grid(row=0, column=4, padx=10, pady=10)

    def start(self):
        self.game_running = True
        self.time_left = 30
        self.moves_left = 15
        self.start_button.config(state=tk.DISABLED)
        self.reset_button.config(state=tk.NORMAL)
        self.update_timer()
        self.update_moves()
        self.create_cards()

    def reveal_card(self, card):
        self.revealed_cards.append(card)
        if len(self.revealed_cards) == 2:
            self.master.after(500, self.check_match)

    def check_match(self):
        card1, card2 = self.revealed_cards
        if card1.number == card2.number:
            card1.disable()
            card2.disable()
            self.check_win()
        else:
            self.moves_left -= 1  # Уменьшаем количество ходов только при ошибке
            self.update_moves()
            card1.hide()
            card2.hide()
            if self.moves_left <= 0:  # Проверка на окончание попыток
                self.end_game("Game Over", "Ходы кончились. Ты проиграл ( ´ ω  )")
        self.revealed_cards.clear()

    def update_moves(self):
        self.moves_label.config(text=f"КОЛИЧЕСТВО ХОДОВ: {self.moves_left}")

    def check_win(self):
        if all(card.button["state"] == tk.DISABLED for card in self.cards):
            self.game_running = False
            messagebox.showinfo("Поздравляем!", "Ты молодец, я тобой горжусь ٩(｡•́‿•̀｡)۶!")
            self.start_button.config(state=tk.DISABLED)

    def update_timer(self):
        if self.game_running:
            self.time_left -= 1
            self.timer_label.config(text=f"ВРЕМЯ: {self.time_left} СЕКУНД")
            if self.time_left > 0:
                self.master.after(1000, self.update_timer)
            else:
                self.end_game("Time's up", "Время вышло! Ты проиграл (´；д；`)")
                
    def end_game(self, title, message):
        self.game_running = False
        messagebox.showinfo(title, message)
        self.start_button.config(state=tk.NORMAL)

    def reset_game(self):
        self.game_running = False
        for card in self.cards:
            card.hide()
            card.button.config(state=tk.NORMAL)
        self.cards.clear()
        self.create_widgets()
        self.start_button.config(state=tk.NORMAL)

    def return_to_difficulty_menu(self):
        game_frame.pack_forget()
        difficulty_frame.pack()

# Функция, запускающая игру с выбранными параметрами
def start_game(num_pairs, bg_color):
    game_frame.pack_forget()  # Скрываем экран игры
    game = Game(game_frame, num_pairs=num_pairs, bg_color=bg_color)  # Создаем объект игры
    game_frame.pack()  # Показываем экран игры
    game.start()

# Главный интерфейс
root = tk.Tk()
root.title("Найди пару")
root.geometry("800x600")
root.configure(bg="#F5F5F5")

# Экран входа/регистрации
login_frame = tk.Frame(root, bg="#F5F5F5")
login_frame.pack()

name_label = tk.Label(login_frame, text="Имя пользователя:", font=("Arial", 14), bg="#F5F5F5")
name_label.grid(row=0, column=0, pady=5)

name_entry = tk.Entry(login_frame, font=("Arial", 14))
name_entry.grid(row=0, column=1, pady=5)

password_label = tk.Label(login_frame, text="Пароль:", font=("Arial", 14), bg="#F5F5F5")
password_label.grid(row=1, column=0, pady=5)

password_entry = tk.Entry(login_frame, show="*", font=("Arial", 14))
password_entry.grid(row=1, column=1, pady=5)

register_button = tk.Button(login_frame, text="Зарегистрировать", command=add_user, bg="#4CAF50", fg="white", font=("Arial", 14, "bold"))
register_button.grid(row=2, column=0, columnspan=2, pady=5)

login_button = tk.Button(login_frame, text="Войти", command=login, bg="#2196F3", fg="white", font=("Arial", 14, "bold"))
login_button.grid(row=3, column=0, columnspan=2, pady=5)

listbox = tk.Listbox(login_frame, font=("Arial", 14), height=10)
listbox.grid(row=4, column=0, columnspan=2, pady=10)

# Экран выбора сложности
difficulty_frame = tk.Frame(root, bg="#F5F5F5")

easy_button = tk.Button(difficulty_frame, text="Легкий", command=lambda: start_game(8, "#A5D6A7"), bg="#66BB6A", font=("Arial", 14, "bold"))
easy_button.grid(row=0, column=0, pady=10)

medium_button = tk.Button(difficulty_frame, text="Средний", command=lambda: start_game(12, "#FFCC80"), bg="#FF7043", font=("Arial", 14, "bold"))
medium_button.grid(row=0, column=1, pady=10)

hard_button = tk.Button(difficulty_frame, text="Трудный", command=lambda: start_game(16, "#FFCDD2"), bg="#F44336", font=("Arial", 14, "bold"))
hard_button.grid(row=0, column=2, pady=10)

# Экран игры
game_frame = tk.Frame(root)

# Загрузка пользователей
USERS = load_users()

# Обновление списка пользователей
update_listbox()

root.mainloop()
