import os
from telegram.ext import Application, CommandHandler, ContextTypes, MessageHandler, filters, PreCheckoutQueryHandler
from telegram import InputMediaPhoto, LabeledPrice
import logging
from telegram import Update, ReplyKeyboardMarkup
from telegram.ext import Application, CommandHandler, ContextTypes, MessageHandler, filters
from apscheduler.schedulers.background import BackgroundScheduler
from apscheduler.triggers.cron import CronTrigger
import sqlite3
import random
import datetime
import re
import os
import asyncio
import pytz

# Настройка логирования
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)

# Конфигурация
BOT_TOKEN = os.environ.get('BOT_TOKEN')
PAYMENT_PROVIDER_TOKEN = "390540012:LIVE:83099"  # ЮКасса через Telegram Payments

# Бесплатный аккаунт
FREE_ACCOUNT_ID = 837222801  # @istolik

# Московский часовой пояс
MOSCOW_TZ = pytz.timezone('Europe/Moscow')

# Глобальная переменная для приложения
application_instance = None

# База данных
class Database:
    def __init__(self):
        self.conn = sqlite3.connect('science_bot.db', check_same_thread=False)
        self.create_tables()

    def create_tables(self):
        cursor = self.conn.cursor()

        # Таблица пользователей
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS users (
                user_id INTEGER PRIMARY KEY,
                username TEXT,
                first_name TEXT,
                last_name TEXT,
                notification_time TEXT DEFAULT '18:00',
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                last_activity TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        ''')

        # Таблица отправленных заданий
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS sent_tasks (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER,
                task_id INTEGER,
                subject TEXT,
                category TEXT,
                sent_date DATE,
                is_correct BOOLEAN
            )
        ''')

        # Таблица статистики
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS user_stats (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER,
                subject TEXT,
                category TEXT,
                tasks_completed INTEGER DEFAULT 0,
                correct_answers INTEGER DEFAULT 0,
                last_active DATE
            )
        ''')

        # Таблица для отслеживания отправленных уведомлений
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS sent_notifications (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER,
                notification_date DATE,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        ''')

        # Таблица подписок
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS subscriptions (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER,
                plan_type TEXT,
                price INTEGER,
                currency TEXT DEFAULT 'RUB',
                start_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                end_date TIMESTAMP,
                is_active BOOLEAN DEFAULT 1,
                payment_charge_id TEXT,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        ''')

        self.conn.commit()

    def add_user(self, user_id, username, first_name, last_name):
        cursor = self.conn.cursor()
        cursor.execute('''
            INSERT OR REPLACE INTO users (user_id, username, first_name, last_name, last_activity)
            VALUES (?, ?, ?, ?, CURRENT_TIMESTAMP)
        ''', (user_id, username, first_name, last_name))
        self.conn.commit()

    def update_user_activity(self, user_id):
        cursor = self.conn.cursor()
        cursor.execute('UPDATE users SET last_activity = CURRENT_TIMESTAMP WHERE user_id = ?', (user_id,))
        self.conn.commit()

    def set_notification_time(self, user_id, time_str):
        cursor = self.conn.cursor()
        cursor.execute('UPDATE users SET notification_time = ? WHERE user_id = ?', (time_str, user_id))
        self.conn.commit()

    def get_notification_time(self, user_id):
        cursor = self.conn.cursor()
        cursor.execute('SELECT notification_time FROM users WHERE user_id = ?', (user_id,))
        result = cursor.fetchone()
        return result[0] if result else '18:00'

    def get_today_completed_tasks(self, user_id):
        today = datetime.date.today()
        cursor = self.conn.cursor()
        cursor.execute('SELECT COUNT(*) FROM sent_tasks WHERE user_id = ? AND sent_date = ?', (user_id, today))
        result = cursor.fetchone()
        return result[0] if result else 0

    def get_all_completed_tasks(self, user_id):
        cursor = self.conn.cursor()
        cursor.execute('SELECT task_id FROM sent_tasks WHERE user_id = ?', (user_id,))
        return [row[0] for row in cursor.fetchall()]

    def get_incorrect_tasks(self, user_id, subject):
        cursor = self.conn.cursor()
        cursor.execute('''
            SELECT task_id FROM sent_tasks
            WHERE user_id = ? AND subject = ? AND is_correct = 0
        ''', (user_id, subject))
        return [row[0] for row in cursor.fetchall()]

    def get_today_tasks_by_category(self, user_id, subject, category):
        today = datetime.date.today()
        cursor = self.conn.cursor()
        cursor.execute('''
            SELECT task_id FROM sent_tasks
            WHERE user_id = ? AND subject = ? AND category = ? AND sent_date = ?
        ''', (user_id, subject, category, today))
        return [row[0] for row in cursor.fetchall()]

    def mark_task_sent(self, user_id, task_id, subject, category, is_correct):
        today = datetime.date.today()
        cursor = self.conn.cursor()

        # Обновляем активность пользователя
        self.update_user_activity(user_id)

        # Сохраняем отправленное задание
        cursor.execute('''
            INSERT INTO sent_tasks (user_id, task_id, subject, category, sent_date, is_correct)
            VALUES (?, ?, ?, ?, ?, ?)
        ''', (user_id, task_id, subject, category, today, is_correct))

        # Обновляем статистику
        cursor.execute('''
            SELECT tasks_completed, correct_answers FROM user_stats
            WHERE user_id = ? AND subject = ? AND category = ?
        ''', (user_id, subject, category))

        result = cursor.fetchone()

        if result:
            # Обновляем существующую запись
            tasks_completed = result[0] + 1
            correct_answers = result[1] + (1 if is_correct else 0)
            cursor.execute('''
                UPDATE user_stats
                SET tasks_completed = ?, correct_answers = ?, last_active = ?
                WHERE user_id = ? AND subject = ? AND category = ?
            ''', (tasks_completed, correct_answers, today, user_id, subject, category))
        else:
            # Создаем новую запись
            cursor.execute('''
                INSERT INTO user_stats (user_id, subject, category, tasks_completed, correct_answers, last_active)
                VALUES (?, ?, ?, 1, ?, ?)
            ''', (user_id, subject, category, 1 if is_correct else 0, today))

        self.conn.commit()

    def get_user_stats_by_category(self, user_id, subject, category):
        cursor = self.conn.cursor()
        cursor.execute('''
            SELECT tasks_completed, correct_answers FROM user_stats
            WHERE user_id = ? AND subject = ? AND category = ?
        ''', (user_id, subject, category))
        result = cursor.fetchone()
        if result:
            return result[0], result[1]
        return 0, 0

    def get_user_stats_by_subject(self, user_id, subject):
        cursor = self.conn.cursor()
        cursor.execute('''
            SELECT SUM(tasks_completed), SUM(correct_answers) FROM user_stats
            WHERE user_id = ? AND subject = ?
        ''', (user_id, subject))
        result = cursor.fetchone()
        if result and result[0] is not None:
            return result[0], result[1] or 0
        return 0, 0

    def get_all_users_stats(self):
        cursor = self.conn.cursor()

        # Получаем всех пользователей с их активностью
        cursor.execute('''
            SELECT u.user_id, u.username, u.first_name, u.last_name, u.last_activity,
                   (SELECT COUNT(*) FROM sent_tasks WHERE user_id = u.user_id) as total_tasks,
                   (SELECT COUNT(*) FROM sent_tasks WHERE user_id = u.user_id AND is_correct = 1) as correct_tasks
            FROM users u
            ORDER BY u.last_activity DESC
        ''')

        users = cursor.fetchall()

        # Получаем общую статистику по предметам для каждого пользователя
        result = []
        for user in users:
            user_id, username, first_name, last_name, last_activity, total_tasks, correct_tasks = user

            # Статистика по химии
            chem_total, chem_correct = self.get_user_stats_by_subject(user_id, 'chemistry')

            # Статистика по биологии
            bio_total, bio_correct = self.get_user_stats_by_subject(user_id, 'biology')

            # Проверяем премиум статус
            has_premium = has_premium_access(user_id)
            premium_status = "💎 Премиум" if has_premium else "🎯 Бесплатный"

            result.append({
                'user_id': user_id,
                'username': username,
                'first_name': first_name,
                'last_name': last_name,
                'last_activity': last_activity,
                'total_tasks': total_tasks,
                'correct_tasks': correct_tasks,
                'chemistry_total': chem_total,
                'chemistry_correct': chem_correct,
                'biology_total': bio_total,
                'biology_correct': bio_correct,
                'premium_status': premium_status
            })

        return result

    def get_users_for_notification(self):
        cursor = self.conn.cursor()
        cursor.execute('SELECT user_id, notification_time FROM users')
        return cursor.fetchall()

    def update_task_correctness(self, user_id, task_id, is_correct):
        """Обновляет правильность ответа на задание"""
        cursor = self.conn.cursor()
        cursor.execute('''
            UPDATE sent_tasks
            SET is_correct = ?
            WHERE user_id = ? AND task_id = ?
        ''', (is_correct, user_id, task_id))
        self.conn.commit()

    def has_received_notification_today(self, user_id):
        """Проверяет, получал ли пользователь уведомление сегодня"""
        today = datetime.date.today()
        cursor = self.conn.cursor()
        cursor.execute('''
            SELECT COUNT(*) FROM sent_notifications
            WHERE user_id = ? AND notification_date = ?
        ''', (user_id, today))
        result = cursor.fetchone()
        return result[0] > 0 if result else False

    def mark_notification_sent(self, user_id):
        """Отмечает, что уведомление было отправлено пользователю сегодня"""
        today = datetime.date.today()
        cursor = self.conn.cursor()
        cursor.execute('''
            INSERT INTO sent_notifications (user_id, notification_date)
            VALUES (?, ?)
        ''', (user_id, today))
        self.conn.commit()

    # Методы для работы с подписками
    def add_subscription(self, user_id, plan_type, price, currency, end_date, payment_charge_id):
        """Добавляет подписку"""
        cursor = self.conn.cursor()
        cursor.execute('''
            INSERT INTO subscriptions (user_id, plan_type, price, currency, end_date, payment_charge_id)
            VALUES (?, ?, ?, ?, ?, ?)
        ''', (user_id, plan_type, price, currency, end_date, payment_charge_id))
        self.conn.commit()

    def get_active_subscription(self, user_id):
        """Получает активную подписку пользователя"""
        cursor = self.conn.cursor()
        cursor.execute('''
            SELECT * FROM subscriptions
            WHERE user_id = ? AND is_active = 1 AND end_date > CURRENT_TIMESTAMP
            ORDER BY created_at DESC LIMIT 1
        ''', (user_id,))
        result = cursor.fetchone()
        if result:
            return {
                'id': result[0],
                'user_id': result[1],
                'plan_type': result[2],
                'price': result[3],
                'currency': result[4],
                'start_date': result[5],
                'end_date': result[6],
                'is_active': result[7],
                'payment_charge_id': result[8],
                'created_at': result[9]
            }
        return None

    def deactivate_subscription(self, user_id):
        """Деактивирует подписку пользователя"""
        cursor = self.conn.cursor()
        cursor.execute('''
            UPDATE subscriptions SET is_active = 0 WHERE user_id = ? AND is_active = 1
        ''', (user_id,))
        self.conn.commit()

# Менеджер заданий по химии
class ChemistryTaskManager:
    def __init__(self):
        self.categories = {
            'Неорганическая химия': [
                {
                    'id': 213,
                    'question': '🧪 Неорганическая химия:\n\nИз указанных в ряду химических элементов выберите три элемента, которые в Периодической системе химических элементов Д. И. Менделеева находятся в одном периоде. Расположите выбранные элементы в порядке уменьшения числа валентных электронов.\n\n1) F 2) Li 3) Cl 4) O 5) As',
                    'options': ['1) 142', '2) 341', '3) 245', '4) 315'],
                    'answer': 0,
                    'explanation': '✅ Правильный ответ: 142\nВ одном периоде находятся фтор, литий и кислород. У элементов главных подгрупп валентными называют электроны, которые находятся на внешнем энергетическом уровне: F (2s²2p⁵), O (2s²2р⁴), Li (2s¹). В порядке уменьшения: F, O, Li.'
                },
                {
                    'id': 214,
                    'question': '🧪 Неорганическая химия:\n\nИз указанных в ряду химических элементов выберите три элемента, которые в Периодической системе химических элементов Д.И. Менделеева находятся в одном периоде. Расположите выбранные элементы в порядке уменьшения атомного радиуса.\n\n1) Se 2) Li 3) Cu 4) As 5) S',
                    'options': ['1) 142', '2) 341', '3) 245', '4) 315'],
                    'answer': 1,
                    'explanation': '✅ Правильный ответ: 341\nВ одном периоде находятся медь, селен и мышьяк. Радиус атома уменьшается при движении по периоду слева направо: Li, As, Se.'
                },
                {
                    'id': 215,
                    'question': '🧪 Неорганическая химия:\n\nИз указанных в ряду химических элементов выберите три элемента, которые в Периодической системе химических элементов Д.И. Менделеева находятся в одной группе. Расположите выбранные элементы в порядке увеличения их атомного радиуса.\n\n1) B 2) C 3) O 4) Si 5) Sn',
                    'options': ['1) 142', '2) 341', '3) 245', '4) 315'],
                    'answer': 2,
                    'explanation': '✅ Правильный ответ: 245\nВ одной группе находятся углерод, кремний и олово. Радиус атома увеличивается при движении по группе сверху вниз: C, Si, Sn.'
                },
                {
                    'id': 216,
                    'question': '🧪 Неорганическая химия:\n\nИз указанных в ряду химических элементов выберите три элемента-неметалла. Расположите выбранные элементы в порядке уменьшения радиусов их атомов.\n\n1) Be 2) P 3) O 4) Li 5) N',
                    'options': ['1) 142', '2) 253', '3) 341', '4) 315'],
                    'answer': 1,
                    'explanation': '✅ Правильный ответ: 253\nНеметаллами являются фосфор, азот и кислород. Радиус атома уменьшается при движении по группе снизу вверх, а по периоду слева направо: P, N, O.'
                },
                {
                    'id': 217,
                    'question': '🧪 Неорганическая химия:\n\nИз указанных в ряду химических элементов выберите три элемента, которые в Периодической системе химических элементов Д.И. Менделеева находятся в одной группе. Расположите выбранные элементы в порядке усиления металлических свойств образуемых ими простых веществ.\n\n1) As 2) O 3) N 4) S 5) Sb',
                    'options': ['1) 142', '2) 253', '3) 315', '4) 341'],
                    'answer': 2,
                    'explanation': '✅ Правильный ответ: 315\nВ одной группе находятся азот, мышьяк и сурьма. Металлические свойства усиливаются при движении по группе сверху вниз: N, As, Sb.'
                },
                {
                    'id': 201,
                    'question': '🧪 Неорганическая химия:\n\nКакой элемент имеет электронную конфигурацию 1s²2s²2p⁶3s²3p⁶4s¹?',
                    'options': ['1) Калий', '2) Натрий', '3) Кальций', '4) Аргон'],
                    'answer': 0,
                    'explanation': '✅ Правильный ответ: Калий\nЭлектронная конфигурация калия: 1s²2s²2p⁶3s²3p⁶4s¹'
                },
                {
                    'id': 202,
                    'question': '🧪 Неорганическая химия:\n\nКакой тип химической связи в молекуле хлорида натрия (NaCl)?',
                    'options': ['1) Ковалентная полярная', '2) Ионная', '3) Металлическая', '4) Ковалентная неполярная'],
                    'answer': 1,
                    'explanation': '✅ Правильный ответ: Ионная\nNaCl - типичный ионный кристалл'
                },
                {
                    'id': 203,
                    'question': '🧪 Неорганическая химия:\n\nКакой оксид является кислотным?',
                    'options': ['1) Na₂O', '2) CaO', '3) SO₂', '4) MgO'],
                    'answer': 2,
                    'explanation': '✅ Правильный ответ: SO₂\nSO₂ - оксид серы(IV), кислотный оксид'
                },
                {
                    'id': 204,
                    'question': '🧪 Неорганическая химия:\n\nКакой металл наиболее активен?',
                    'options': ['1) Медь', '2) Цинк', '3) Натрий', '4) Железо'],
                    'answer': 2,
                    'explanation': '✅ Правильный ответ: Натрий\nНатрий находится в начале ряда активности металлов'
                },
                {
                    'id': 205,
                    'question': '🧪 Неорганическая химия:\n\nКакой газ выделяется при реакции кислоты с карбонатом?',
                    'options': ['1) Кислород', '2) Водород', '3) Углекислый газ', '4) Азот'],
                    'answer': 2,
                    'explanation': '✅ Правильный ответ: Углекислый газ\nCaCO₃ + 2HCl → CaCl₂ + CO₂ + H₂O'
                },
            ],
            'Органическая химия': [
                {
                    'id': 218,
                    'question': '🧪 Органическая химия:\n\nУстановите соответствие между названием вещества и классом / группой органических соединений, к которому оно относится:\n\nА) стеариновая кислота\nБ) олеиновая кислота\nВ) анилин\n\n1) насыщенные жирные кислоты\n2) ненасыщенные жирные кислоты\n3) аминокислоты\n4) амины',
                    'options': ['1) 124', '2) 243', '3) 321', '4) 142'],
                    'answer': 0,
                    'explanation': '✅ Правильный ответ: 124\nА) стеариновая кислота (C₁₇H₃₅COOH) → насыщенные жирные кислоты (1)\nБ) олеиновая кислота (C₁₇H₃₃COOH) → ненасыщенные жирные кислоты (2)\nВ) анилин (C₆H₅NH₂) → амины (4)'
                },
                {
                    'id': 219,
                    'question': '🧪 Органическая химия:\n\nУстановите соответствие между общей формулой и названием вещества, составу которого соответствует эта формула:\n\nА) CnH2nO2\nБ) CnH2n+2O3\nВ) CnH2n+1NO2\n\n1) олеиновая кислота\n2) пальмитиновая кислота\n3) глицин\n4) глицерин',
                    'options': ['1) 124', '2) 243', '3) 321', '4) 142'],
                    'answer': 1,
                    'explanation': '✅ Правильный ответ: 243\nА) CnH2nO2 → пальмитиновая кислота (C₁₆H₃₂O₂) (2)\nБ) CnH2n+2O3 → глицерин (C₃H₈O₃) (4)\nВ) CnH2n+1NO2 → глицин (NH₂-CH₂-COOH) (3)'
                },
                {
                    'id': 220,
                    'question': '🧪 Органическая химия:\n\nУстановите соответствие между общей формулой и названием вещества, составу которого соответствует эта формула:\n\nА) CnH2nO2\nБ) CnH2n-2O2\nВ) CnH2n-2O4\n\n1) этандиовая кислота\n2) пропеновая кислота\n3) бутановая кислота\n4) бензойная кислота',
                    'options': ['1) 124', '2) 243', '3) 321', '4) 142'],
                    'answer': 2,
                    'explanation': '✅ Правильный ответ: 321\nА) CnH2nO2 → бутановая кислота (CH₃-CH₂-CH₂-COOH) (3)\nБ) CnH2n-2O2 → пропеновая кислота (CH₂=CH-COOH) (2)\nВ) CnH2n-2O4 → этандиовая кислота (HOOC-COOH) (1)'
                },
                {
                    'id': 206,
                    'question': '🧪 Органическая химия:\n\nКакой класс соединений представляет уксусная кислота?',
                    'options': ['1) Карбоновые кислоты', '2) Спирты', '3) Альдегиды', '4) Кетоны'],
                    'answer': 0,
                    'explanation': '✅ Правильный ответ: Карбоновые кислоты\nУксусная кислота CH₃COOH - карбоновая кислота'
                },
                {
                    'id': 207,
                    'question': '🧪 Органическая химия:\n\nКакая формула соответствует этилену?',
                    'options': ['1) CH₄', '2) C₂H₄', '3) C₂H₆', '4) C₆H₆'],
                    'answer': 1,
                    'explanation': '✅ Правильный ответ: C₂H₄\nЭтилен - непредельный углеводород с двойной связью'
                },
            ],
            'Задачи': [
                {
                    'id': 211,
                    'question': '🧪 Задача:\n\nКакой объем водорода выделится при взаимодействии 2,3 г натрия с водой?',
                    'options': ['1) 1,12 л', '2) 2,24 л', '3) 0,56 л', '4) 4,48 л'],
                    'answer': 0,
                    'explanation': '✅ Правильный ответ: 1,12 л\n2Na + 2H₂O → 2NaOH + H₂\nn(Na)=0,1 моль, V(H₂)=1,12 л'
                },
                {
                    'id': 212,
                    'question': '🧪 Задача:\n\nКакая масса кислорода потребуется для полного сгорания 16 г метана?',
                    'options': ['1) 32 г', '2) 64 г', '3) 48 г', '4) 16 г'],
                    'answer': 1,
                    'explanation': '✅ Правильный ответ: 64 г\nCH₄ + 2O₂ → CO₂ + 2H₂O\nn(CH₄)=1 моль, m(O₂)=64 г'
                },
            ]
        }

    def get_random_task(self, user_id, category=None, error_work=False):
        db = Database()

        if category == 'Солянка заданий по химии' or category == 'Работа над ошибками':
            # Собираем все задачи из всех категорий ХИМИИ
            all_tasks = []
            for cat_tasks in self.categories.values():
                all_tasks.extend(cat_tasks)
        else:
            all_tasks = self.categories.get(category, [])

        if not all_tasks:
            return None

        if error_work:
            # Для работы над ошибками берем только те задания, где пользователь ошибался по ХИМИИ
            incorrect_tasks = db.get_incorrect_tasks(user_id, 'chemistry')
            available_tasks = [task for task in all_tasks if task['id'] in incorrect_tasks]
        else:
            # Для обычного режима исключаем все когда-либо решенные задания по ХИМИИ
            all_completed_tasks = db.get_all_completed_tasks(user_id)
            available_tasks = [task for task in all_tasks if task['id'] not in all_completed_tasks]

        if not available_tasks:
            return None

        # Выбираем случайное задание
        selected_task = random.choice(available_tasks)

        # Определяем категорию для выбранной задачи
        task_category = category
        if category in ['Солянка заданий', 'Работа над ошибками']:
            for cat_name, tasks in self.categories.items():
                if selected_task in tasks:
                    task_category = cat_name
                    break

        return selected_task, task_category

# Менеджер заданий по биологии
class BiologyTaskManager:
    def __init__(self):
        self.categories = {
            'Общая биология': [
                {
                    'id': 122,
                    'question': '🔬 Общая биология:\n\nЭкспериментатор поместил зерновки пшеницы в сушильный шкаф. Как при этом изменились концентрация солей и количество воды в клетках семян?\n\n1) увеличилась\n2) уменьшилась\n3) не изменилась',
                    'options': ['1) 12', '2) 21', '3) 33', '4) 22'],
                    'answer': 0,
                    'explanation': '✅ Правильный ответ: 12\nВ сушильном шкафу вода из зерновок испаряется, поэтому количество воды уменьшается (2). Так как количество воды уменьшается, концентрация солей увеличивается (1).'
                },
                {
                    'id': 123,
                    'question': '🔬 Общая биология:\n\nВ некоторой молекуле ДНК эукариотического организма на долю нуклеотидов с цитозином приходится 31%. Определите долю нуклеотидов с тимином, входящих в состав этой молекулы.',
                    'options': ['1) 19%', '2) 31%', '3) 38%', '4) 25%'],
                    'answer': 0,
                    'explanation': '✅ Правильный ответ: 19%\nЕсли на долю нуклеотидов с цитозином приходится 31%, то на долю нуклеотидов с тимином приходится 50%-31%=19%.'
                },
                {
                    'id': 124,
                    'question': '🔬 Общая биология:\n\nВ транскрибируемой цепи ДНК содержится 30% аденина и 20% тимина. Определите содержание гуанина во фрагменте двуцепочечной молекулы ДНК.',
                    'options': ['1) 19%', '2) 25%', '3) 30%', '4) 20%'],
                    'answer': 1,
                    'explanation': '✅ Правильный ответ: 25%\nВ смысловой цепи ДНК будет 30% тимина и 20% аденина. Тогда во фрагменте двуцепочечной молекулы ДНК 25% тимина и 25% аденина. Согласно правилу комплементарности: Г + Т = 50%, значит Г = 50% - Т = 50% - 25% = 25%.'
                },
                {
                    'id': 125,
                    'question': '🔬 Общая биология:\n\nИсследователь выделил фермент пероксидазу из клеток сои и определил ее активность. Затем в первую пробирку с пероксидазой он внес раствор соляной кислоты, а во вторую -- хлорида ртути (II). Как изменится активность фермента в обеих пробирках?\n\n1) увеличилась\n2) уменьшилась\n3) не изменилась',
                    'options': ['1) 12', '2) 22', '3) 33', '4) 11'],
                    'answer': 1,
                    'explanation': '✅ Правильный ответ: 22\nРастворы соляной кислоты и хлорида ртути (II) не являются оптимальной средой для работы фермента пероксидазы, поэтому его активность уменьшится в обоих случаях.'
                },
                {
                    'id': 126,
                    'question': '🔬 Общая биология:\n\nРассмотрите рисунок. Заполните пустые ячейки таблицы:\n\nА - углевод\nБ - строение\nВ - функции',
                    'options': ['1) 815', '2) 234', '3) 126', '4) 356'],
                    'answer': 0,
                    'photo_url': 'https://neofamily.ru/_next/image?url=https%3A%2F%2Fafb4a530-22b8-416e-b47b-cdbbbe63bf2f.selstorage.ru%2FmjRPN8NALwiQWY7YbygKAxHAtLlGxXKrNsSJJPy4.png&w=384&q=75',
                    'explanation': '✅ Правильный ответ: 815\nНа рисунке изображен гликоген - многоразветвленный полисахарид глюкозы.'
                },
                {
                    'id': 127,
                    'question': '🔬 Общая биология:\n\nУстановите последовательность процессов, происходящих при формировании нативной структуры белка.\n\n1) формирование водородных связей между пептидными группами аминокислот\n2) образование дисульфидных связей между различными участками белковой молекулы\n3) синтез полипептидной цепи из аминокислот\n4) присоединение фосфатной группы к белку с третичной структурой\n5) сборка нескольких полипептидных субъединиц в один белок',
                    'options': ['1) 31245', '2) 12345', '3) 32154', '4) 21345'],
                    'answer': 0,
                    'explanation': '✅ Правильный ответ: 31245\nСначала синтезируется полипептидная цепь (3), затем водородные связи (1), дисульфидные связи (2), присоединение фосфатной группы (4), сборка субъединиц (5).'
                },
                {
                    'id': 128,
                    'question': '🔬 Общая биология:\n\nВ некоторой молекуле РНК на долю нуклеотидов с урацилом приходится 13%. Определите долю нуклеотидов с аденином на матричной цепи молекулы ДНК.',
                    'options': ['1) 13%', '2) 25%', '3) 37%', '4) 19%'],
                    'answer': 0,
                    'explanation': '✅ Правильный ответ: 13%\nПри синтезе РНК урацил (У) в молекуле РНК всегда комплементарен аденину (A) в матричной цепи ДНК. Следовательно, если в РНК содержится 13% урацила, то в соответствующей матричной цепи ДНК будет 13% аденина.'
                },
                {
                    'id': 129,
                    'question': '🔬 Общая биология:\n\nУстановите соответствие между признаками и уровнями структурной организации белков, обозначенными на рисунке цифрами 1-4:\n\nА) содержит несколько полипептидных цепей\nБ) закодирована в нуклеиновых кислотах\nВ) стабилизируется только водородными связями\nГ) стабилизируется дисульфидными и гидрофобными связями\nД) представлена правозакрученной спиралью\nЕ) имеет форму глобулы из одной полипептидной цепи',
                    'options': ['1) 412323', '2) 124321', '3) 341232', '4) 234123'],
                    'answer': 0,
                    'photo_url': 'https://neofamily.ru/_next/image?url=https%3A%2F%2Fafb4a530-22b8-416e-b47b-cdbbbe63bf2f.selstorage.ru%2F3v0aFIt5JH13OFspNp5Zf0kcgLnWZiXgYlUp3gO8.png&w=1200&q=75',
                    'explanation': '✅ Правильный ответ: 412323\n1 - первичная, 2 - вторичная, 3 - третичная, 4 - четвертичная структура.'
                },
                {
                    'id': 130,
                    'question': '🔬 Общая биология:\n\nКаким номером на рисунке обозначен уровень структуры белка, который может быть представлен β-складчатым слоем?',
                    'options': ['1) 1', '2) 2', '3) 3', '4) 4'],
                    'answer': 1,
                    'photo_url': 'https://neofamily.ru/_next/image?url=https%3A%2F%2Fafb4a530-22b8-416e-b47b-cdbbbe63bf2f.selstorage.ru%2F3v0aFIt5JH13OFspNp5Zf0kcgLnWZiXgYlUp3gO8.png&w=1200&q=75',
                    'explanation': '✅ Правильный ответ: 2\nВторичная структура белка может быть представлена β-складчатым слоем.'
                },
                {
                    'id': 131,
                    'question': '🔬 Общая биология:\n\nКаким номером на рисунке обозначены связи, стабилизирующие структуру одной полинуклеотидной цепи?',
                    'options': ['1) 1', '2) 2', '3) 3', '4) 5'],
                    'answer': 3,
                    'photo_url': 'https://neofamily.ru/_next/image?url=https%3A%2F%2Fafb4a530-22b8-416e-b47b-cdbbbe63bf2f.selstorage.ru%2Ffiles%2Ft5Di5YznMfCPP976tXGgy5MH0U99bRvGV6Bl38eK.png&w=750&q=75',
                    'explanation': '✅ Правильный ответ: 5\nСвязи, стабилизирующие структуру одной полинуклеотидной цепи - ковалентные фосфодиэфирные связи, обозначены цифрой 5.'
                },
                {
                    'id': 101,
                    'question': '🔬 Общая биология:\n\nКакие функции выполняет АТФ в клетке?',
                    'options': ['1) Универсальный источник энергии', '2) Запасное вещество', '3) Строительный материал', '4) Фермент'],
                    'answer': 0,
                    'explanation': '✅ Правильный ответ: Универсальный источник энергии\nАТФ - аденозинтрифосфат, основной источник энергии в клетке'
                },
                {
                    'id': 102,
                    'question': '🔬 Общая биология:\n\nВ каких органеллах происходит синтез белка?',
                    'options': ['1) Митохондрии', '2) Рибосомы', '3) Комплекс Гольджи', '4) Лизосомы'],
                    'answer': 1,
                    'explanation': '✅ Правильный ответ: Рибосомы\nРибосомы - органеллы, отвечающие за синтез белка'
                },
                {
                    'id': 151,
                    'question': '🔬 Общая биология:\n\nВыберите три верно обозначенные подписи к рисунку. Запишите цифры, под которыми они указаны.\n\n1) наружная мембрана\n2) мембрана тилакоида\n3) зёрна крахмала\n4) строма\n5) матрикс\n6) кристы',
                    'options': ['1) 235', '2) 256', '3) 156', '4) 356'],
                    'answer': 2,
                    'photo_url': 'https://neofamily.ru/_next/image?url=https%3A%2F%2Fafb4a530-22b8-416e-b47b-cdbbbe63bf2f.selstorage.ru%2Fz2Qh0WDVrM6uY2LpxDkeqTUaXKnxsa5ekYLSp4gG.png&w=750&q=75',
                    'explanation': '✅ Правильный ответ: 156\nВерные варианты:1) наружная мембрана - внешняя оболочка митохондрии;5) матрикс - внутреннее содержимое митохондрии;6) кристы - складки внутренней мембраны, увеличивающие поверхность.'
                },
                {
                    'id': 152,
                    'question': '🔬 Общая биология:\n\nРассмотрите рисунок. Заполните пустые ячейки таблицы, используя элементы, приведённые в списке. Для каждой ячейки, обозначенной буквой, выберите соответствующий элемент из предложенного списка\n\nА-органоид\nБ-строение\nВ-химический состав\nСписок элементов:\n1) две центриоли, состоящие из микротрубочек \n2) актин и миозин\n3) веретено деления\n4) клеточный центр\n5) малая и большая субъединицы\n6) рибосома\n7) белки и рРНК\n8) тубулин',
                    'options': ['1) 418', '2) 657', '3) 132', '4) 432'],
                    'answer': 0,
                    'photo_url': 'https://neofamily.ru/_next/image?url=https%3A%2F%2Fafb4a530-22b8-416e-b47b-cdbbbe63bf2f.selstorage.ru%2FhgN59XxGDBCfyewEUtxQhcJBVVxhS6sqwwdX7a7d.png&w=384&q=75',
                    'explanation': '✅ Правильный ответ: 418\nА - 4; клеточный центр - органоид, участвующий в образовании веретена деления при клеточном делении; Б - 1; две центриоли, состоящие из микротрубочек - структурные компоненты клеточного центра, расположенные под прямым углом друг к другу; В - 8; тубулин - белок, из которого состоят микротрубочки центриолей клеточного центра.'
                },
                {
                    'id': 153,
                    'question': '🔬 Общая биология:\n\nНа рисунке с каким номером показана электронная микрофотография органоида, отсутствующего в клетках высших семенных растений?',
                    'options': ['1) 1', '2) 2', '3) 3'],
                    'answer': 0,
                    'photo_url': 'https://neofamily.ru/_next/image?url=https%3A%2F%2Fafb4a530-22b8-416e-b47b-cdbbbe63bf2f.selstorage.ru%2Ffiles%2F26NaKXHoIznzSsO4p71yvcgoMmLQ9R4HuSuAeRKR.png&w=1920&q=75',
                    'explanation': '✅ Правильный ответ: 1\n1 - центриоль, 2 - аппарат Гольджи, 3 - митохондрия. Из представленных органоид, отсутствующий в клетках высших семенных растений - клеточный центр (1).'
                },
                {
                    'id': 154,
                    'question': '🔬 Общая биология:\n\nКакой цифрой на рисунке показан органоид, содержащий крупные крахмальные зерна?',
                    'options': ['1) 1', '2) 2', '3) 3'],
                    'answer': 0,
                    'photo_url': 'https://neofamily.ru/_next/image?url=https%3A%2F%2Fafb4a530-22b8-416e-b47b-cdbbbe63bf2f.selstorage.ru%2Ffiles%2Fc32sDNrfheXWWXb1skRPolgmf57MVlmMT2Kmwrnn.png&w=828&q=75',
                    'explanation': '✅ Правильный ответ: 1\n1 - лейкопласт, 2 - хлоропласт, 3 - хромопласт.Лейкопласт (цифра 1) - это органоид, в котором накапливаются крупные крахмальные зёрна. Он служит для хранения запасных питательных веществ в растительных клетках.'
                },
                {
                    'id': 155,
                    'question': '🔬 Общая биология:\n\nКаким номером на рисунке обозначен структурный компонент, в котором содержится пигмент?',
                    'options': ['1) 1', '2) 2', '3) 3','4) 4', '5) 6', '6) 7'],
                    'answer': 1,
                    'photo_url': 'https://neofamily.ru/_next/image?url=https%3A%2F%2Fafb4a530-22b8-416e-b47b-cdbbbe63bf2f.selstorage.ru%2Ffiles%2Fk9IfiW95YbWnyfXSmyxJhKgSOrqQFoKiXWzXcEAg.png&w=750&q=75',
                    'explanation': '✅ Правильный ответ: 2\n1 - строма, 2 - грана, 3 - кольцевая ДНК, 4 - скопление крахмальных гранул, 5 - тилакоид, 6 - внешняя мембрана, 7 - внутреннаяя мембрана. Пигменты, такие как хлорофилл, содержатся в гранах (номер 2) и тилакоидах (номер 5) хлоропласта, где происходит светозависимая фаза фотосинтеза. Эти структуры обеспечивают поглощение света и преобразование его энергии.'
                },
                {
                    'id': 156,
                    'question': '🔬 Общая биология:\n\nКаким номером на рисунке обозначен структурный компонент клетки, участвующий в катаболизме?',
                    'options': ['1) 1', '2) 2', '3) 3','4) 4'],
                    'answer': 2,
                    'photo_url': 'https://neofamily.ru/_next/image?url=https%3A%2F%2Fafb4a530-22b8-416e-b47b-cdbbbe63bf2f.selstorage.ru%2Ffiles%2FSNiqwCtRNGRnDT8WGT5vGFIpATUxI3zrtFeD8BpN.png&w=750&q=75',
                    'explanation': '✅ Правильный ответ: 3\n1 - плазмолемма, 2 - аппарат Гольджи, 3 - митохондрия, 4 - хлоропласт. Митохондрия (номер 3) - это структурный компонент клетки, участвующий в катаболизме, так как в ней происходит клеточное дыхание, включающее расщепление органических веществ с выделением энергии в форме АТФ. Это ключевой процесс энергетического обмена.'
                },
                {
                    'id': 157,
                    'question': '🔬 Общая биология:\n\nВыберите три верных ответа из шести и запишите цифры, под которыми они указаны. Какие из перечисленных ниже признаков можно использовать для описания изображённой на рисунке структуры клетки?\n\n1) участвуют в процессе синтеза АТФ\n2) участвуют в процессе формирования веретена деления\n3) участвуют в процессе синтеза белка\n4) состоят из пучков микротрубочек\n5) состоят из белка и РНК\n6) немембранные органоиды',
                    'options': ['1) 123', '2) 136', '3) 356','4) 256'],
                    'answer': 2,
                    'photo_url': 'https://neofamily.ru/_next/image?url=https%3A%2F%2Fafb4a530-22b8-416e-b47b-cdbbbe63bf2f.selstorage.ru%2Ffiles%2FdYzCbj9cK9T1Z0wMRbmF2v0QWG3NZ4XOd3HxmRCj.png&w=256&q=75',
                    'explanation': '✅ Правильный ответ: 356\nВерные варианты (характеристики рибосом): 3) участвуют в процессе синтеза белка - рибосомы являются местом трансляции (синтеза белка на матрице мРНК); 5) состоят из белка и РНК - рибосомы представляют собой рибонуклеопротеиновые комплексы (малая и большая субъединицы); 6) немембранные органоиды - рибосомы не имеют мембранной структуры.'
                },
                {
                    'id': 158,
                    'question': '🔬 Общая биология:\n\nВыберите три верных ответа из шести и запишите цифры, под которыми они указаны. Какие функции в клетке выполняют хлоропласты?\n\n1) синтез липидов\n2) синтез углеводов\n3) образование нитей веретена деления\n4) расщепление органических веществ до мономеров\n5) синтез органических веществ из неорганических\n6) использование энергии солнечного света для синтеза органических веществ',
                    'options': ['1) 123', '2) 136', '3) 356','4) 256'],
                    'answer': 3,
                    'explanation': '✅ Правильный ответ: 256\nВерные варианты (функции хлоропластов): 2) синтез углеводов - в строме хлоропластов происходит цикл Кальвина, в результате которого образуются углеводы; 5) синтез органических веществ из неорганических - фотосинтез позволяет создавать органические соединения (глюкозу) из CO₂ и воды; 6) использование энергии солнечного света для синтеза органических веществ - световая фаза фотосинтеза преобразует световую энергию в химическую (АТФ и НАДФН).'
                },
                {
                    'id': 159,
                    'question': '🔬 Общая биология:\n\nКаким номером обозначен одномембранный клеточный органоид?',
                    'options': ['1) 1', '2) 2', '3) 3','4) 4'],
                    'answer': 1,
                    'photo_url': 'https://neofamily.ru/_next/image?url=https%3A%2F%2Fafb4a530-22b8-416e-b47b-cdbbbe63bf2f.selstorage.ru%2Ffiles%2Fno47ck67hQ2XmNokujJB7lywbFEBQdD70HEcdF56.png&w=1080&q=75',
                    'explanation': '✅ Правильный ответ: 2\n1 - хлоропласт, 2 - аппарат Гольджи, 3 - митохондрия, 4 - клеточный центр. Аппарат Гольджи (номер 2) - это одномембранный органоид, участвующий в модификации, сортировке и транспорте белков и липидов в клетке. Он состоит из стопки уплощенных мембранных цистерн и связан с эндоплазматической сетью.'
                },
                {
                    'id': 160,
                    'question': '🔬 Общая биология:\n\nВыберите три верных ответа из шести и запишите цифры, под которыми они указаны. Какие из приведённых характеристик относятся к изображённой на рисунке структуре?\n\n1) участвует в синтезе АТФ\n2) имеет слой гликокаликса\n3) состоит из целлюлозы\n4) ограничивает внутреннее содержимое клетки от внешней среды\n5) обладает избирательной проницаемостью\n6) двумембранный органоид клетки',
                    'options': ['1) 245', '2) 135', '3) 234','4) 134'],
                    'answer': 0,
                    'photo_url': 'https://neofamily.ru/_next/image?url=https%3A%2F%2Fafb4a530-22b8-416e-b47b-cdbbbe63bf2f.selstorage.ru%2Ffiles%2FUkVgrSJemW0GbnzCVSsDVxvPSiKtrjD7AwhLjaua.jpg&w=640&q=75',
                    'explanation': '✅ Правильный ответ: 245\nВерные варианты (характеристики плазмалеммы): 2) имеет слой гликокаликса - углеводный компонент плазматической мембраны, участвующий в узнавании клеток; 4) ограничивает внутреннее содержимое клетки от внешней среды - основная барьерная функция мембраны; 5) обладает избирательной проницаемостью - регулирует транспорт веществ в клетку и из нее.'
                },
            ],
            'Анатомия': [
                {
                    'id': 132,
                    'question': '🫀 Анатомия:\n\nВыберите три верно обозначенные подписи к рисунку скелета человека:\n\n1) локтевая кость\n2) лучевая кость\n3) плечевая кость\n4) крестец\n5) бедренная кость\n6) стопа',
                    'options': ['1) 135', '2) 246', '3) 356', '4) 124'],
                    'answer': 2,
                    'photo_url': 'https://neofamily.ru/_next/image?url=https%3A%2F%2Fafb4a530-22b8-416e-b47b-cdbbbe63bf2f.selstorage.ru%2FO6VY2MYuRNrW2yWaBivHTVl0etLLzfSMJ8Rd8x8O.png&w=384&q=75',
                    'explanation': '✅ Правильный ответ: 356\n3) плечевая кость - верно\n5) бедренная кость - верно\n6) стопа - верно\nОстальные подписи неверны.'
                },
                {
                    'id': 133,
                    'question': '🫀 Анатомия:\n\nЧто характерно для вен, в отличие от артерий?\n\n1) относительно тонкий мышечный слой\n2) наличие клапанов\n3) высокое кровяное давление\n4) быстрый ток крови\n5) разносят кровь к органам и тканям\n6) транспорт крови к сердцу',
                    'options': ['1) 123', '2) 126', '3) 456', '4) 234'],
                    'answer': 1,
                    'explanation': '✅ Правильный ответ: 126\n1) относительно тонкий мышечный слой\n2) наличие клапанов\n6) транспорт крови к сердцу\nОстальные признаки характерны для артерий.'
                },
                {
                    'id': 106,
                    'question': '🫀 Анатомия:\n\nВ каком отделе пищеварительной системы происходит всасывание основной массы воды?',
                    'options': ['1) Желудок', '2) Тонкий кишечник', '3) Толстый кишечник', '4) Ротовая полость'],
                    'answer': 2,
                    'explanation': '✅ Правильный ответ: Толстый кишечник\nЗдесь происходит всасывание воды и формирование каловых масс'
                },
                {
                    'id': 107,
                    'question': '🫀 Анатомия:\n\nСколько камер в сердце человека?',
                    'options': ['1) 2', '2) 3', '3) 4', '4) 5'],
                    'answer': 2,
                    'explanation': '✅ Правильный ответ: 4\nСердце человека имеет 4 камеры: 2 предсердия и 2 желудочка'
                },
            ],
            'Ботаника': [
                {
                    'id': 134,
                    'question': '🍃 Ботаника:\n\nВыберите три верно обозначенные подписи к рисунку строения корня:\n\n1) придаточный корень\n2) зона, образованная постоянно делящимися клетками верхушечной образовательной ткани\n3) зона растущих клеток с начальной дифференциацией\n4) зона проведения\n5) боковой корень\n6) структура, обеспечивающая всасывание воды',
                    'options': ['1) 123', '2) 234', '3) 345', '4) 456'],
                    'answer': 1,
                    'photo_url': 'https://neofamily.ru/_next/image?url=https%3A%2F%2Fafb4a530-22b8-416e-b47b-cdbbbe63bf2f.selstorage.ru%2FlCQE0BxpII1MJrWAgW9VbWMgdseUVzZXC0PI6XYU.png&w=640&q=75',
                    'explanation': '✅ Правильный ответ: 234\n2) зона деления (апикальная меристема)\n3) зона растяжения\n4) зона проведения\nОстальные подписи не соответствуют изображению.'
                },
                {
                    'id': 111,
                    'question': '🍃 Ботаника:\n\nВ каких органеллах клетки происходит фотосинтез?',
                    'options': ['1) Митохондрии', '2) Хлоропласты', '3) Рибосомы', '4) Комплекс Гольджи'],
                    'answer': 1,
                    'explanation': '✅ Правильный ответ: Хлоропласты\nХлоропласты содержат хлорофилл и фотосинтетические мембраны'
                },
                {
                    'id': 161,
                    'question': '🍃 Ботаника:\n\nВыберите три верных ответа из шести и запишите цифры, под которыми они указаны. Какие утверждения о реакции растения на водный режим являются верными?\n\n1) при повышении температуры с 20°C до 30°C интенсивность транспирации увеличивается\n2) при потере тургора устьица открываются\n3) растения степей поглощают водяной пар при открывании устьиц\n4) с уменьшением влажности почвы транспирация уменьшается\n5) чем меньше относительная влажность воздуха, тем выше интенсивность транспирации\n6) чем концентрированнее клеточный сок, тем сильнее транспирация',
                    'options': ['1) 145', '2) 245', '3) 234', '4) 123'],
                    'answer': 0,
                    'explanation': '✅ Правильный ответ: 145,Подходят варианты 1) при повышении температуры с 20°C до 30°C интенсивность транспирации увеличивается – рост температуры всегда усиливает испарение воды с поверхности листьев; 4) с уменьшением влажности почвы транспирация уменьшается – недостаток воды в почве приводит к закрыванию устьиц и снижению испарения; 5) чем меньше относительная влажность воздуха, тем выше интенсивность транспирации – сухой воздух увеличивает градиент влажности, ускоряя испарение.'
                },
                {
                    'id': 162,
                    'question': '🍃 Ботаника:\n\nВыберите три верных ответа из шести и запишите цифры, под которыми они указаны. Для клеток, обозначенных на рисунке цифрой 1, характерны следующие признаки:\n\n1) высокое содержание суберина\n2) имеют хлоропласты\n3) закрывают устьичную щель при снижении тургора\n4) у водных растений располагаются на верхней стороне листа\n5) формируют чечевички\n6) клеточная стенка равномерно утолщена',
                    'options': ['1) 145', '2) 245', '3) 234', '4) 123'],
                    'answer': 2,
                    'photo_url': 'https://neofamily.ru/_next/image?url=https%3A%2F%2Fafb4a530-22b8-416e-b47b-cdbbbe63bf2f.selstorage.ru%2Ffiles%2FiSSriVwvz6eFjrGaafR8Xv5ChFGsHQNKab7kZBqM.png&w=640&q=75',
                    'explanation': '✅ Правильный ответ: 234,Для замыкающих клеток устьиц, обозначенных на рисунке цифрой 1, характерны следующие признаки: 2) имеют хлоропласты; 3) закрывают устьичную щель при снижении тургора; 4) у водных растений располагаются на верхней стороне листа.'
                },
                {
                    'id': 163,
                    'question': '🍃 Ботаника:\n\nВыберите три верных ответа из шести и запишите цифры, под которыми они указаны. К тканям растений относятся:\n\n1) проводящая\n2) нервная\n3) эпителиальная\n4) покровная\n5) основная\n6) соединительная',
                    'options': ['1) 145', '2) 245', '3) 234', '4) 123'],
                    'answer': 0,
                    'explanation': '✅ Правильный ответ: 145,Ткань - это группа клеток, сходных по строению, происхождению и выполняемым функциям. Растительные ткани делят на несколько групп в зависимости от основной функции: образовательные, покровные, основные, механические, проводящие, секреторные (выделительные). '
                },
                {
                    'id': 164,
                    'question': '🍃 Ботаника:\n\nКакой цифрой на рисунке обозначена ткань растения, которая обеспечивает всасывание воды и минеральных солей из почвы?',
                    'options': ['1) 1', '2) 2', '3) 3', '4) 4','5) 5', '6) 6'],
                    'answer': 5,
                    'photo_url': 'https://neofamily.ru/_next/image?url=https%3A%2F%2Fafb4a530-22b8-416e-b47b-cdbbbe63bf2f.selstorage.ru%2Ffiles%2F08lc1WQgQGKIesCYz984LMcbkLI6UZPOPj6eYfax.png&w=1920&q=75',
                    'explanation': '✅ Правильный ответ: 6,1 - первичная покровная ткань (эпидерма, кожица) с замыкающими клетками устьиц (слева) и железистым волоском - трихомой (справа), 2 - столбчатый мезофилл листа, 3 - пробка (феллема), 4 - ситовидные трубки с клетками-спутницами, 5 - сосуды, 6 - ризодерма с корневым волоском.Всасывание воды и минеральных солей из почвы осуществляется корневыми волосками - выростами клеток ризодермы (6). '
                },
                {
                    'id': 165,
                    'question': '🍃 Ботаника:\n\nКаким номером на рисунке обозначено основание междоузлий?',
                    'options': ['1) 1', '2) 2', '3) 3'],
                    'answer': 2,
                    'photo_url': 'https://neofamily.ru/_next/image?url=https%3A%2F%2Fafb4a530-22b8-416e-b47b-cdbbbe63bf2f.selstorage.ru%2Ffiles%2FStrZZ4hbKiUaALpHyUBIqk2zTQtX2IYTsAEWsTG3.png&w=828&q=75',
                    'explanation': '✅ Правильный ответ: 3- Образовательные ткани (меристемы): 1 - верхушечная (апикальная), 2 - боковая (латеральная), 3 - вставочная (интеркалярная).Узел - это участок стебля, на котором развиваются боковые органы (листья, почки, побеги и т.д.); междоузлие - это участок стебля между двумя соседними узлами. '
                },
            ],
            'Зоология': [
                {
                    'id': 135,
                    'question': '🐢 Зоология:\n\nКаким номером на рисунке обозначена стадия жизненного цикла паразита, которая попадает в окончательного хозяина?',
                    'options': ['1) 1', '2) 3', '3) 6', '4) 7'],
                    'answer': 2,
                    'photo_url': 'https://neofamily.ru/_next/image?url=https%3A%2F%2Fafb4a530-22b8-416e-b47b-cdbbbe63bf2f.selstorage.ru%2FHK0Fm787XIdHY9hZXdPkpHTQKSCcxwHC9KsqctlG.png&w=1080&q=75',
                    'explanation': '✅ Правильный ответ: 6\nЦеркария (6) и адолескария (7) являются инвазионными стадиями сосальщиков, которые попадают в организм окончательного хозяина.'
                },
                {
                    'id': 136,
                    'question': '🐢 Зоология:\n\nУстановите соответствие между характеристиками и стадиями жизненного цикла паразита, обозначенными на рисунке цифрами 1, 2, 3:\n\nА) проникает в промежуточного хозяина\nБ) представляет собой личиночную стадию\nВ) является непосредственным результатом оплодотворения\nГ) развивается в печени основного хозяина\nД) активно плавает в воде\nЕ) имеет гермафродитную половую систему',
                    'options': ['1) 332131', '2) 123213', '3) 231321', '4) 312123'],
                    'answer': 0,
                    'photo_url': 'https://neofamily.ru/_next/image?url=https%3A%2F%2Fafb4a530-22b8-416e-b47b-cdbbbe63bf2f.selstorage.ru%2FHK0Fm787XIdHY9hZXdPkpHTQKSCcxwHC9KsqctlG.png&w=1080&q=75',
                    'explanation': '✅ Правильный ответ: 332131\n1 - взрослый сосальщик, 2 - яйцо, 3 - личинка с ресничками (мирацидий).'
                },
                {
                    'id': 137,
                    'question': '🐢 Зоология:\n\nКакие признаки характерны для дождевого червя?\n\n1) кислород поступает в организм через всю поверхность тела\n2) кишечник не дифференцирован на отделы\n3) кровеносная система относится к замкнутому типу\n4) нервная система относится к стволовому типу\n5) полость тела разделена перегородками\n6) промежутки между органами заполнены паренхимой',
                    'options': ['1) 123', '2) 135', '3) 246', '4) 456'],
                    'answer': 1,
                    'explanation': '✅ Правильный ответ: 135\n1) дышит всей поверхностью тела\n3) замкнутая кровеносная система\n5) сегментированная полость тела\nОстальные признаки не характерны.'
                },
                {
                    'id': 138,
                    'question': '🐢 Зоология:\n\nКакие признаки характерны для аскариды?\n\n1) поверхность тела покрыта кутикулой, защищающей червя от переваривания\n2) самец аскариды крупнее, чем самка\n3) нервная система стволового типа\n4) раздельнополый представитель кольчатых червей\n5) полость тела не разделена перегородками\n6) полость тела вторичная - целом',
                    'options': ['1) 123', '2) 135', '3) 246', '4) 456'],
                    'answer': 1,
                    'explanation': '✅ Правильный ответ: 135\n1) наличие кутикулы\n3) стволовая нервная система\n5) первичная полость тела\nОстальные признаки неверны.'
                },
                {
                    'id': 116,
                    'question': '🐢 Зоология:\n\nКакие клетки крови отвечают за специфический иммунный ответ?',
                    'options': ['1) Эритроциты', '2) Тромбоциты', '3) Лимфоциты', '4) Нейтрофилы'],
                    'answer': 2,
                    'explanation': '✅ Правильный ответ: Лимфоциты\nТ-лимфоциты и В-лимфоциты обеспечивают специфический иммунитет'
                },
                {
                    'id': 139,
                    'question': '🐢 Зоология:\n\nВыберите три верных ответа из шести и запишите цифры, под которыми они указаны. У организма, имеющего скелет, показанный на рисунке:\n\n1)безъядерные эритроциты\n2) двухкамерное сердце\n3) редуцирован тазовый пояс\n4) передние конечности — плавники\n5) жаберное дыхание\n6) наружное оплодотворение',
                    'options': ['1) 123', '2) 136', '3) 235', '4) 134'],
                    'answer': 3,
                    'photo_url': 'https://neofamily.ru/_next/image?url=https%3A%2F%2Fafb4a530-22b8-416e-b47b-cdbbbe63bf2f.selstorage.ru%2Ffiles%2FTTlSGJex7luS1QoNh4xxlMd2z5lUhGhIwdB3bAym.png&w=750&q=75',
                    'explanation': '✅ Правильный ответ: 134\nТХарактерные особенности китообразных как млекопитающих, приспособившихся к водному образу жизни:1) безъядерные эритроциты - у всех млекопитающих, включая китов, зрелые эритроциты не содержат ядер;3) редуцирован тазовый пояс - в ходе эволюции у китов произошла редукция задних конечностей и их пояса как приспособление к водному образу жизни;4) передние конечности - плавники - передние конечности китообразных видоизменились в ласты (плавники) для эффективного плавания.'
                },
                {
                    'id': 140,
                    'question': '🐢 Зоология:\n\nОпределите зародышевые оболочки, которые формируются только у амниот. Запишите цифры, под которыми они указаны:\n\n1) плацента\n2) амнион\n3) желточный мешок\n4) пуповина\n5) хорион\n6) аллантоис',
                    'options': ['1) 345', '2) 125', '3) 256', '4) 134'],
                    'answer': 2,
                    'explanation': '✅ Правильный ответ: 256\n;Амниоты (пресмыкающиеся, птицы и млекопитающие) характеризуются наличием особых зародышевых оболочек, обеспечивающих развитие эмбриона вне водной среды.2) амнион - образует заполненную жидкостью полость, защищающую эмбрион от механических повреждений;5) хорион - наружная оболочка, участвующая в газообмене и формировании плаценты у млекопитающих;6) аллантоис - выполняет функции выделения и газообмена, у млекопитающих участвует в образовании пуповины.'
                },
                {
                    'id': 141,
                    'question': '🐢 Зоология:\n\nКаким номером на рисунках обозначен организм, обитающий в пресной воде?',
                    'options': ['1) 1', '2) 2', '3) 3'],
                    'answer': 1,
                    'photo_url': 'https://neofamily.ru/_next/image?url=https%3A%2F%2Fafb4a530-22b8-416e-b47b-cdbbbe63bf2f.selstorage.ru%2Ffiles%2FJL2JBazgaF8IG3QU17X6D6ZALoqguH8HqdYK05yK.png&w=1080&q=75',
                    'explanation': ('✅ Правильный ответ: 2\n'
               'Кишечнополостные: 1 - коралл, 2 - гидра, 3 - медуза. '
               'Организм, обитающий в пресной воде - гидра. '
               'В отличие от медуз и кораллов, гидра способна обитать в пресной воде '
               'благодаря наличию более эффективных механизмов осморегуляции.')
                },
                {
                    'id': 142,
                    'question': '🐢 Зоология:\n\nУстановите соответствие между характеристиками и объектами, обозначенными на рисунках цифрами 1, 2, 3: к каждой позиции, данной в первом столбце, подберите соответствующую позицию из второго столбца.\n\nА)одиночный пресноводный полип, ведущий прикреплённый образ жизни\nБ)передвигается «кувырканием»\nВ) образуют атоллы\nГ) имеют вид зонтика\nД) колониальный организм с известковым скелетом\nЕ) передвижение реактивным способом',
                    'options': ['1) 221313', '2) 231312', '3) 122313','4) 222313'],
                    'answer': 0,
                    'photo_url': 'https://neofamily.ru/_next/image?url=https%3A%2F%2Fafb4a530-22b8-416e-b47b-cdbbbe63bf2f.selstorage.ru%2Ffiles%2FeWirQfIEmyZFqXWUxX3kuNwXBxtzpbu7RiwMP2Rf.png&w=1080&q=75',
                    'explanation': '✅ Правильный ответ: 221313\n;Кишечнополостные: 1 - коралл, 2 - гидра, 3 - медуза.'
                },
                {
                    'id': 143,
                    'question': '🐢 Зоология:\n\nУстановите последовательность систематических групп животных, начиная с самого высокого ранга. Запишите соответствующую последовательность цифр\n\n1) Волчьи (Псовые)\n2) Обыкновенная лисица\n3) Млекопитающие\n4) Хищные\nД5) Хордовые\n6) Лисица',
                    'options': ['1) 162534', '2) 351426', '3) 534162','4) 425163'],
                    'answer': 2,
                    'explanation': '✅ Правильный ответ: 534162\nТаксоны: тип-класс-отряд-семейство-род-вид'
                },
                {
                    'id': 144,
                    'question': '🐢 Зоология:\n\nКаким номером на рисунке обозначен представитель головоногих моллюсков?',
                    'options': ['1) 1', '2) 2', '3) 3'],
                    'answer': 1,
                    'photo_url':'https://neofamily.ru/_next/image?url=https%3A%2F%2Fafb4a530-22b8-416e-b47b-cdbbbe63bf2f.selstorage.ru%2Ffiles%2FkUdxIaUQMtMRH1DgSXaoCCjVBn5bOeIPvlljuc5B.png&w=1200&q=75',
                    'explanation': '✅ Правильный ответ: 2\nГоловоногий моллюск (номер 2) - это высокоорганизованный представитель типа, включающий осьминогов, кальмаров и каракатиц, обладающий развитой нервной системой и способностью к реактивному движению'
                },
                {
                    'id': 145,
                    'question': '🐢 Зоология:\n\nУстановите соответствие между характеристиками и объектами, обозначенными на рисунках цифрами 1, 2, 3: к каждой позиции, данной в первом столбце, подберите соответствующую позицию из второго столбца.\n\nА) питается путём фильтрации\nБ) реактивный способ передвижения\nВ) дыхание лёгочное\nГ) тело асимметричное\nД) голова отсутствует\nЕ) нога преобразована в щупальца',
                    'options': ['1) 321132', '2) 231213', '3) 123321', '4) 312123'],
                    'answer': 0,
                    'photo_url':'https://neofamily.ru/_next/image?url=https%3A%2F%2Fafb4a530-22b8-416e-b47b-cdbbbe63bf2f.selstorage.ru%2Ffiles%2FkUdxIaUQMtMRH1DgSXaoCCjVBn5bOeIPvlljuc5B.png&w=1200&q=75',
                    'explanation': '✅ Правильный ответ: 321132\n1 - брюхоногий моллюск, 2 - головоногий моллюск, 3 - двустворчатый моллюск.'
                },
                {
                    'id': 146,
                    'question': '🐢 Зоология:\n\nВыберите три верных ответа из шести и запишите цифры, под которыми они указаны. У насекомых с полным превращением:\n1) личинка превращается во взрослое насекомое\n2) личинка не походит на взрослое насекомое\n3) личинка походит на взрослое насекомое\n4) за стадией личинки следует стадия куколки\n5) наблюдаются четыре стадии развития\n6) наблюдаются три стадии развития',
                    'options': ['1) 235', '2) 245', '3) 145', '4) 123'],
                    'answer': 1,
                    'explanation': '✅ Правильный ответ: 245\nУ насекомых с полным превращением: 2) личинка не походит на взрослое насекомое; 4) за стадией личинки следует стадия куколки; 5) наблюдаются четыре стадии развития.'
                },
                {
                    'id': 147,
                    'question': '🐢 Зоология:\n\nУстановите последовательность систематических групп животных, начиная с самого низкого ранга. Запишите соответствующую последовательность цифр\n1) Двукрылые\n2) Животные\n3) Мухи\n4) Насекомые\n5) Комнатная муха\n6) Членистоногие',
                    'options': ['1) 462531', '2) 315264', '3) 246135', '4) 531462'],
                    'answer': 3,
                    'explanation': '✅ Правильный ответ: 531462\n5) Комнатная муха - вид 3) Мухи - род 1) Двукрылые - отряд 4) Насекомые - класс 6) Членистоногие - тип 2) Животные - царство'
                },
                {
                    'id': 148,
                    'question': '🐢 Зоология:\n\nКаким номером на рисунках обозначен представитель отряда Перепончатокрылые?',
                    'options': ['1) 1', '2) 2', '3) 3', '4) 4'],
                    'answer': 0,
                    'photo_url':'https://neofamily.ru/_next/image?url=https%3A%2F%2Fafb4a530-22b8-416e-b47b-cdbbbe63bf2f.selstorage.ru%2Ffiles%2F6urxosDdCttH58XzBF4ZXdbQ14XrPnDgrlnani7G.png&w=1200&q=75',
                    'explanation': '✅ Правильный ответ: 1\n1 - перепончатокрылые (пчела), 2 - чешуекрылые, или бабочки (бабочка), 3 - стрекозы (стрекоза), 4 - двукрылые (комар).'
                },
                {
                    'id': 149,
                    'question': '🐢 Зоология:\n\nУстановите соответствие между характеристиками и объектами, обозначенными на рисунках цифрами 1, 2, 3, 4: к каждой позиции, данной в первом столбце, подберите соответствующую позицию из второго столбца. Отряды насекомых, к которым относятся изображенные представители:\n\nА) ротовой аппарат колюще-сосущего типа\nБ) развитие с неполным превращением\nВ) личинка - гусеница\nГ) крылья покрыты разноцветными чешуйками\nД) две пары прозрачных крыльев, задняя пара меньше передней\nЕ) имеются жужжальца - видоизмененные задние крылья',
                    'options': ['1) 342124', '2) 432214', '3) 124324', '4) 243142'],
                    'answer': 1,
                    'photo_url':'https://neofamily.ru/_next/image?url=https%3A%2F%2Fafb4a530-22b8-416e-b47b-cdbbbe63bf2f.selstorage.ru%2Ffiles%2F6urxosDdCttH58XzBF4ZXdbQ14XrPnDgrlnani7G.png&w=1200&q=75',
                    'explanation': '✅ Правильный ответ: 1\n1 - перепончатокрылые (пчела), 2 - чешуекрылые, или бабочки (бабочка), 3 - стрекозы (стрекоза), 4 - двукрылые (комар).'
                },
                {
                    'id': 150,
                    'question': '🐢 Зоология:\n\nВыберите три верных ответа из шести и запишите цифры, под которыми они указаны. У изображённых на рисунке организмов:\n\n1) замкнутая кровеносная система\n2) тело имеет лучевую симметрию\n3) органы состоят из тканей\n4) тело состоит из двух слоёв клеток\n5) в наружном слое тела располагаются стрекательные клетки\n6) каждая клетка выполняет все функции живого организма',
                    'options': ['1) 245', '2) 123', '3) 345', '4) 145'],
                    'answer': 0,
                    'photo_url':'https://neofamily.ru/_next/image?url=https%3A%2F%2Fafb4a530-22b8-416e-b47b-cdbbbe63bf2f.selstorage.ru%2Ffiles%2Fg4toIH9n45TeSBPMQY63C88Zqv6f7hIEF7jI4X1E.png&w=1080&q=75',
                    'explanation': '✅ Правильный ответ: 245\nНа рисунке изображены представители разных классов типа Кишечнополостные (1 - гидроидные полипы, 2 - коралловые полипы, 3 - сцифоидные), для которых характерны признаки: 2) тело имеет лучевую симметрию; 4) тело состоит из двух слоёв клеток (эктодермы и энтодермы); 5) в наружном слое тела располагаются стрекательные клетки (в эктодерме).'
                },
            ],
            'Генетика': [
                {
                    'id': 121,
                    'question': '🧬 Генетика:\n\nПри скрещивании растений гороха с гладкими и морщинистыми семенами в F1 все потомство имело гладкие семена. Каков генотип родителей?',
                    'options': ['1) AA × aa', '2) Aa × Aa', '3) AA × Aa', '4) aa × aa'],
                    'answer': 0,
                    'explanation': '✅ Правильный ответ: AA × aa\nВ F1 все потомство единообразно - это признак анализирующего скрещивания'
                },
            ]
        }

    def get_random_task(self, user_id, category=None, error_work=False):
        db = Database()

        if category == 'Солянка заданий по биологии' or category == 'Работа над ошибками':
            # Собираем все задачи из всех категорий БИОЛОГИИ
            all_tasks = []
            for cat_tasks in self.categories.values():
                all_tasks.extend(cat_tasks)
        else:
            all_tasks = self.categories.get(category, [])

        if not all_tasks:
            return None

        if error_work:
            # Для работы над ошибками берем только те задания, где пользователь ошибался по БИОЛОГИИ
            incorrect_tasks = db.get_incorrect_tasks(user_id, 'biology')
            available_tasks = [task for task in all_tasks if task['id'] in incorrect_tasks]
        else:
            # Для обычного режима исключаем все когда-либо решенные задания по БИОЛОГИИ
            all_completed_tasks = db.get_all_completed_tasks(user_id)
            available_tasks = [task for task in all_tasks if task['id'] not in all_completed_tasks]

        if not available_tasks:
            return None

        selected_task = random.choice(available_tasks)
        task_category = category
        if category in ['Солянка заданий', 'Работа над ошибками']:
            for cat_name, tasks in self.categories.items():
                if selected_task in tasks:
                    task_category = cat_name
                    break

        return selected_task, task_category

# Инициализация
db = Database()
chemistry_manager = ChemistryTaskManager()
biology_manager = BiologyTaskManager()

# Функции для проверки подписки
def has_premium_access(user_id):
    """Проверяет, есть ли у пользователя премиум доступ"""
    # Бесплатный аккаунт
    if user_id == FREE_ACCOUNT_ID:
        return True

    subscription = db.get_active_subscription(user_id)
    if subscription:
        # Исправлено: правильное сравнение дат
        if isinstance(subscription['end_date'], str):
            try:
                end_date = datetime.datetime.strptime(subscription['end_date'], '%Y-%m-%d %H:%M:%S')
            except ValueError:
                try:
                    end_date = datetime.datetime.strptime(subscription['end_date'], '%Y-%m-%d %H:%M:%S.%f')
                except ValueError:
                    logging.error(f"Неизвестный формат даты: {subscription['end_date']}")
                    return False
        else:
            end_date = subscription['end_date']

        current_time = datetime.datetime.now()
        return end_date > current_time

    return False

async def check_premium_required(update: Update, context: ContextTypes.DEFAULT_TYPE, feature_name: str):
    """Проверяет премиум доступ и показывает сообщение если его нет"""
    user_id = update.effective_user.id
    if not has_premium_access(user_id):
        await update.message.reply_text(
            f"🚫 Функция '{feature_name}' доступна только с премиум подпиской!\n\n"
            "💎 Премиум подписка включает:\n"
            "✅ Полную статистику прогресса\n"
            "✅ Солянку заданий (все категории)\n"
            "✅ Работу над ошибками\n"
            "✅ Приоритетную поддержку\n\n"
            "💳 Выберите тариф:",
            reply_markup=get_premium_plans()
        )
        return False
    return True

# Клавиатуры
def get_main_menu():
    return ReplyKeyboardMarkup([
        ['🧪 Химия', '🧬 Биология'],
        ['📊 Статистика', '👤 Мой профиль'],
        ['🎯 Сегодняшний прогресс', '🔄 Работа над ошибками']
    ], resize_keyboard=True)

def get_chemistry_categories():
    return ReplyKeyboardMarkup([
        ['Неорганическая химия', 'Органическая химия'],
        ['Задачи', 'Солянка заданий по химии'],
        ['🔙 Главное меню']
    ], resize_keyboard=True)

def get_biology_categories():
    return ReplyKeyboardMarkup([
        ['Общая биология', 'Анатомия'],
        ['Ботаника', 'Зоология'],
        ['Генетика', 'Солянка заданий по биологии'],
        ['🔙 Главное меню']
    ], resize_keyboard=True)

def get_profile_menu():
    return ReplyKeyboardMarkup([
        ['🕐 Изменить время уведомлений', '💰 Премиум подписка'],
        ['🔙 Главное меню']
    ], resize_keyboard=True)

def get_back_menu():
    return ReplyKeyboardMarkup([
        ['🔙 Назад']
    ], resize_keyboard=True)

def get_continue_menu():
    return ReplyKeyboardMarkup([
        ['✅ Продолжить', '🔙 Выбрать другой раздел'],
        ['🔙 Главное меню']
    ], resize_keyboard=True)

def get_error_work_menu():
    return ReplyKeyboardMarkup([
        ['🧪 Ошибки по химии', '🧬 Ошибки по биологии'],
        ['🔙 Главное меню']
    ], resize_keyboard=True)

def get_premium_menu():
    return ReplyKeyboardMarkup([
        ['💎 Купить премиум', '📊 Моя подписка'],
        ['🔙 Главное меню']
    ], resize_keyboard=True)

def get_premium_plans():
    return ReplyKeyboardMarkup([
        ['📅 Месяц - 199₽', '📅 Год - 1910₽'],
        ['🔙 Назад']
    ], resize_keyboard=True)

def normalize_answer(answer):
    """Нормализует ответ пользователя для сравнения"""
    if not answer:
        return ""

    # Убираем лишние пробелы и приводим к нижнему регистру
    normalized = answer.strip().lower()

    # Убираем точку в конце, если есть
    if normalized.endswith('.'):
        normalized = normalized[:-1]

    return normalized

# Функции для работы с подписками
async def send_invoice(update: Update, context: ContextTypes.DEFAULT_TYPE, plan_type: str):
    """Отправляет счет для оплаты подписки"""
    prices = {
        'monthly': 19900,  # 199 рублей в копейках
        'yearly': 191000,  # 1910 рублей в копейках
    }

    titles = {
        'monthly': 'Премиум подписка на 1 месяц',
        'yearly': 'Премиум подписка на 1 год'
    }

    descriptions = {
        'monthly': '✅ Полная статистика прогресса\n✅ Солянка заданий (все категории)\n✅ Работа над ошибками\n✅ Приоритетная поддержка',
        'yearly': '✅ Все преимущества месячной подписки\n✅ Экономия 20%\n✅ Бесплатные обновления'
    }

    await context.bot.send_invoice(
        chat_id=update.effective_chat.id,
        title=titles[plan_type],
        description=descriptions[plan_type],
        payload=f"subscription_{plan_type}",
        provider_token=PAYMENT_PROVIDER_TOKEN,
        currency='RUB',
        prices=[LabeledPrice(titles[plan_type], prices[plan_type])],
        start_parameter=f"subscription_{plan_type}",
        need_name=False,
        need_phone_number=False,
        need_email=False,
        need_shipping_address=False,
        is_flexible=False,
        disable_notification=False,
        protect_content=False
    )

async def pre_checkout_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Обработчик предварительной проверки платежа"""
    query = update.pre_checkout_query
    await context.bot.answer_pre_checkout_query(
        pre_checkout_query_id=query.id,
        ok=True
    )

async def successful_payment_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Обработчик успешного платежа"""
    payment = update.message.successful_payment
    user_id = update.effective_user.id

    # Определяем тип подписки из payload
    plan_type = payment.invoice_payload.replace('subscription_', '')

    # Рассчитываем дату окончания подписки
    if plan_type == 'monthly':
        end_date = datetime.datetime.now() + datetime.timedelta(days=30)
    else:  # yearly
        end_date = datetime.datetime.now() + datetime.timedelta(days=365)

    # Сохраняем подписку в базу
    db.add_subscription(user_id, plan_type, payment.total_amount,
                       payment.currency, end_date, payment.telegram_payment_charge_id)

    await update.message.reply_text(
        f"🎉 Спасибо за покупку премиум подписки!\n\n"
        f"✅ Ваша подписка активна до {end_date.strftime('%d.%m.%Y')}\n\n"
        "Теперь вам доступны:\n"
        "📊 Полная статистика прогресса\n"
        "🎯 Солянка заданий (все категории)\n"
        "🔄 Работа над ошибками\n"
        "⚡ Приоритетная поддержка\n\n"
        "Приятного обучения! 🚀",
        reply_markup=get_main_menu()
    )

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    db.add_user(user.id, user.username, user.first_name, user.last_name)

    # Проверяем подписку
    has_premium = has_premium_access(user.id)
    premium_status = "💎 Премиум аккаунт" if has_premium else "🎯 Бесплатный аккаунт"

    await update.message.reply_text(
        f"🎓 Привет, {user.first_name}!\n{premium_status}\n\n"
        "🧪🔬 Я - БОТ-РЕШАЛКА, помогу тебе в подготовке к ЕГЭ по химии и биологии!\n\n"
        "📚 Решай задания каждый день для лучшего результата!\n"
        "🔄 Работа над ошибками поможет закрепить сложные темы!\n"
        "⚡ Выбирай предмет и начинай готовиться!",
        reply_markup=get_main_menu()
    )

async def show_profile(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    user_id = user.id

    # Получаем информацию о пользователе
    current_time = db.get_notification_time(user_id)
    has_premium = has_premium_access(user_id)
    subscription = db.get_active_subscription(user_id)

    profile_text = f"👤 Ваш профиль:\n\n"
    profile_text += f"🆔 ID: {user_id}\n"
    profile_text += f"👤 Имя: {user.first_name or 'Не указано'}\n"
    profile_text += f"📱 Username: @{user.username or 'Не указан'}\n"
    profile_text += f"🕐 Время уведомлений: {current_time}\n"
    profile_text += f"💎 Статус: {'Премиум' if has_premium else 'Бесплатный'}\n"

    if subscription:
        # Парсим дату окончания подписки
        if isinstance(subscription['end_date'], str):
            try:
                end_date = datetime.datetime.strptime(subscription['end_date'], '%Y-%m-%d %H:%M:%S')
            except ValueError:
                try:
                    end_date = datetime.datetime.strptime(subscription['end_date'], '%Y-%m-%d %H:%M:%S.%f')
                except ValueError:
                    end_date = datetime.datetime.now()
        else:
            end_date = subscription['end_date']

        days_left = (end_date - datetime.datetime.now()).days
        profile_text += f"📅 Подписка активна до: {end_date.strftime('%d.%m.%Y %H:%M')}\n"
        profile_text += f"📆 Осталось дней: {days_left}\n"
    else:
        profile_text += f"📅 Подписка: не активна\n"

    await update.message.reply_text(
        profile_text,
        reply_markup=get_profile_menu()
    )

async def show_premium_menu(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    subscription = db.get_active_subscription(user_id)

    if subscription and has_premium_access(user_id):
        if isinstance(subscription['end_date'], str):
            try:
                end_date = datetime.datetime.strptime(subscription['end_date'], '%Y-%m-%d %H:%M:%S')
            except ValueError:
                try:
                    end_date = datetime.datetime.strptime(subscription['end_date'], '%Y-%m-%d %H:%M:%S.%f')
                except ValueError:
                    end_date = datetime.datetime.now()
        else:
            end_date = subscription['end_date']

        days_left = (end_date - datetime.datetime.now()).days

        await update.message.reply_text(
            f"💎 Ваша премиум подписка активна!\n\n"
            f"📅 Тип: {subscription['plan_type']}\n"
            f"💰 Стоимость: {subscription['price'] / 100} {subscription['currency']}\n"
            f"⏰ Действует до: {end_date.strftime('%d.%m.%Y')}\n"
            f"📆 Осталось дней: {days_left}\n\n"
            "✅ Вам доступны все премиум функции!",
            reply_markup=get_premium_menu()
        )
    else:
        await update.message.reply_text(
            "💰 Премиум подписка\n\n"
            "💎 Премиум функции:\n"
            "✅ Полная статистика прогресса\n"
            "✅ Солянка заданий (все категории)\n"
            "✅ Работа над ошибками\n"
            "✅ Приоритетная поддержка\n\n"
            "💳 Выберите тариф:",
            reply_markup=get_premium_plans()
        )

async def handle_premium_plans(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text

    if text == '📅 Месяц - 199₽':
        await send_invoice(update, context, 'monthly')
    elif text == '📅 Год - 1910₽':
        await send_invoice(update, context, 'yearly')
    elif text == '🔙 Назад':
        await show_profile(update, context)

async def stats_all(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Показывает статистику всех пользователей"""
    user = update.effective_user

    all_stats = db.get_all_users_stats()

    if not all_stats:
        await update.message.reply_text("📊 Пока нет данных о пользователях.")
        return

    # Формируем сообщение со статистикой
    message = "📊 СТАТИСТИКА ВСЕХ ПОЛЬЗОВАТЕЛЕЙ\n\n"

    for i, user_stats in enumerate(all_stats, 1):
        # Форматируем имя пользователя
        username = user_stats['username'] or "Без username"
        first_name = user_stats['first_name'] or ""
        last_name = user_stats['last_name'] or ""

        # Форматируем время последней активности
        last_activity = user_stats['last_activity']
        if isinstance(last_activity, str):
            last_activity = last_activity[:16]  # Обрезаем до минут
        else:
            last_activity = str(last_activity)[:16]

        # Рассчитываем точность
        total_tasks = user_stats['total_tasks']
        correct_tasks = user_stats['correct_tasks']
        accuracy = (correct_tasks / total_tasks * 100) if total_tasks > 0 else 0

        # Статистика по предметам
        chem_total = user_stats['chemistry_total']
        chem_correct = user_stats['chemistry_correct']
        bio_total = user_stats['biology_total']
        bio_correct = user_stats['biology_correct']

        message += (
            f"👤 {i}. {first_name} {last_name}\n"
            f"   📱 @{username}\n"
            f"   🆔 ID: {user_stats['user_id']}\n"
            f"   📊 Всего заданий: {total_tasks}\n"
            f"   ✅ Правильных: {correct_tasks}\n"
            f"   🎯 Точность: {accuracy:.1f}%\n"
            f"   🧪 Химия: {chem_total} ({chem_correct} правильных)\n"
            f"   🧬 Биология: {bio_total} ({bio_correct} правильных)\n"
            f"   💎 Статус: {user_stats['premium_status']}\n"
            f"   ⏰ Последняя активность: {last_activity}\n"
            f"{'-' * 40}\n"
        )

    # Если сообщение слишком длинное, разбиваем на части
    if len(message) > 4000:
        parts = []
        current_part = ""
        lines = message.split('\n')

        for line in lines:
            if len(current_part + line + '\n') > 4000:
                parts.append(current_part)
                current_part = line + '\n'
            else:
                current_part += line + '\n'

        if current_part:
            parts.append(current_part)

        for part in parts:
            await update.message.reply_text(part)
    else:
        await update.message.reply_text(message)

async def handle_main_menu(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text

    if text == '🧪 Химия':
        await update.message.reply_text(
            "🧪 Выбери раздел химии:",
            reply_markup=get_chemistry_categories()
        )
    elif text == '🧬 Биология':
        await update.message.reply_text(
            "🧬 Выбери раздел биологии:",
            reply_markup=get_biology_categories()
        )
    elif text == '📊 Статистика':
        # Проверяем премиум доступ для статистики
        user_id = update.effective_user.id
        if not has_premium_access(user_id):
            await update.message.reply_text(
                "🚫 Полная статистика доступна только с премиум подпиской!\n\n"
                "💎 Премиум подписка включает:\n"
                "✅ Подробную статистику по всем категориям\n"
                "✅ Прогресс по времени\n"
                "✅ Сравнение с другими учениками\n\n"
                "💳 Выберите тариф:",
                reply_markup=get_premium_plans()
            )
            return
        await show_stats_menu(update, context)
    elif text == '👤 Мой профиль':
        await show_profile(update, context)
    elif text == '🎯 Сегодняшний прогресс':
        await show_today_progress(update, context)
    elif text == '🔄 Работа над ошибками':
        # Проверяем премиум доступ для работы над ошибками
        user_id = update.effective_user.id
        if not has_premium_access(user_id):
            await update.message.reply_text(
                "🚫 Работа над ошибками доступна только с премиум подпиской!\n\n"
                "💎 Премиум подписка включает:\n"
                "✅ Анализ ошибок по всем предметам\n"
                "✅ Персональные рекомендации\n"
                "✅ Тренировка слабых мест\n\n"
                "💳 Выберите тариф:",
                reply_markup=get_premium_plans()
            )
            return
        await show_error_work_menu(update, context)

async def show_error_work_menu(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user

    chem_errors = len(db.get_incorrect_tasks(user.id, 'chemistry'))
    bio_errors = len(db.get_incorrect_tasks(user.id, 'biology'))

    await update.message.reply_text(
        f"🔄 Работа над ошибками:\n\n"
        f"🧪 Ошибок по химии: {chem_errors}\n"
        f"🧬 Ошибок по биологии: {bio_errors}\n\n"
        f"Выбери предмет для работы над ошибками:",
        reply_markup=get_error_work_menu()
    )

async def handle_error_work(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text

    if text == '🧪 Ошибки по химии':
        context.user_data['error_work'] = True
        context.user_data['error_subject'] = 'chemistry'
        await send_next_chemistry_task(update, context, 'Работа над ошибками', error_work=True)
        return True
    elif text == '🧬 Ошибки по биологии':
        context.user_data['error_work'] = True
        context.user_data['error_subject'] = 'biology'
        await send_next_biology_task(update, context, 'Работа над ошибками', error_work=True)
        return True

    return False

async def show_stats_menu(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user

    chem_total, chem_correct = db.get_user_stats_by_subject(user.id, 'chemistry')
    bio_total, bio_correct = db.get_user_stats_by_subject(user.id, 'biology')

    total_tasks = chem_total + bio_total
    total_correct = chem_correct + bio_correct

    today_completed = db.get_today_completed_tasks(user.id)

    accuracy = (total_correct / total_tasks * 100) if total_tasks > 0 else 0

    await update.message.reply_text(
        f"📊 Общая статистика:\n\n"
        f"📚 Всего заданий: {total_tasks}\n"
        f"✅ Правильных ответов: {total_correct}\n"
        f"🎯 Точность: {accuracy:.1f}%\n\n"
        f"🧪 Химия: {chem_total} заданий\n"
        f"🧬 Биология: {bio_total} заданий\n\n"
        f"📅 Сегодня выполнено: {today_completed} заданий",
        reply_markup=ReplyKeyboardMarkup([
            ['📊 Статистика по химии', '📈 Статистика по биологии'],
            ['🔙 Главное меню']
        ], resize_keyboard=True)
    )

async def show_chemistry_stats(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user

    # Получаем статистику по всем категориям химии
    categories = ['Неорганическая химия', 'Органическая химия', 'Задачи']
    stats_text = "📊 Статистика по химии:\n\n"

    total_chem = 0
    total_correct_chem = 0

    for category in categories:
        completed, correct = db.get_user_stats_by_category(user.id, 'chemistry', category)
        total_chem += completed
        total_correct_chem += correct
        accuracy = (correct / completed * 100) if completed > 0 else 0
        stats_text += f"🔸 {category}: {completed} заданий, {correct} правильных ({accuracy:.1f}%)\n"

    overall_accuracy = (total_correct_chem / total_chem * 100) if total_chem > 0 else 0
    stats_text += f"\n🎯 Общее по химии: {total_chem} заданий, {total_correct_chem} правильных ({overall_accuracy:.1f}%)"

    await update.message.reply_text(
        stats_text,
        reply_markup=ReplyKeyboardMarkup([
            ['📈 Статистика по биологии', '📊 Общая статистика'],
            ['🔙 Главное меню']
        ], resize_keyboard=True)
    )

async def show_biology_stats(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user

    # Получаем статистику по всем категориям биологии
    categories = ['Общая биология', 'Анатомия', 'Ботаника', 'Зоология', 'Генетика']
    stats_text = "📈 Статистика по биологии:\n\n"

    total_bio = 0
    total_correct_bio = 0

    for category in categories:
        completed, correct = db.get_user_stats_by_category(user.id, 'biology', category)
        total_bio += completed
        total_correct_bio += correct
        accuracy = (correct / completed * 100) if completed > 0 else 0
        stats_text += f"🔸 {category}: {completed} заданий, {correct} правильных ({accuracy:.1f}%)\n"

    overall_accuracy = (total_correct_bio / total_bio * 100) if total_bio > 0 else 0
    stats_text += f"\n🎯 Общее по биологии: {total_bio} заданий, {total_correct_bio} правильных ({overall_accuracy:.1f}%)"

    await update.message.reply_text(
        stats_text,
        reply_markup=ReplyKeyboardMarkup([
            ['📊 Статистика по химии', '📊 Общая статистика'],
            ['🔙 Главное меню']
        ], resize_keyboard=True)
    )

async def show_today_progress(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    today_completed = db.get_today_completed_tasks(user.id)

    if today_completed >= 5:
        message = f"🎉 Отлично! Сегодня ты уже выполнил {today_completed} заданий!\nРекомендуемая норма выполнена! 💪"
    elif today_completed > 0:
        message = f"📅 Сегодня выполнено: {today_completed} заданий\nРекомендуется решить еще {5 - today_completed} заданий! 🎯"
    else:
        message = "📅 Сегодня ты еще не решал задания.\nНачни подготовку прямо сейчас! 🚀"

    await update.message.reply_text(
        message,
        reply_markup=get_main_menu()
    )

async def handle_chemistry_category(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text

    chemistry_categories = ['Неорганическая химия', 'Органическая химия', 'Задачи', 'Солянка заданий по химии']

    if text not in chemistry_categories:
        return False

    # Проверяем премиум доступ для "Солянка заданий по химии"
    if text == 'Солянка заданий по химии':
        user_id = update.effective_user.id
        if not has_premium_access(user_id):
            await update.message.reply_text(
                "🚫 Солянка заданий по химии доступна только с премиум подпиской!\n\n"
                "💎 Премиум подписка включает:\n"
                "✅ Смешанные задания из всех категорий\n"
                "✅ Неограниченное количество заданий\n"
                "✅ Все предметы в одном разделе\n\n"
                "💳 Выберите тариф:",
                reply_markup=get_premium_plans()
            )
            return True

    category = text

    # Инициализируем счетчик заданий в сессии
    if 'tasks_in_session' not in context.user_data:
        context.user_data['tasks_in_session'] = 0
        context.user_data['session_category'] = category
        context.user_data['session_subject'] = 'chemistry'

    await send_next_chemistry_task(update, context, category)
    return True

async def handle_biology_category(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text

    biology_categories = ['Общая биология', 'Анатомия', 'Ботаника', 'Зоология',
                         'Генетика', 'Солянка заданий по биологии']

    if text not in biology_categories:
        return False

    # Проверяем премиум доступ для "Солянка заданий по биологии"
    if text == 'Солянка заданий по биологии':
        user_id = update.effective_user.id
        if not has_premium_access(user_id):
            await update.message.reply_text(
                "🚫 Солянка заданий по биологии доступна только с премиум подпиской!\n\n"
                "💎 Премиум подписка включает:\n"
                "✅ Смешанные задания из всех категорий\n"
                "✅ Неограниченное количество заданий\n"
                "✅ Все предметы в одном разделе\n\n"
                "💳 Выберите тариф:",
                reply_markup=get_premium_plans()
            )
            return True

    category = text

    # Инициализируем счетчик заданий в сессии
    if 'tasks_in_session' not in context.user_data:
        context.user_data['tasks_in_session'] = 0
        context.user_data['session_category'] = category
        context.user_data['session_subject'] = 'biology'

    await send_next_biology_task(update, context, category)
    return True

async def send_next_chemistry_task(update: Update, context: ContextTypes.DEFAULT_TYPE, category, error_work=False):
    task_info = chemistry_manager.get_random_task(update.effective_user.id, category, error_work)

    if task_info is None:
        if error_work:
            message = "🎉 Отлично! Ты проработал все ошибки по химии!\nПродолжай заниматься! 💪"
        else:
            if category == 'Солянка заданий по химии':
                message = "📚 На данный момент все доступные задания в разделе 'Солянка заданий по химии' выполнены.\n\n🔔 Новые задания появятся позже! Возвращайтесь завтра!"
            else:
                message = f"🎉 На сегодня все задания по химии ({category}) выполнены!\nВыбери другой раздел или возвращайся завтра!"

        await update.message.reply_text(
            message,
            reply_markup=get_chemistry_categories()
        )
        # Сбрасываем сессию
        context.user_data.pop('tasks_in_session', None)
        context.user_data.pop('session_category', None)
        context.user_data.pop('session_subject', None)
        context.user_data.pop('error_work', None)
        return

    task, actual_category = task_info

    keyboard = [[opt] for opt in task['options']]
    keyboard.append(['🔙 Назад к разделам'])

    # Если есть фото, отправляем его с текстом
    if 'photo_url' in task:
        try:
            await update.message.reply_photo(
                photo=task['photo_url'],
                caption=task['question'] + "\n\nВыбери правильный вариант:",
                reply_markup=ReplyKeyboardMarkup(keyboard, resize_keyboard=True)
            )
        except Exception as e:
            # Если не удалось отправить фото, отправляем только текст
            logging.error(f"Ошибка отправки фото: {e}")
            await update.message.reply_text(
                task['question'] + "\n\nВыбери правильный вариант:",
                reply_markup=ReplyKeyboardMarkup(keyboard, resize_keyboard=True)
            )
    else:
        # Если фото нет, отправляем только текст
        await update.message.reply_text(
            task['question'] + "\n\nВыбери правильный вариант:",
            reply_markup=ReplyKeyboardMarkup(keyboard, resize_keyboard=True)
        )

    # Сохраняем задание в контексте
    context.user_data['current_task'] = task
    context.user_data['current_subject'] = 'chemistry'
    context.user_data['current_category'] = actual_category
    if error_work:
        context.user_data['error_work'] = True

async def send_next_biology_task(update: Update, context: ContextTypes.DEFAULT_TYPE, category, error_work=False):
    task_info = biology_manager.get_random_task(update.effective_user.id, category, error_work)

    if task_info is None:
        if error_work:
            message = "🎉 Отлично! Ты проработал все ошибки по биологии!\nПродолжай заниматься! 💪"
        else:
            if category == 'Солянка заданий по биологии':
                message = "📚 На данный момент все доступные задания в разделе 'Солянка заданий по биологии' выполнены.\n\n🔔 Новые задания появятся позже! Возвращайтесь завтра!"
            else:
                message = f"🎉 На сегодня все задания по биологии ({category}) выполнены!\nВыбери другой раздел или возвращайся завтра!"

        await update.message.reply_text(
            message,
            reply_markup=get_biology_categories()
        )
        # Сбрасываем сессию
        context.user_data.pop('tasks_in_session', None)
        context.user_data.pop('session_category', None)
        context.user_data.pop('session_subject', None)
        context.user_data.pop('error_work', None)
        return

    task, actual_category = task_info

    keyboard = [[opt] for opt in task['options']]
    keyboard.append(['🔙 Назад к разделам'])

    # Если есть фото, отправляем его с текстом
    if 'photo_url' in task:
        try:
            await update.message.reply_photo(
                photo=task['photo_url'],
                caption=task['question'] + "\n\nВыбери правильный вариант:",
                reply_markup=ReplyKeyboardMarkup(keyboard, resize_keyboard=True)
            )
        except Exception as e:
            # Если не удалось отправить фото, отправляем только текст
            logging.error(f"Ошибка отправки фото: {e}")
            await update.message.reply_text(
                task['question'] + "\n\nВыбери правильный вариант:",
                reply_markup=ReplyKeyboardMarkup(keyboard, resize_keyboard=True)
            )
    else:
        # Если фото нет, отправляем только текст
        await update.message.reply_text(
            task['question'] + "\n\nВыбери правильный вариант:",
            reply_markup=ReplyKeyboardMarkup(keyboard, resize_keyboard=True)
        )

    # Сохраняем задание в контексте
    context.user_data['current_task'] = task
    context.user_data['current_subject'] = 'biology'
    context.user_data['current_category'] = actual_category
    if error_work:
        context.user_data['error_work'] = True

async def handle_settings(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text

    if text == '🕐 Изменить время уведомлений':
        current_time = db.get_notification_time(update.effective_user.id)
        await update.message.reply_text(
            f"🕐 Текущее время уведомлений: {current_time}\n\n"
            "Введи новое время для ежедневных напоминаний в формате ЧЧ:ММ (например, 18:00):",
            reply_markup=get_back_menu()
        )
        context.user_data['waiting_for_time'] = True
        return True
    elif text == '💰 Премиум подписка':
        await show_premium_menu(update, context)
        return True

    return False

async def handle_time_input(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if context.user_data.get('waiting_for_time'):
        time_str = update.message.text

        if re.match(r'^([0-1]?[0-9]|2[0-3]):[0-5][0-9]$', time_str):
            db.set_notification_time(update.effective_user.id, time_str)

            await update.message.reply_text(
                f"✅ Время уведомлений установлено на {time_str}\n"
                f"Я буду напоминать тебе каждый день в это время!",
                reply_markup=get_profile_menu()
            )
            context.user_data.pop('waiting_for_time', None)
        else:
            await update.message.reply_text(
                "❌ Неверный формат времени. Введи время в формате ЧЧ:ММ (например, 18:00):",
                reply_markup=get_back_menu()
            )
        return True

    return False

async def handle_navigation(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text

    if text == '🔙 Главное меню':
        await update.message.reply_text(
            "Возвращаемся в главное меню:",
            reply_markup=get_main_menu()
        )
        # Сбрасываем сессию
        context.user_data.pop('tasks_in_session', None)
        context.user_data.pop('session_category', None)
        context.user_data.pop('session_subject', None)
        context.user_data.pop('current_task', None)
        context.user_data.pop('current_subject', None)
        context.user_data.pop('current_category', None)
        context.user_data.pop('error_work', None)
        context.user_data.pop('waiting_for_time', None)
        return True
    elif text == '🔙 Назад к разделам':
        # Определяем, из какого меню мы пришли
        current_subject = context.user_data.get('current_subject')
        if current_subject == 'chemistry':
            await update.message.reply_text(
                "🧪 Выбери раздел химии:",
                reply_markup=get_chemistry_categories()
            )
        elif current_subject == 'biology':
            await update.message.reply_text(
                "🧬 Выбери раздел биологии:",
                reply_markup=get_biology_categories()
            )
        else:
            # Если не можем определить предмет, возвращаем в главное меню
            await update.message.reply_text(
                "Возвращаемся в главное меню:",
                reply_markup=get_main_menu()
            )

        # Сбрасываем сессию
        context.user_data.pop('tasks_in_session', None)
        context.user_data.pop('session_category', None)
        context.user_data.pop('session_subject', None)
        context.user_data.pop('current_task', None)
        context.user_data.pop('current_subject', None)
        context.user_data.pop('current_category', None)
        context.user_data.pop('error_work', None)
        return True
    elif text == '✅ Продолжить':
        # Продолжаем сессию
        subject = context.user_data.get('session_subject')
        category = context.user_data.get('session_category')
        error_work = context.user_data.get('error_work', False)

        if subject == 'chemistry':
            await send_next_chemistry_task(update, context, category, error_work)
        elif subject == 'biology':
            await send_next_biology_task(update, context, category, error_work)
        return True
    elif text == '🔙 Выбрать другой раздел':
        subject = context.user_data.get('session_subject')
        if subject == 'chemistry':
            await update.message.reply_text(
                "🧪 Выбери раздел химии:",
                reply_markup=get_chemistry_categories()
            )
        elif subject == 'biology':
            await update.message.reply_text(
                "🧬 Выбери раздел биологии:",
                reply_markup=get_biology_categories()
            )
        # Сбрасываем сессию
        context.user_data.pop('tasks_in_session', None)
        context.user_data.pop('session_category', None)
        context.user_data.pop('session_subject', None)
        context.user_data.pop('error_work', None)
        return True
    elif text == '🔙 Назад':
        await show_profile(update, context)
        return True

    return False

async def handle_task_answer(update: Update, context: ContextTypes.DEFAULT_TYPE):
    # Проверяем, есть ли текущее задание в контексте
    if 'current_task' not in context.user_data:
        await update.message.reply_text(
            "Пожалуйста, выбери задание из меню.",
            reply_markup=get_main_menu()
        )
        return

    task = context.user_data['current_task']
    subject = context.user_data['current_subject']
    category = context.user_data['current_category']
    user_answer = update.message.text

    correct_index = task['answer']
    correct_answer = task['options'][correct_index]

    # Нормализуем ответы для сравнения
    normalized_user_answer = normalize_answer(user_answer)
    normalized_correct = normalize_answer(correct_answer)

    # Проверяем разные форматы ответа
    is_correct = False

    # 1. Проверка полного совпадения
    if normalized_user_answer == normalized_correct:
        is_correct = True

    # 2. Проверка по номеру (только цифра)
    elif normalized_user_answer == str(correct_index + 1):
        is_correct = True

    # 3. Проверка по тексту ответа (без номера)
    elif ')' in correct_answer:
        correct_text = correct_answer.split(') ', 1)[1].lower().strip()
        if normalized_user_answer == normalize_answer(correct_text):
            is_correct = True

    # 4. Проверка частичного совпадения
    elif normalized_user_answer in normalized_correct:
        is_correct = True

    # 5. Проверка по первому слову
    elif normalized_user_answer.split()[0] == normalized_correct.split()[0]:
        is_correct = True

    # Сохраняем результат в базу
    db.mark_task_sent(update.effective_user.id, task['id'], subject, category, is_correct)

    # Увеличиваем счетчик заданий в сессии
    if 'tasks_in_session' in context.user_data:
        context.user_data['tasks_in_session'] += 1
    else:
        context.user_data['tasks_in_session'] = 1

    tasks_in_session = context.user_data['tasks_in_session']

    # ОБНОВЛЯЕМ СТАТУС ОШИБКИ - если задание решено правильно и это была работа над ошибками
    error_work = context.user_data.get('error_work', False)
    if error_work and is_correct:
        # Помечаем задание как правильное в базе данных
        db.update_task_correctness(update.effective_user.id, task['id'], True)

    # Формируем ответ
    if is_correct:
        response = f"✅ Правильно!\n\n{task['explanation']}"
    else:
        response = f"❌ Неправильно. Правильный ответ: {correct_answer}\n\n{task['explanation']}"

    await update.message.reply_text(response)

    # Проверяем, достигнута ли рекомендуемая норма
    today_completed = db.get_today_completed_tasks(update.effective_user.id)

    if tasks_in_session >= 5:
        # Поздравляем с выполнением нормы
        await update.message.reply_text(
            f"🎉 Поздравляю! Ты выполнил рекомендуемую норму в 5 заданий!\n"
            f"📊 Всего сегодня: {today_completed} заданий\n\n"
            f"Хочешь продолжить или выбрать другой раздел?",
            reply_markup=get_continue_menu()
        )
    else:
        # Предлагаем следующее задание
        if subject == 'chemistry':
            await send_next_chemistry_task(update, context, context.user_data.get('session_category'), error_work)
        elif subject == 'biology':
            await send_next_biology_task(update, context, context.user_data.get('session_category'), error_work)

async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text

    # Сначала проверяем главное меню
    main_menu_buttons = ['🧪 Химия', '🧬 Биология', '📊 Статистика', '👤 Мой профиль', '🎯 Сегодняшний прогресс', '🔄 Работа над ошибками']
    if text in main_menu_buttons:
        await handle_main_menu(update, context)
        return

    # Проверяем профиль меню
    profile_menu_buttons = ['🕐 Изменить время уведомлений', '💰 Премиум подписка']
    if text in profile_menu_buttons:
        await handle_settings(update, context)
        return

    # Проверяем премиум меню
    premium_menu_buttons = ['💎 Купить премиум', '📊 Моя подписка']
    if text in premium_menu_buttons:
        if text == '💎 Купить премиум':
            await show_premium_menu(update, context)
        elif text == '📊 Моя подписка':
            await show_premium_menu(update, context)
        return

    # Проверяем тарифы премиум
    premium_plan_buttons = ['📅 Месяц - 199₽', '📅 Год - 1910₽', '🔙 Назад']
    if text in premium_plan_buttons:
        await handle_premium_plans(update, context)
        return

    # Проверяем статистику
    if text == '📊 Статистика по химии':
        user_id = update.effective_user.id
        if not has_premium_access(user_id):
            await update.message.reply_text(
                "🚫 Детальная статистика доступна только с премиум подпиской!",
                reply_markup=get_premium_plans()
            )
            return
        await show_chemistry_stats(update, context)
        return
    elif text == '📈 Статистика по биологии':
        user_id = update.effective_user.id
        if not has_premium_access(user_id):
            await update.message.reply_text(
                "🚫 Детальная статистика доступна только с премиум подпиской!",
                reply_markup=get_premium_plans()
            )
            return
        await show_biology_stats(update, context)
        return
    elif text == '📊 Общая статистика':
        user_id = update.effective_user.id
        if not has_premium_access(user_id):
            await update.message.reply_text(
                "🚫 Полная статистика доступна только с премиум подпиской!",
                reply_markup=get_premium_plans()
            )
            return
        await show_stats_menu(update, context)
        return

    # Проверяем работу над ошибками
    if await handle_error_work(update, context):
        return

    # Затем проверяем навигационные кнопки
    if await handle_navigation(update, context):
        return

    # Проверяем настройки
    if await handle_settings(update, context):
        return

    # Проверяем ввод времени
    if await handle_time_input(update, context):
        return

    # Проверяем категории химии
    if await handle_chemistry_category(update, context):
        return

    # Проверяем категории биологии
    if await handle_biology_category(update, context):
        return

    # Если это ответ на задание
    if 'current_task' in context.user_data:
        await handle_task_answer(update, context)
        return

    # Если не распознали команду, показываем главное меню
    await update.message.reply_text(
        "Выбери действие из меню:",
        reply_markup=get_main_menu()
    )

# Функции для уведомлений
async def send_daily_notifications():
    """Функция для ежедневной рассылки уведомлений по Московскому времени"""
    global application_instance

    if application_instance is None:
        logging.error("Application instance not available")
        return

    users = db.get_users_for_notification()

    # Получаем текущее время по Москве
    moscow_time = datetime.datetime.now(MOSCOW_TZ)
    current_time = moscow_time.strftime('%H:%M')

    logging.info(f"🔔 Проверка уведомлений. Московское время: {current_time}")

    notification_sent = 0
    for user_id, notification_time in users:
        try:
            # Проверяем совпадение времени и отсутствие уведомления сегодня
            if current_time == notification_time and not db.has_received_notification_today(user_id):
                today_completed = db.get_today_completed_tasks(user_id)

                if today_completed == 0:
                    message = "📚 Привет! Напоминаю, что сегодня ты еще не решал задания.\nНе откладывай подготовку к ЕГЭ! 🚀"
                elif today_completed < 5:
                    message = f"📚 Привет! Сегодня ты уже выполнил {today_completed} заданий.\nРекомендуется решить еще {5 - today_completed} для лучшего результата! 💪"
                else:
                    message = f"🎉 Отлично! Сегодня ты уже выполнил {today_completed} заданий!\nТак держать! 💪"

                await application_instance.bot.send_message(
                    chat_id=user_id,
                    text=message,
                    reply_markup=get_main_menu()
                )

                # Отмечаем, что уведомление было отправлено сегодня
                db.mark_notification_sent(user_id)
                notification_sent += 1

                logging.info(f"✅ Уведомление отправлено пользователю {user_id} в {current_time} МСК")

                await asyncio.sleep(1)  # Задержка между отправками

        except Exception as e:
            logging.error(f"❌ Ошибка отправки уведомления пользователю {user_id}: {e}")

    if notification_sent > 0:
        logging.info(f"📨 Отправлено {notification_sent} уведомлений в {current_time} МСК")

async def check_subscription_expiry():
    """Проверяет и уведомляет пользователей об окончании подписки по Московскому времени"""
    global application_instance

    if application_instance is None:
        logging.error("Application instance not available")
        return

    try:
        # Получаем текущую дату по Москве
        current_date_moscow = datetime.datetime.now(MOSCOW_TZ).date()

        # Получаем всех пользователей с активными подписками
        cursor = db.conn.cursor()
        cursor.execute('''
            SELECT s.user_id, s.end_date, u.first_name
            FROM subscriptions s
            JOIN users u ON s.user_id = u.user_id
            WHERE s.is_active = 1 AND s.end_date > CURRENT_TIMESTAMP
        ''')
        active_subscriptions = cursor.fetchall()

        for user_id, end_date, first_name in active_subscriptions:
            if isinstance(end_date, str):
                try:
                    # Предполагаем, что дата в UTC, конвертируем в Москву
                    end_date_utc = datetime.datetime.strptime(end_date, '%Y-%m-%d %H:%M:%S').replace(tzinfo=pytz.UTC)
                    end_date_moscow = end_date_utc.astimezone(MOSCOW_TZ)
                except ValueError:
                    try:
                        end_date_utc = datetime.datetime.strptime(end_date, '%Y-%m-%d %H:%M:%S.%f').replace(tzinfo=pytz.UTC)
                        end_date_moscow = end_date_utc.astimezone(MOSCOW_TZ)
                    except ValueError:
                        logging.error(f"Неизвестный формат даты: {end_date}")
                        continue
            else:
                # Если это datetime объект, конвертируем в Москву
                end_date_utc = end_date.replace(tzinfo=pytz.UTC)
                end_date_moscow = end_date_utc.astimezone(MOSCOW_TZ)

            time_until_expiry = end_date_moscow.date() - current_date_moscow

            # Уведомляем за разные периоды
            if time_until_expiry.days == 3:
                message = f"🔔 Напоминание, {first_name or 'друг'}!\n\nВаша премиум подписка закончится через 3 дня.\nДата окончания: {end_date_moscow.strftime('%d.%m.%Y')}\n\nНе упустите возможность продолжить обучение с полным доступом! 💎"
                await send_notification_safe(user_id, message)

            elif time_until_expiry.days == 1:
                message = f"⚠️ Важно, {first_name or 'друг'}!\n\nВаша премиум подписка закончится ЗАВТРА!\nДата окончания: {end_date_moscow.strftime('%d.%m.%Y')}\n\nПродлите подписку, чтобы сохранить все преимущества! 🚀"
                await send_notification_safe(user_id, message)

            elif time_until_expiry.days == 0:
                message = f"⏰ Срочное уведомление, {first_name or 'друг'}!\n\nВаша премиум подписка заканчивается СЕГОДНЯ!\n\nПосле окончания подписки будут недоступны:\n• Полная статистика\n• Солянка заданий\n• Работа над ошибками\n\nНе забудьте продлить подписку! 💎"
                await send_notification_safe(user_id, message)

            # Уведомление после окончания подписки
            elif time_until_expiry.days == -1:
                message = f"📢 {first_name or 'друг'}, ваша премиум подписка закончилась.\n\nХотите вернуть все преимущества?\n• Полную статистику прогресса\n• Солянку заданий\n• Работу над ошибками\n\n💳 Перейдите в раздел 'Мой профиль' → 'Премиум подписка' для продления!"
                await send_notification_safe(user_id, message)

    except Exception as e:
        logging.error(f"Ошибка при проверке подписок: {e}")

async def send_notification_safe(user_id, message):
    """Безопасная отправка уведомления с обработкой ошибок"""
    global application_instance

    if application_instance is None:
        logging.error("Application instance not available")
        return

    try:
        await application_instance.bot.send_message(
            chat_id=user_id,
            text=message,
            reply_markup=get_main_menu()
        )
        logging.info(f"Уведомление о подписке отправлено пользователю {user_id}")
        await asyncio.sleep(0.5)  # Задержка между отправками
    except Exception as e:
        logging.error(f"Ошибка отправки уведомления пользователю {user_id}: {e}")

def schedule_notifications():
    """Планировщик уведомлений - проверяет каждую минуту по Московскому времени"""
    scheduler = BackgroundScheduler(timezone=MOSCOW_TZ)

    # Ежедневные напоминания о заданиях - проверяем каждую минуту
    scheduler.add_job(
        lambda: asyncio.run(send_daily_notifications()),
        trigger=CronTrigger(minute='*'),  # Проверяем каждую минуту
        id='daily_reminder',
        replace_existing=True
    )

    # Проверка окончания подписок (каждый день в 10:00 по Москве)
    scheduler.add_job(
        lambda: asyncio.run(check_subscription_expiry()),
        trigger=CronTrigger(hour=10, minute=0),
        id='subscription_check',
        replace_existing=True
    )

    scheduler.start()
    logging.info("⏰ Планировщик уведомлений запущен (Московское время)")
    return scheduler

def main():
    global application_instance

    application_instance = Application.builder().token(BOT_TOKEN).build()

    # Обработчики команд
    application_instance.add_handler(CommandHandler("start", start))
    application_instance.add_handler(CommandHandler("statsALL", stats_all))

    # Обработчики платежей
    application_instance.add_handler(MessageHandler(filters.SUCCESSFUL_PAYMENT, successful_payment_handler))
    application_instance.add_handler(PreCheckoutQueryHandler(pre_checkout_handler))

    # Главный обработчик для всех текстовых сообщений
    application_instance.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))

    # Запускаем планировщик уведомлений
    schedule_notifications()

    logging.info("🤖 Бот запущен с системой подписок!")
    application_instance.run_polling()

if __name__ == "__main__":
    main()
