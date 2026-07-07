#!/bin/bash



echo "========================================="

echo "🔄 بازسازی کامل پروژه AsisCheng"

echo "========================================="



mkdir -p AssistantProject

cd AssistantProject

mkdir -p modules

mkdir -p outputs



cat > main.py << 'MAIN'

#!/usr/bin/env python3

# -*- coding: utf-8 -*-

import os, sys, json, threading

from kivy.app import App

from kivy.uix.boxlayout import BoxLayout

from kivy.uix.button import Button

from kivy.uix.label import Label

from kivy.uix.textinput import TextInput

from kivy.uix.scrollview import ScrollView

from kivy.uix.filechooser import FileChooserListView

from kivy.uix.popup import Popup

from kivy.uix.camera import Camera

from kivy.core.window import Window



Window.size = (400, 700)

sys.path.append(os.path.dirname(os.path.abspath(__file__)))



try:

    from modules.qa import QA

    from modules.bug_reporter import BugReporter

    from modules.updater import check_for_updates

    from modules.file_reader import FileReader

except ImportError as e:

    print(f"⚠️ خطا در بارگذاری ماژول‌ها: {e}")

    class DummyModule:

        def ask(self, q): return "ماژول در دسترس نیست"

        def interact(self): return "ماژول در دسترس نیست"

    QA = DummyModule

    BugReporter = DummyModule

    check_for_updates = lambda: False

    FileReader = DummyModule



class MainScreen(BoxLayout):

    def __init__(self, **kwargs):

        super().__init__(orientation='vertical', **kwargs)

        self.reporter = BugReporter()

        try:

            self.qa = QA()

            self.file_reader = FileReader()

        except Exception as e:

            self.reporter.report_error(str(e), "ماژول‌ها")

            raise

        self.build_ui()

        threading.Thread(target=self._check_update_background, daemon=True).start()



    def _check_update_background(self):

        try:

            if check_for_updates():

                self.status_label.text = "🔄 به‌روزرسانی جدید موجود است!"

        except:

            pass



    def build_ui(self):

        header = BoxLayout(size_hint_y=0.1, padding=10, spacing=10)

        header.add_widget(Label(text="🧠 AsisCheng", font_size=24, color=(0.2, 0.6, 1, 1)))

        self.add_widget(header)



        input_box = BoxLayout(size_hint_y=0.12, padding=10, spacing=10)

        self.question_input = TextInput(hint_text="سوال خود را بپرسید...", multiline=False, font_size=16)

        send_btn = Button(text="ارسال", size_hint_x=0.2, background_color=(0.2, 0.6, 1, 1))

        send_btn.bind(on_press=self.ask_question)

        input_box.add_widget(self.question_input)

        input_box.add_widget(send_btn)

        self.add_widget(input_box)



        tools = BoxLayout(size_hint_y=0.1, padding=5, spacing=5)

        btn_camera = Button(text="📷 دوربین")

        btn_file = Button(text="📂 فایل")

        btn_clear = Button(text="🗑️ پاک")

        btn_camera.bind(on_press=self.open_camera)

        btn_file.bind(on_press=self.open_file_chooser)

        btn_clear.bind(on_press=self.clear_output)

        tools.add_widget(btn_camera)

        tools.add_widget(btn_file)

        tools.add_widget(btn_clear)

        self.add_widget(tools)



        self.output_area = ScrollView()

        self.output_label = Label(

            text="👋 به AsisCheng خوش آمدید!\nسوال خود را بپرسید.",

            size_hint_y=None,

            font_size=16,

            color=(0.9, 0.9, 0.9, 1),

            markup=True,

            halign='left',

            valign='top'

        )

        self.output_label.bind(size=self.output_label.setter('text_size'))

        self.output_area.add_widget(self.output_label)

        self.add_widget(self.output_area)



        self.status_label = Label(text="✅ آماده", size_hint_y=0.05, font_size=12, color=(0.5, 0.5, 0.5, 1))

        self.add_widget(self.status_label)



    def ask_question(self, instance):

        try:

            question = self.question_input.text.strip()

            if not question:

                return

            self.question_input.text = ""

            self.output_label.text += f"\n\n❓ {question}"

            answer = self.qa.ask(question)

            self.output_label.text += f"\n🤖 {answer}"

            self.output_label.text += "\n" + "-"*40

        except Exception as e:

            self.output_label.text += f"\n⚠️ خطا: {str(e)}"



    def open_camera(self, instance):

        try:

            layout = BoxLayout(orientation='vertical', padding=10, spacing=10)

            camera = Camera(resolution=(640, 480), play=True)

            layout.add_widget(camera)

            btn_box = BoxLayout(size_hint_y=0.15, spacing=10)

            capture_btn = Button(text="📸 عکس بگیر")

            close_btn = Button(text="❌ بستن")

            def capture(instance):

                camera.export_to_png("/tmp/captured.png")

                self.output_label.text += "\n📸 عکس گرفته شد: /tmp/captured.png"

                self.status_label.text = "✅ عکس ذخیره شد"

            capture_btn.bind(on_press=capture)

            close_btn.bind(on_press=lambda x: popup.dismiss())

            btn_box.add_widget(capture_btn)

            btn_box.add_widget(close_btn)

            layout.add_widget(btn_box)

            popup = Popup(title="📷 دوربین", content=layout, size_hint=(0.9, 0.8))

            popup.open()

        except Exception as e:

            self.output_label.text += f"\n⚠️ خطا در دوربین: {str(e)}"



    def open_file_chooser(self, instance):

        try:

            layout = BoxLayout(orientation='vertical', padding=10, spacing=10)

            filechooser = FileChooserListView()

            layout.add_widget(filechooser)

            btn_box = BoxLayout(size_hint_y=0.15, spacing=10)

            open_btn = Button(text="📂 باز کردن")

            close_btn = Button(text="❌ بستن")

            def open_file(instance):

                if filechooser.selection:

                    path = filechooser.selection[0]

                    content = self.file_reader.read_file(path)

                    self.output_label.text += f"\n📄 فایل: {path}\n{content[:500]}..."

                    self.status_label.text = f"✅ فایل باز شد: {os.path.basename(path)}"

                    popup.dismiss()

            open_btn.bind(on_press=open_file)

            close_btn.bind(on_press=lambda x: popup.dismiss())

            btn_box.add_widget(open_btn)

            btn_box.add_widget(close_btn)

            layout.add_widget(btn_box)

            popup = Popup(title="📂 انتخاب فایل", content=layout, size_hint=(0.9, 0.8))

            popup.open()

        except Exception as e:

            self.output_label.text += f"\n⚠️ خطا در فایل: {str(e)}"



    def clear_output(self, instance):

        self.output_label.text = "👋 خوش آمدید! سوال خود را بپرسید."

        self.status_label.text = "✅ پاک شد"



    def on_touch_down(self, touch):

        if self.question_input.focus:

            self.question_input.focus = False

        return super().on_touch_down(touch)



class AsisChengApp(App):

    def build(self):

        from kivy.utils import get_color_from_hex

        from kivy.core.window import Window

        Window.clearcolor = get_color_from_hex('#1a1a2e')

        return MainScreen()



if __name__ == '__main__':

    AsisChengApp().run()

MAIN



cat > cli.py << 'CLI'

#!/usr/bin/env python3

import json, os, sys

from datetime import datetime



class KnowledgeBase:

    def __init__(self, db_file='knowledge_base.json'):

        self.db_file = db_file

        self.data = self._load()

    def _load(self):

        if os.path.exists(self.db_file):

            try:

                with open(self.db_file, 'r', encoding='utf-8') as f:

                    return json.load(f)

            except:

                return {}

        return {}

    def _save(self):

        with open(self.db_file, 'w', encoding='utf-8') as f:

            json.dump(self.data, f, ensure_ascii=False, indent=2)

    def learn(self, question, answer):

        self.data[question.strip().lower()] = answer.strip()

        self._save()

        return f"✅ دانش '{question}' ذخیره شد."

    def ask(self, question):

        q = question.strip().lower()

        if q in self.data:

            return self.data[q]

        for key, value in self.data.items():

            if q in key or key in q:

                return f"💡 شاید منظور شما این باشد:\n{key} → {value}"

        return "❌ پاسخی برای این سوال پیدا نشد.\nبرای یادگیری: یادگیری:سوال:پاسخ"

    def show_all(self):

        if not self.data:

            return "📭 هیچ دانشی ذخیره نشده است."

        result = "🧠 دانش ذخیره‌شده:\n" + "-"*40 + "\n"

        for q, a in self.data.items():

            result += f"❓ {q}\n📝 {a}\n\n"

        return result



class Assistant:

    def __init__(self):

        self.kb = KnowledgeBase()

        self.running = True

    def process(self, user_input):

        if user_input.startswith("یادگیری:"):

            parts = user_input.split(":", 2)

            if len(parts) == 3:

                _, q, a = parts

                return self.kb.learn(q, a)

            return "⚠️ فرمت: یادگیری:سوال:پاسخ"

        if user_input.lower() in ['خروج', 'exit', 'quit']:

            self.running = False

            return "👋 خداحافظ!"

        if user_input.lower() in ['همه', 'list']:

            return self.kb.show_all()

        return self.kb.ask(user_input)

    def run(self):

        print("\n" + "="*50)

        print("🧠 **دستیار هوشمند AsisCheng (CLI)**")

        print("="*50)

        print("📚 دستورات:")

        print("  سوال بپرسید → دریافت پاسخ")

        print("  یادگیری:سوال:پاسخ → ذخیره دانش جدید")

        print("  همه → نمایش تمام دانش‌ها")

        print("  خروج → پایان برنامه")

        print("="*50 + "\n")

        while self.running:

            try:

                user_input = input("❓ ").strip()

                if not user_input:

                    continue

                response = self.process(user_input)

                if response:

                    print(f"🤖 {response}\n")

            except KeyboardInterrupt:

                print("\n👋 خداحافظ!")

                break

            except Exception as e:

                print(f"⚠️ خطا: {e}\n")



if __name__ == "__main__":

    Assistant().run()

CLI



cat > buildozer.spec << 'SPEC'

[app]

title = AsisCheng

package.name = myapp

package.domain = org.test

source.dir = .

source.include_exts = py,png,jpg,kv,atlas

version = 1.0.0

version.code = 1

requirements = python3,kivy==2.2.1,kivymd==1.1.1,pyjnius,Pillow,numpy,scipy,matplotlib

android.api = 30

android.minapi = 29

android.ndk = 23b

android.archs = armeabi-v7a, arm64-v8a

android.permissions = INTERNET,CAMERA,RECORD_AUDIO,READ_EXTERNAL_STORAGE,WRITE_EXTERNAL_STORAGE

fullscreen = 0

orientation = portrait

window.size = 800x600

osx.python_version = 3

osx.kivy_version = 2.2.1

android.gradle_dependencies =

SPEC



mkdir -p modules

cat > modules/qa.py << 'QA'

import json, os

class QA:

    def __init__(self, db_file='knowledge_base.json'):

        self.db_file = db_file

        self.knowledge = self.load_knowledge()

    def load_knowledge(self):

        if os.path.exists(self.db_file):

            with open(self.db_file, 'r', encoding='utf-8') as f:

                return json.load(f)

        return {}

    def save_knowledge(self):

        with open(self.db_file, 'w', encoding='utf-8') as f:

            json.dump(self.knowledge, f, ensure_ascii=False, indent=2)

    def learn(self, question, answer):

        self.knowledge[question.lower()] = answer

        self.save_knowledge()

        return "✅ دانش جدید ذخیره شد."

    def ask(self, question):

        q = question.lower()

        return self.knowledge.get(q, "❌ پاسخی برای این سوال پیدا نشد.")

    def interact(self):

        return "ماژول QA آماده است."

QA



cat > modules/bug_reporter.py << 'BUG'

import os, traceback, json

from datetime import datetime

class BugReporter:

    def __init__(self, log_file='bug_report.log', max_entries=100):

        self.log_file = log_file

        self.max_entries = max_entries

        self._ensure_file()

    def _ensure_file(self):

        if not os.path.exists(self.log_file):

            with open(self.log_file, 'w') as f:

                json.dump([], f)

    def _read_logs(self):

        try:

            with open(self.log_file, 'r') as f:

                return json.load(f)

        except:

            return []

    def _write_logs(self, logs):

        with open(self.log_file, 'w') as f:

            json.dump(logs[-self.max_entries:], f, indent=2)

    def report_error(self, error_msg, context=None):

        logs = self._read_logs()

        entry = {

            "timestamp": datetime.now().isoformat(),

            "error": error_msg,

            "context": context or "",

            "traceback": traceback.format_exc()

        }

        logs.append(entry)

        self._write_logs(logs)

BUG



cat > modules/updater.py << 'UPD'

import os, sys, json, urllib.request, hashlib, zipfile, tempfile

UPDATE_SERVER = "https://your-server.com/updates/"

CURRENT_VERSION = "1.0.0"

def get_latest_version():

    try:

        url = UPDATE_SERVER + "version.json"

        with urllib.request.urlopen(url, timeout=5) as response:

            data = json.loads(response.read().decode())

            return data.get('version'), data.get('download_url'), data.get('checksum')

    except:

        return None, None, None

def check_for_updates():

    version, url, checksum = get_latest_version()

    if not version or version == CURRENT_VERSION:

        return False

    return True

UPD



cat > modules/file_reader.py << 'FILEREAD'

import os

class FileReader:

    def read_file(self, path):

        try:

            with open(path, 'r', encoding='utf-8') as f:

                return f.read()

        except:

            return "خطا در خواندن فایل"

    def interact(self):

        return "ماژول خواندن فایل: از منوی اصلی استفاده کنید."

FILEREAD



echo "✅ تمام فایل‌های پروژه با موفقیت ساخته شدند!"

echo ""

echo "📂 ساختار پروژه:"

echo "  ├── main.py"

echo "  ├── cli.py"

echo "  ├── buildozer.spec"

echo "  └── modules/"

echo "      ├── qa.py"

echo "      ├── bug_reporter.py"

echo "      ├── updater.py"

echo "      └── file_reader.py"

echo ""

echo "🚀 برای اجرای نسخه CLI: python cli.py"

echo "📦 برای ساخت APK: buildozer android debug deploy run"


