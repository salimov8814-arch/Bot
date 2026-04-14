import sqlite3
import logging
import io
from datetime import datetime, timedelta
from typing import Dict, List, Optional

from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    Application, CommandHandler, CallbackQueryHandler, ContextTypes,
    MessageHandler, filters
)
import matplotlib.pyplot as plt
from openpyxl import Workbook

# --- Настройка ---
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO
)
logger = logging.getLogger(__name__)

TOKEN = "YOUR_BOT_TOKEN_HERE"
DB_PATH = "earnings.db"

CATEGORIES = {
    "main": {"name": "💰 Основной доход", "unit": "currency"},
    "tonnage": {"name": "🚛 Тонаж", "unit": "ton"},
    "extra": {"name": "🔧 Халтура", "unit": "currency"}
}

# --- База данных ---
def init_db():
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute('''CREATE TABLE IF NOT EXISTS income (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER NOT NULL,
        category TEXT NOT NULL,
        amount REAL NOT NULL,
        description TEXT,
        date TEXT NOT NULL,
        created_at TEXT NOT NULL,
        linked_id INTEGER DEFAULT NULL
    )''')
    c.execute('''CREATE TABLE IF NOT EXISTS user_settings (
        user_id INTEGER PRIMARY KEY,
        main_currency TEXT DEFAULT '₽'
    )''')
    c.execute('''CREATE TABLE IF NOT EXISTS goals (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER NOT NULL,
        goal_type TEXT NOT NULL,
        threshold REAL NOT NULL,
        achieved BOOLEAN DEFAULT 0,
        created_at TEXT NOT NULL
    )''')
    conn.commit()
    conn.close()

def add_income(user_id: int, category: str, amount: float, description: str, date_str: str, linked_id: int = None) -> int:
    now = datetime.now().isoformat()
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute('''INSERT INTO income (user_id, category, amount, description, date, created_at, linked_id)
                 VALUES (?, ?, ?, ?, ?, ?, ?)''',
              (user_id, category, amount, description, date_str, now, linked_id))
    last_id = c.lastrowid
    conn.commit()
    conn.close()
    return last_id

def update_income(entry_id: int, amount: float, description: str):
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute('UPDATE income SET amount = ?, description = ? WHERE id = ?',
              (amount, description, entry_id))
    conn.commit()
    conn.close()

def delete_income(entry_id: int):
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    # Находим связанную запись (если есть)
    c.execute('SELECT linked_id FROM income WHERE id = ?', (entry_id,))
    row = c.fetchone()
    if row and row[0]:
        c.execute('DELETE FROM income WHERE id = ?', (row[0],))
    # Удаляем записи, которые ссылаются на этот id (на случай, если он был родителем)
    c.execute('DELETE FROM income WHERE linked_id = ?', (entry_id,))
    c.execute('DELETE FROM income WHERE id = ?', (entry_id,))
    conn.commit()
    conn.close()

def get_income_entry(entry_id: int) -> Optional[Dict]:
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute('SELECT id, user_id, category, amount, description, date, linked_id FROM income WHERE id = ?', (entry_id,))
    row = c.fetchone()
    conn.close()
    if row:
        return {'id': row[0], 'user_id': row[1], 'category': row[2], 'amount': row[3],
                'description': row[4], 'date': row[5], 'linked_id': row[6]}
    return None

def get_user_entries(user_id: int, limit: int = 50) -> List[Dict]:
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute('''SELECT id, category, amount, description, date FROM income 
                 WHERE user_id = ? ORDER BY date DESC, created_at DESC LIMIT ?''',
              (user_id, limit))
    rows = c.fetchall()
    conn.close()
    return [{'id': r[0], 'category': r[1], 'amount': r[2], 'description': r[3], 'date': r[4]} for r in rows]

def get_income_summary(user_id: int, period: str = 'month') -> Dict:
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    now = datetime.now()
    
    if period == 'day':
        start_date = now.strftime('%Y-%m-%d')
        period_name = "сегодня"
    elif period == 'week':
        start_date = (now - timedelta(days=now.weekday())).strftime('%Y-%m-%d')
        period_name = "на этой неделе"
    elif period == 'month':
        start_date = now.replace(day=1).strftime('%Y-%m-%d')
        period_name = "в этом месяце"
    elif period == 'year':
        start_date = now.replace(month=1, day=1).strftime('%Y-%m-%d')
        period_name = "в этом году"
    else:
        start_date = None
        period_name = "за всё время"
    
    if start_date:
        c.execute('SELECT category, amount, description, date FROM income WHERE user_id = ? AND date >= ?',
                  (user_id, start_date))
    else:
        c.execute('SELECT category, amount, description, date FROM income WHERE user_id = ?', (user_id,))
    
    rows = c.fetchall()
    conn.close()
    
    result = {
        'period_name': period_name,
        'main_total': 0.0,
        'tonnage_total': 0.0,
        'extra_total': 0.0,
        'details': {'main': [], 'tonnage': [], 'extra': []}
    }
    for cat, amount, desc, date in rows:
        if cat == 'main':
            result['main_total'] += amount
            result['details']['main'].append((amount, desc, date))
        elif cat == 'tonnage':
            result['tonnage_total'] += amount
            result['details']['tonnage'].append((amount, desc, date))
        elif cat == 'extra':
            result['extra_total'] += amount
            result['details']['extra'].append((amount, desc, date))
    
    for key in result['details']:
        result['details'][key].sort(key=lambda x: x[2], reverse=True)
    return result

def get_main_currency(user_id: int) -> str:
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute('SELECT main_currency FROM user_settings WHERE user_id = ?', (user_id,))
    row = c.fetchone()
    conn.close()
    return row[0] if row else '₽'

def set_main_currency(user_id: int, currency: str):
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute('''INSERT INTO user_settings (user_id, main_currency) VALUES (?, ?)
                 ON CONFLICT(user_id) DO UPDATE SET main_currency = excluded.main_currency''',
              (user_id, currency))
    conn.commit()
    conn.close()

# --- Цели ---
def add_goal(user_id: int, goal_type: str, threshold: float):
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute('''INSERT INTO goals (user_id, goal_type, threshold, achieved, created_at)
                 VALUES (?, ?, ?, 0, ?)''',
              (user_id, goal_type, threshold, datetime.now().isoformat()))
    conn.commit()
    conn.close()

def get_user_goals(user_id: int, only_unachieved: bool = False) -> List[Dict]:
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    if only_unachieved:
        c.execute('SELECT id, goal_type, threshold FROM goals WHERE user_id = ? AND achieved = 0', (user_id,))
    else:
        c.execute('SELECT id, goal_type, threshold, achieved FROM goals WHERE user_id = ?', (user_id,))
    rows = c.fetchall()
    conn.close()
    if only_unachieved:
        return [{'id': r[0], 'type': r[1], 'threshold': r[2]} for r in rows]
    else:
        return [{'id': r[0], 'type': r[1], 'threshold': r[2], 'achieved': r[3]} for r in rows]

def delete_goal(goal_id: int):
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute('DELETE FROM goals WHERE id = ?', (goal_id,))
    conn.commit()
    conn.close()

def check_and_notify_goals(user_id: int, context: ContextTypes.DEFAULT_TYPE):
    goals = get_user_goals(user_id, only_unachieved=True)
    if not goals:
        return
    summary_all = get_income_summary(user_id, 'all')
    current_money = summary_all['main_total'] + summary_all['extra_total']
    current_tonnage = summary_all['tonnage_total']
    for goal in goals:
        achieved = False
        if goal['type'] == 'tonnage' and current_tonnage >= goal['threshold']:
            achieved = True
            context.bot.send_message(chat_id=user_id, text=f"🚛 *Цель по тоннажу достигнута!* {current_tonnage:.1f} т (цель: {goal['threshold']:.1f} т)", parse_mode='Markdown')
        elif goal['type'] == 'money' and current_money >= goal['threshold']:
            achieved = True
            curr = get_main_currency(user_id)
            context.bot.send_message(chat_id=user_id, text=f"💰 *Цель по доходу достигнута!* {current_money:.2f} {curr} (цель: {goal['threshold']:.2f} {curr})", parse_mode='Markdown')
        if achieved:
            conn = sqlite3.connect(DB_PATH)
            c = conn.cursor()
            c.execute('UPDATE goals SET achieved = 1 WHERE id = ?', (goal['id'],))
            conn.commit()
            conn.close()

# --- Экспорт в Excel ---
async def export_to_excel(user_id: int) -> io.BytesIO:
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute('''SELECT category, amount, description, date, created_at FROM income 
                 WHERE user_id = ? ORDER BY date DESC''', (user_id,))
    rows = c.fetchall()
    conn.close()
    wb = Workbook()
    ws_all = wb.active
    ws_all.title = "Все доходы"
    ws_all.append(["Категория", "Сумма/Тоннаж", "Описание", "Дата", "Время добавления"])
    for row in rows:
        cat = row[0]
        amount = row[1]
        desc = row[2] or ""
        date = row[3]
        created = row[4]
        if cat == 'tonnage':
            amount_str = f"{amount} т"
        elif cat == 'main':
            amount_str = f"{amount} {get_main_currency(user_id)}"
        else:
            amount_str = f"{amount} ₽"
        ws_all.append([CATEGORIES[cat]['name'], amount_str, desc, date, created])
    ws_main = wb.create_sheet("Основной доход")
    ws_main.append(["Сумма", "Описание", "Дата"])
    for row in rows:
        if row[0] == 'main':
            ws_main.append([row[1], row[2] or "", row[3]])
    ws_ton = wb.create_sheet("Тонаж")
    ws_ton.append(["Тонны", "Описание", "Дата"])
    for row in rows:
        if row[0] == 'tonnage':
            ws_ton.append([row[1], row[2] or "", row[3]])
    ws_extra = wb.create_sheet("Халтура")
    ws_extra.append(["Сумма (₽)", "Описание", "Дата"])
    for row in rows:
        if row[0] == 'extra':
            ws_extra.append([row[1], row[2] or "", row[3]])
    output = io.BytesIO()
    wb.save(output)
    output.seek(0)
    return output

# --- Графики ---
async def generate_chart(user_id: int, period: str) -> io.BytesIO:
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    now = datetime.now()
    if period == 'week':
        start_date = (now - timedelta(days=now.weekday())).strftime('%Y-%m-%d')
        interval = "день"
    elif period == 'month':
        start_date = now.replace(day=1).strftime('%Y-%m-%d')
        interval = "день"
    elif period == 'year':
        start_date = now.replace(month=1, day=1).strftime('%Y-%m-%d')
        interval = "месяц"
    else:
        start_date = None
        interval = "месяц"
    if start_date:
        c.execute('''SELECT date, category, amount FROM income 
                     WHERE user_id = ? AND date >= ? ORDER BY date''', (user_id, start_date))
    else:
        c.execute('SELECT date, category, amount FROM income WHERE user_id = ? ORDER BY date', (user_id,))
    rows = c.fetchall()
    conn.close()
    if not rows:
        return None
    data = {'dates': [], 'money': [], 'tonnage': []}
    if interval == "день":
        daily = {}
        for date, cat, amt in rows:
            if date not in daily:
                daily[date] = {'money': 0.0, 'tonnage': 0.0}
            if cat == 'tonnage':
                daily[date]['tonnage'] += amt
            else:
                daily[date]['money'] += amt
        dates = sorted(daily.keys())
        data['dates'] = dates
        data['money'] = [daily[d]['money'] for d in dates]
        data['tonnage'] = [daily[d]['tonnage'] for d in dates]
    else:
        monthly = {}
        for date, cat, amt in rows:
            ym = date[:7]
            if ym not in monthly:
                monthly[ym] = {'money': 0.0, 'tonnage': 0.0}
            if cat == 'tonnage':
                monthly[ym]['tonnage'] += amt
            else:
                monthly[ym]['money'] += amt
        months = sorted(monthly.keys())
        data['dates'] = months
        data['money'] = [monthly[m]['money'] for m in months]
        data['tonnage'] = [monthly[m]['tonnage'] for m in months]
    fig, ax1 = plt.subplots(figsize=(10, 5))
    curr = get_main_currency(user_id)
    ax1.set_xlabel('Дата')
    ax1.set_ylabel(f'Доход ({curr})', color='tab:green')
    ax1.plot(data['dates'], data['money'], color='tab:green', marker='o', label=f'Доход ({curr})')
    ax1.tick_params(axis='y', labelcolor='tab:green')
    ax2 = ax1.twinx()
    ax2.set_ylabel('Тонаж (т)', color='tab:blue')
    ax2.plot(data['dates'], data['tonnage'], color='tab:blue', marker='s', label='Тонаж (т)')
    ax2.tick_params(axis='y', labelcolor='tab:blue')
    plt.title(f'Динамика доходов и тоннажа\n({data["dates"][0]} — {data["dates"][-1]})')
    fig.legend(loc='upper left', bbox_to_anchor=(0.1, 0.9))
    plt.xticks(rotation=45)
    plt.tight_layout()
    output = io.BytesIO()
    plt.savefig(output, format='png')
    output.seek(0)
    plt.close(fig)
    return output

# --- Клавиатуры ---
def main_menu_keyboard() -> InlineKeyboardMarkup:
    keyboard = [
        [InlineKeyboardButton("💰 Добавить основной доход", callback_data="add_main")],
        [InlineKeyboardButton("🚛 Добавить тоннаж", callback_data="add_tonnage")],
        [InlineKeyboardButton("🔧 Добавить халтуру", callback_data="add_extra")],
        [InlineKeyboardButton("📊 Отчёт за сегодня", callback_data="report_day"),
         InlineKeyboardButton("📅 Отчёт за месяц", callback_data="report_month")],
        [InlineKeyboardButton("📈 Общий отчёт", callback_data="report_all"),
         InlineKeyboardButton("⚙️ Настройки", callback_data="settings")],
        [InlineKeyboardButton("📎 Экспорт в Excel", callback_data="export"),
         InlineKeyboardButton("📈 График", callback_data="chart_menu")],
        [InlineKeyboardButton("🎯 Мои цели", callback_data="goals_menu"),
         InlineKeyboardButton("➕ Новая цель", callback_data="new_goal")]
    ]
    return InlineKeyboardMarkup(keyboard)

def back_keyboard() -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup([[InlineKeyboardButton("◀️ Назад", callback_data="back_to_menu")]])

def chart_menu_keyboard() -> InlineKeyboardMarkup:
    keyboard = [
        [InlineKeyboardButton("📅 За неделю", callback_data="chart_week"),
         InlineKeyboardButton("📆 За месяц", callback_data="chart_month")],
        [InlineKeyboardButton("📊 За год", callback_data="chart_year"),
         InlineKeyboardButton("🗓 Всё время", callback_data="chart_all")],
        [InlineKeyboardButton("◀️ Назад", callback_data="back_to_menu")]
    ]
    return InlineKeyboardMarkup(keyboard)

def goals_menu_keyboard(goals: List[Dict]) -> InlineKeyboardMarkup:
    keyboard = []
    for g in goals:
        status = "✅" if g['achieved'] else "⏳"
        type_name = "Тоннаж" if g['type'] == 'tonnage' else "Доход"
        keyboard.append([InlineKeyboardButton(f"{status} {type_name}: {g['threshold']}", callback_data=f"goal_del_{g['id']}")])
    keyboard.append([InlineKeyboardButton("➕ Новая цель", callback_data="new_goal")])
    keyboard.append([InlineKeyboardButton("◀️ Назад", callback_data="back_to_menu")])
    return InlineKeyboardMarkup(keyboard)

# --- Обработчики ---
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    user = update.effective_user
    await update.message.reply_text(
        f"👋 Привет, {user.first_name}!\n\n"
        f"Я помогу отслеживать доходы, тоннаж и халтуру.\n"
        f"При добавлении тоннажа автоматически начисляется основной доход из расчёта 1 т = 1000 руб.\n"
        f"Используйте кнопки меню:",
        reply_markup=main_menu_keyboard()
    )

async def button_handler(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    query = update.callback_query
    await query.answer()
    data = query.data
    user_id = query.from_user.id

    # ----- Добавление дохода (устанавливаем состояние) -----
    if data in ("add_main", "add_tonnage", "add_extra"):
        if data == "add_main":
            context.user_data['awaiting'] = 'main'
            unit = get_main_currency(user_id)
            name = "Основной доход"
        elif data == "add_tonnage":
            context.user_data['awaiting'] = 'tonnage'
            unit = "т"
            name = "Тонаж"
        else:
            context.user_data['awaiting'] = 'extra'
            unit = "₽"
            name = "Халтура"
        context.user_data['step'] = 'amount'
        await query.edit_message_text(
            f"➕ *Добавление: {name}*\n\nВведите количество ({unit}):\nНапример: `100` или `15.5`",
            parse_mode='Markdown',
            reply_markup=back_keyboard()
        )
        return

    # ----- Отчёты -----
    if data.startswith("report_"):
        period = data.split("_")[1]
        summary = get_income_summary(user_id, period)
        curr = get_main_currency(user_id)
        text = f"📊 *Отчёт {summary['period_name']}*\n\n"
        text += f"💰 *Основной доход:* `{summary['main_total']:.2f} {curr}`\n"
        text += f"🚛 *Тонаж:* `{summary['tonnage_total']:.2f} т`\n"
        text += f"🔧 *Халтура:* `{summary['extra_total']:.2f} ₽`\n"
        text += f"💵 *Всего денег:* `{summary['main_total'] + summary['extra_total']:.2f}` {curr if curr != '₽' else '₽'}\n\n"
        if any(summary['details'][k] for k in summary['details']):
            text += "*Последние записи:*\n"
            for cat_key, cat_name in [('main', 'Основной'), ('tonnage', 'Тонаж'), ('extra', 'Халтура')]:
                records = summary['details'][cat_key][:5]
                if records:
                    text += f"\n{cat_name}:\n"
                    for amt, desc, date in records:
                        unit = curr if cat_key == 'main' else ('т' if cat_key == 'tonnage' else '₽')
                        text += f"  • `{amt:.2f} {unit}` — {desc or 'без описания'} ({date})\n"
        keyboard = [
            [InlineKeyboardButton("📋 Список записей", callback_data="list_entries")],
            [InlineKeyboardButton("◀️ Назад", callback_data="back_to_menu")]
        ]
        await query.edit_message_text(text, parse_mode='Markdown', reply_markup=InlineKeyboardMarkup(keyboard))
        return

    if data == "list_entries":
        entries = get_user_entries(user_id, limit=20)
        if not entries:
            await query.edit_message_text("Нет записей.", reply_markup=back_keyboard())
            return
        text = "📋 *Ваши последние записи:*\n\n"
        keyboard = []
        for e in entries:
            cat_name = CATEGORIES[e['category']]['name']
            unit = get_main_currency(user_id) if e['category'] == 'main' else ('т' if e['category'] == 'tonnage' else '₽')
            text += f"ID {e['id']}: {cat_name} – {e['amount']:.2f} {unit} – {e['date']}\n"
            keyboard.append([
                InlineKeyboardButton(f"✏️ {e['id']} ({cat_name})", callback_data=f"edit_{e['id']}"),
                InlineKeyboardButton(f"🗑 {e['id']}", callback_data=f"del_{e['id']}")
            ])
        keyboard.append([InlineKeyboardButton("◀️ Назад", callback_data="back_to_menu")])
        await query.edit_message_text(text, parse_mode='Markdown', reply_markup=InlineKeyboardMarkup(keyboard))
        return

    # Редактирование
    if data.startswith("edit_"):
        entry_id = int(data.split("_")[1])
        entry = get_income_entry(entry_id)
        if not entry or entry['user_id'] != user_id:
            await query.edit_message_text("Запись не найдена.", reply_markup=back_keyboard())
            return
        context.user_data['edit_id'] = entry_id
        context.user_data['awaiting'] = 'edit'
        context.user_data['step'] = 'amount'
        await query.edit_message_text(
            f"✏️ *Редактирование записи #{entry_id}*\n"
            f"Категория: {CATEGORIES[entry['category']]['name']}\n"
            f"Текущая сумма: {entry['amount']}\n"
            f"Текущее описание: {entry['description'] or '—'}\n\n"
            f"Введите новую сумму:",
            parse_mode='Markdown',
            reply_markup=back_keyboard()
        )
        return

    # Удаление
    if data.startswith("del_"):
        entry_id = int(data.split("_")[1])
        entry = get_income_entry(entry_id)
        if entry and entry['user_id'] == user_id:
            delete_income(entry_id)
            await query.edit_message_text(f"✅ Запись #{entry_id} удалена.", reply_markup=back_keyboard())
        else:
            await query.edit_message_text("Запись не найдена.", reply_markup=back_keyboard())
        return

    # Настройки
    if data == "settings":
        curr = get_main_currency(user_id)
        keyboard = [
            [InlineKeyboardButton("₽ Рубль", callback_data="set_currency_₽"),
             InlineKeyboardButton("$ Доллар", callback_data="set_currency_$")],
            [InlineKeyboardButton("€ Евро", callback_data="set_currency_€"),
             InlineKeyboardButton("◀️ Назад", callback_data="back_to_menu")]
        ]
        await query.edit_message_text(
            f"⚙️ *Настройки*\n\nВалюта основного дохода: `{curr}`\n\nВыберите новую:",
            parse_mode='Markdown',
            reply_markup=InlineKeyboardMarkup(keyboard)
        )
        return

    if data.startswith("set_currency_"):
        new_curr = data.split("_")[2]
        set_main_currency(user_id, new_curr)
        await query.edit_message_text(f"✅ Валюта изменена на `{new_curr}`.", parse_mode='Markdown', reply_markup=back_keyboard())
        return

    if data == "export":
        await query.edit_message_text("⏳ Формирую Excel-файл...")
        excel_file = await export_to_excel(user_id)
        await query.message.reply_document(document=excel_file, filename="earnings.xlsx")
        await query.edit_message_text("✅ Экспорт завершён.", reply_markup=main_menu_keyboard())
        return

    if data == "chart_menu":
        await query.edit_message_text("Выберите период для графика:", reply_markup=chart_menu_keyboard())
        return

    if data.startswith("chart_"):
        period = data.split("_")[1]
        await query.edit_message_text(f"⏳ Строю график за {period}...")
        chart_img = await generate_chart(user_id, period)
        if chart_img:
            await query.message.reply_photo(photo=chart_img, caption=f"📈 Динамика за {period}")
            await query.edit_message_text("График готов.", reply_markup=main_menu_keyboard())
        else:
            await query.edit_message_text("Недостаточно данных для графика.", reply_markup=back_keyboard())
        return

    if data == "goals_menu":
        goals = get_user_goals(user_id)
        if not goals:
            await query.edit_message_text("У вас пока нет целей. Добавьте новую цель.", reply_markup=back_keyboard())
        else:
            text = "🎯 *Ваши цели:*\n\n"
            for g in goals:
                status = "✅ достигнута" if g['achieved'] else "⏳ активна"
                type_name = "Тонаж (т)" if g['type'] == 'tonnage' else f"Доход ({get_main_currency(user_id)})"
                text += f"• {type_name}: {g['threshold']} — {status}\n"
            await query.edit_message_text(text, parse_mode='Markdown', reply_markup=goals_menu_keyboard(goals))
        return

    if data == "new_goal":
        await query.edit_message_text(
            "🎯 *Новая цель*\n\nВыберите тип цели:",
            parse_mode='Markdown',
            reply_markup=InlineKeyboardMarkup([
                [InlineKeyboardButton("🚛 Тонаж (тонны)", callback_data="goal_type_tonnage")],
                [InlineKeyboardButton("💰 Доход (деньги)", callback_data="goal_type_money")],
                [InlineKeyboardButton("◀️ Назад", callback_data="back_to_menu")]
            ])
        )
        return

    if data.startswith("goal_type_"):
        goal_type = data.split("_")[2]
        context.user_data['awaiting'] = 'goal'
        context.user_data['goal_type'] = goal_type
        unit = "т" if goal_type == "tonnage" else get_main_currency(user_id)
        await query.edit_message_text(
            f"Введите пороговое значение для цели ({unit}):\nНапример: `100`",
            parse_mode='Markdown',
            reply_markup=back_keyboard()
        )
        return

    if data.startswith("goal_del_"):
        goal_id = int(data.split("_")[2])
        delete_goal(goal_id)
        await query.edit_message_text("✅ Цель удалена.", reply_markup=back_keyboard())
        return

    if data == "back_to_menu":
        context.user_data.clear()
        await query.edit_message_text("Главное меню:", reply_markup=main_menu_keyboard())
        return

# --- Обработчик текстовых сообщений ---
async def text_handler(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    user_id = update.effective_user.id
    text = update.message.text.strip()
    state = context.user_data.get('awaiting')
    step = context.user_data.get('step')

    # Добавление: ввод суммы
    if state in ('main', 'tonnage', 'extra') and step == 'amount':
        try:
            amount = float(text.replace(',', '.'))
            if amount <= 0:
                raise ValueError
            context.user_data['amount'] = amount
            context.user_data['step'] = 'desc'
            await update.message.reply_text(
                f"✅ Количество: {amount:.2f}\n\nВведите описание (или «-», чтобы пропустить):",
                reply_markup=back_keyboard()
            )
        except ValueError:
            await update.message.reply_text("❌ Введите положительное число. Попробуйте ещё раз:")
        return

    # Добавление: ввод описания -> сохранение
    if state in ('main', 'tonnage', 'extra') and step == 'desc':
        description = None if text == '-' else text
        category = state
        amount = context.user_data['amount']
        date_str = datetime.now().strftime('%Y-%m-%d')
        
        if category == 'tonnage':
            # Добавляем тоннаж
            tonnage_id = add_income(user_id, 'tonnage', amount, description, date_str, linked_id=None)
            # Создаём связанный основной доход (1 т = 1000 руб)
            main_amount = amount * 1000
            main_desc = f"Автоматически за тоннаж: {amount} т -> {main_amount} руб"
            main_id = add_income(user_id, 'main', main_amount, main_desc, date_str, linked_id=tonnage_id)
            # Обновляем запись тоннажа, чтобы она ссылалась на основной доход
            conn = sqlite3.connect(DB_PATH)
            c = conn.cursor()
            c.execute('UPDATE income SET linked_id = ? WHERE id = ?', (main_id, tonnage_id))
            conn.commit()
            conn.close()
            
            await update.message.reply_text(
                f"✅ *Добавлен тоннаж и автоматически начислен основной доход!*\n\n"
                f"🚛 *Тонаж:* {amount:.2f} т\n"
                f"📝 Описание: {description or '—'}\n"
                f"💰 *Начислено в основной доход:* {main_amount:.2f} {get_main_currency(user_id)}\n"
                f"📅 Дата: {date_str}",
                parse_mode='Markdown',
                reply_markup=main_menu_keyboard()
            )
        else:
            # Обычное добавление (основной доход вручную или халтура)
            entry_id = add_income(user_id, category, amount, description, date_str)
            unit = get_main_currency(user_id) if category == 'main' else ('т' if category == 'tonnage' else '₽')
            cat_name = CATEGORIES[category]['name']
            await update.message.reply_text(
                f"✅ *Запись #{entry_id} добавлена!*\n\n"
                f"📂 {cat_name}\n"
                f"📊 {amount:.2f} {unit}\n"
                f"📝 Описание: {description or '—'}\n"
                f"📅 Дата: {date_str}",
                parse_mode='Markdown',
                reply_markup=main_menu_keyboard()
            )
        
        check_and_notify_goals(user_id, context)
        context.user_data.clear()
        return

    # Редактирование: ввод суммы
    if state == 'edit' and step == 'amount':
        try:
            amount = float(text.replace(',', '.'))
            if amount <= 0:
                raise ValueError
            context.user_data['edit_amount'] = amount
            context.user_data['step'] = 'desc'
            await update.message.reply_text(
                f"Новая сумма: {amount:.2f}\n\nТеперь введите новое описание (или «-», чтобы оставить текущее):",
                reply_markup=back_keyboard()
            )
        except ValueError:
            await update.message.reply_text("❌ Введите корректную сумму. Попробуйте ещё раз:")
        return

    if state == 'edit' and step == 'desc':
        entry_id = context.user_data.get('edit_id')
        entry = get_income_entry(entry_id)
        if not entry:
            await update.message.reply_text("Запись не найдена.", reply_markup=main_menu_keyboard())
            context.user_data.clear()
            return
        new_amount = context.user_data.get('edit_amount')
        new_desc = entry['description'] if text == '-' else text
        
        # Обновляем саму запись
        update_income(entry_id, new_amount, new_desc)
        
        # Если это тоннаж, то обновляем связанный основной доход
        if entry['category'] == 'tonnage':
            conn = sqlite3.connect(DB_PATH)
            c = conn.cursor()
            c.execute('SELECT linked_id FROM income WHERE id = ?', (entry_id,))
            row = c.fetchone()
            if row and row[0]:
                main_id = row[0]
                new_main_amount = new_amount * 1000
                new_main_desc = f"Автоматически за тоннаж: {new_amount} т -> {new_main_amount} руб"
                c.execute('UPDATE income SET amount = ?, description = ? WHERE id = ?',
                          (new_main_amount, new_main_desc, main_id))
                conn.commit()
            conn.close()
            await update.message.reply_text(
                f"✅ *Тонаж обновлён!*\n\n"
                f"Новый тоннаж: {new_amount} т\n"
                f"Основной доход автоматически пересчитан: {new_amount * 1000} {get_main_currency(user_id)}",
                reply_markup=main_menu_keyboard()
            )
        else:
            await update.message.reply_text(
                f"✅ Запись #{entry_id} обновлена.\nНовая сумма: {new_amount}\nНовое описание: {new_desc or '—'}",
                reply_markup=main_menu_keyboard()
            )
        
        check_and_notify_goals(user_id, context)
        context.user_data.clear()
        return

    # Добавление цели
    if state == 'goal':
        try:
            threshold = float(text.replace(',', '.'))
            if threshold <= 0:
                raise ValueError
            goal_type = context.user_data.get('goal_type')
            add_goal(user_id, goal_type, threshold)
            await update.message.reply_text(
                f"✅ Цель добавлена!\nТип: {'Тоннаж' if goal_type == 'tonnage' else 'Доход'}\nПорог: {threshold}",
                reply_markup=main_menu_keyboard()
            )
            context.user_data.clear()
        except ValueError:
            await update.message.reply_text("❌ Введите положительное число. Попробуйте ещё раз:")
        return

    # Если нет активного состояния, напоминаем меню
    await update.message.reply_text("Используйте кнопки меню для работы.", reply_markup=main_menu_keyboard())

# --- Команды ---
async def cancel(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    context.user_data.clear()
    await update.message.reply_text("Операция отменена.", reply_markup=main_menu_keyboard())

async def help_command(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    await update.message.reply_text(
        "🤖 *Помощь*\n\n• /start – главное меню\n• /export – выгрузить данные в Excel\n• /chart – показать график\n• /cancel – отменить текущую операцию\n• /help – эта справка\n\n"
        "При добавлении тоннажа автоматически начисляется основной доход из расчёта 1 т = 1000 руб.",
        parse_mode='Markdown'
    )

async def export_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    await update.message.reply_text("⏳ Формирую Excel...")
    excel_file = await export_to_excel(user_id)
    await update.message.reply_document(document=excel_file, filename="earnings.xlsx")

async def chart_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("Выберите период для графика:", reply_markup=chart_menu_keyboard())

# --- Запуск ---
def main():
    init_db()
    app = Application.builder().token(TOKEN).build()

    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("help", help_command))
    app.add_handler(CommandHandler("export", export_command))
    app.add_handler(CommandHandler("chart", chart_command))
    app.add_handler(CommandHandler("cancel", cancel))

    app.add_handler(CallbackQueryHandler(button_handler))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, text_handler))

    logger.info("Бот запущен")
    app.run_polling()

if __name__ == '__main__':
    main()