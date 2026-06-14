import telebot
from telebot.types import InlineKeyboardButton, InlineKeyboardMarkup
import requests
import json
import os
import config
bot = telebot.TeleBot(config.token)
quiz_file = "quizzes.json"
scores_file = "scores.json"
user_state = {}
def load_data(file):
    if not os.path.exists(file):
        return{}
    with open(file, "r", encoding="utf-8") as f:
        return json.load(f)
def save_data(file, data):
    with open(file, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)
quizzes = load_data(quiz_file)
scores = load_data(scores_file)
def generate_quiz(topic):
    prompt = f"""
Создай 5 вопросов по теме {topic}.

Верни только JSON:

[
  {{
    "question":"...",
    "options":["A","B","C","D"],
    "correct":0
  }}
]
"""

    r = requests.post(
        "https://openrouter.ai/api/v1/chat/completions",
        headers={
            "Authorization": f"Bearer {config.openrouter_key}",
            "Content-Type": "application/json"
        },
        json={
            "model": "openrouter/free",
            "messages": [
                {"role": "user", "content": prompt}
            ]
        }
    )

    data = r.json()

    if "choices" not in data:
        raise Exception(
            f"OpenRouter error:\n{json.dumps(data, indent=2)}"
        )

    text = data["choices"][0]["message"]["content"]

    text = text.replace("```json", "")
    text = text.replace("```", "")

    return json.loads(text)
@bot.message_handler(commands=["start"])
def start(message):
    bot.send_message(message.chat.id, "Hello, I am a trivia bot. Ask me to give you a trivia question on a specific topic, and you'll have to answer it correctly to get a point. /quiz to start the quiz, /rating to see the rating of correctly answered questions, and /list to see what type of quizzes there are.")

@bot.message_handler(commands=["quiz"])
def quiz(message):
    user_state[message.chat.id]="theme"
    bot.send_message(message.chat.id, "Type in the topic of the trivia question.")
    
@bot.message_handler(func=lambda message:user_state.get(message.chat.id)=="theme")
def topic_input(message):
    topic=message.text
    msg=bot.send_message(message.chat.id, "Generate the question...")
    try:
        q=generate_quiz(topic)
        quiz_id=str(len(quizzes)+1)
        quizzes[quiz_id] = {
                "author": message.from_user.id,
                "topic": topic,
                "questions": q
            }
        save_data(quiz_file, quizzes)
        bot.edit_message_text(f"Quiz create \n id: {quiz_id}", msg.chat.id, msg.message_id)
    except Exception as e:
         bot.edit_message_text(
            f"Ошибка:\n{e}",
            message.chat.id,
            msg.message_id
        )
    user_state.pop(message.chat.id)
@bot.message_handler(commands=["list"])
def list_quiz(message):
    if not quizzes:
        bot.send_message(message.chat.id, "No quizzes")
        return
    kb=InlineKeyboardMarkup()
    for qid in quizzes:
        kb.add(InlineKeyboardButton(quizzes[qid]["topic"], callback_data=f"play_{qid}"))
    bot.send_message(message.chat.id, "Available quizzes", reply_markup=kb)

sessions={}
def send_question(chat, qid):
    s=sessions[chat]
    q=quizzes[qid]["questions"][s["index"]]
    kb=InlineKeyboardMarkup()
    for i, opt in enumerate(q["options"]):
        kb.add(InlineKeyboardButton(opt, callback_data=f"ans_{qid}_{i}"))
    bot.send_message(chat, q["question"], reply_markup=kb)
@bot.callback_query_handler(func=lambda c: c.data.startswith("play_"))
def play(c):
    qid = c.data.split("_")[1]
    sessions[c.message.chat.id] = {
        "qid": qid,
        "index": 0,
        "score": 0
    }
    send_question(c.message.chat.id, qid)
@bot.callback_query_handler(func= lambda c: c.data.startswith("ans_"))
def answer(c):
        _, qid, user_ans = c.data.split("_")

        s = sessions[c.message.chat.id]

        q = quizzes[qid]["questions"][s["index"]]

        if int(user_ans) == q["correct"]:
            s["score"] += 1
        s["index"] +=1
        if s["index"] >=len(quizzes[qid]["questions"]):
            uid=str(c.from_user.id)
            scores[uid]=(scores.get(uid, 0)+s["score"])
            save_data(scores_file, scores)
            bot.send_message(c.message.chat.id, f"""
🏁 Ready

Points: {s['score']}
""")
            del sessions[c.message.chat.id]
            return 
        send_question(c.message.chat.id, qid)
@bot.message_handler(commands=["rating"])
def rating(message):
    if not scores:
        bot.send_message(message.chat.id, "Empty rating")
        return
    txt="rating\n\n"
    sorted_users= sorted(scores.items(), key=lambda x: x[1], reverse=True)
    for uid, pts in sorted_users:
        txt+=(f"{uid}:"
              f"{pts} points\n")
    bot.send_message(message.chat.id, txt)
bot.infinity_polling()

