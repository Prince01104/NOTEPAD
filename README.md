from kivy.lang import Builder
from kivy.uix.screenmanager import Screen,
ScreenManager
from kivy.properties import StringProperty
from kivymd.app import MDApp
from kivymd.uix.list import OneLineListltem,
MDList
from kivymd.uix.menu import MDDropdownMenu
from kivymd.uix.button import MDFloatingActionButton
from kivymd.uix.textfield import MDTextField
from kivymd.uix.dialog import MDDialog
from kivymd.uix.card import MDCard
import sqlite3

KV = '''
ScreenManager:
    HomeScreen:
    AddNoteScreen:

<HomeScreen>:
    name: "home"
    
    MDBoxLayout:
        orientation: "vertical"

        MDTopAppBar:
            title: "Notepad"
            left_action_items: [["menu", lambda x: nav_drawer.set_state("open")]]
            right_action_items: [["magnify", lambda x: app.search_notes()], ["theme-light-dark", lambda x: app.toggle_dark_mode()]]

        ScrollView:
            MDList:
                id: notes_list
        
        MDFloatingActionButton:
            icon: "plus"
            pos_hint: {"center_x": 0.9, "center_y": 0.1}
            on_release: app.change_screen("add_note")

    MDNavigationDrawer:
        id: nav_drawer
        MDNavigationDrawerMenu:
            MDNavigationDrawerItem:
                text: "All Notes"
                icon: "notebook"
                on_release: app.filter_notes("all")
            MDNavigationDrawerItem:
                text: "Work"
                icon: "briefcase"
                on_release: app.filter_notes("work")
            MDNavigationDrawerItem:
                text: "Personal"
                icon: "account"
                on_release: app.filter_notes("personal")
            MDNavigationDrawerItem:
                text: "Ideas"
                icon: "lightbulb"
                on_release: app.filter_notes("ideas")

<AddNoteScreen>:
    name: "add_note"
    
    MDBoxLayout:
        orientation: "vertical"
        padding: "20dp"

        MDTextField:
            id: title_input
            hint_text: "Enter note title"
        
        MDTextField:
            id: content_input
            hint_text: "Enter your note..."
            multiline: True

        MDBoxLayout:
            spacing: "10dp"
            MDFloatingActionButton:
                icon: "content-save"
                pos_hint: {"center_x": 0.5}
                on_release: app.save_note()

            MDFloatingActionButton:
                icon: "arrow-left"
                pos_hint: {"center_x": 0.5}
                on_release: app.change_screen("home")
'''

class HomeScreen(Screen):
    pass

class AddNoteScreen(Screen):
    pass

class NoteApp(MDApp):
    db_path = "notes.db"

    def build(self):
        self.theme_cls.primary_palette = "Blue"
        self.theme_cls.theme_style = "Light"
        self.create_database()
        return Builder.load_string(KV)

    def create_database(self):
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS notes (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                title TEXT,
                content TEXT,
                category TEXT
            )
        ''')
        conn.commit()
        conn.close()

    def change_screen(self, screen_name):
        self.root.current = screen_name

    def save_note(self):
        title = self.root.get_screen("add_note").ids.title_input.text
        content = self.root.get_screen("add_note").ids.content_input.text
        if title and content:
            conn = sqlite3.connect(self.db_path)
            cursor = conn.cursor()
            cursor.execute("INSERT INTO notes (title, content, category) VALUES (?, ?, ?)", (title, content, "all"))
            conn.commit()
            conn.close()
            self.display_notes()
            self.change_screen("home")

    def display_notes(self):
        notes_list = self.root.get_screen("home").ids.notes_list
        notes_list.clear_widgets()
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        cursor.execute("SELECT id, title, content FROM notes")
        notes = cursor.fetchall()
        conn.close()
        
        for note in notes:
            note_card = MDCard(
                size_hint=(1, None),
                height="120dp",
                padding="10dp",
                elevation=3,
                on_release=lambda x, note_id=note[0]: self.edit_note(note_id),
            )
            note_card.add_widget(
                OneLineListItem(text=f"{note[1]} - {note[2][:50]}...")
            )
            notes_list.add_widget(note_card)

    def edit_note(self, note_id):
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        cursor.execute("SELECT title, content FROM notes WHERE id=?", (note_id,))
        note = cursor.fetchone()
        conn.close()

        if note:
            self.root.get_screen("add_note").ids.title_input.text = note[0]
            self.root.get_screen("add_note").ids.content_input.text = note[1]
            self.change_screen("add_note")
            self.note_to_edit = note_id  # Store note ID for updating

    def delete_note(self, note_id):
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        cursor.execute("DELETE FROM notes WHERE id=?", (note_id,))
        conn.commit()
        conn.close()
        self.display_notes()

    def filter_notes(self, category):
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        if category == "all":
            cursor.execute("SELECT id, title, content FROM notes")
        else:
            cursor.execute("SELECT id, title, content FROM notes WHERE category=?", (category,))
        notes = cursor.fetchall()
        conn.close()

        notes_list = self.root.get_screen("home").ids.notes_list
        notes_list.clear_widgets()
        
        for note in notes:
            note_card = MDCard(
                size_hint=(1, None),
                height="120dp",
                padding="10dp",
                elevation=3,
                on_release=lambda x, note_id=note[0]: self.edit_note(note_id),
            )
            note_card.add_widget(
                OneLineListItem(text=f"{note[1]} - {note[2][:50]}...")
            )
            notes_list.add_widget(note_card)

    def search_notes(self):
        dialog = MDDialog(
            title="Search Notes",
            type="custom",
            content_cls=MDTextField(hint_text="Enter search term"),
            buttons=[
                MDFloatingActionButton(icon="magnify", on_release=lambda x: self.perform_search(dialog.content_cls.text))
            ],
        )
        dialog.open()

    def perform_search(self, search_term):
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        cursor.execute("SELECT id, title, content FROM notes WHERE title LIKE ? OR content LIKE ?", (f"%{search_term}%", f"%{search_term}%"))
        notes = cursor.fetchall()
        conn.close()

        notes_list = self.root.get_screen("home").ids.notes_list
        notes_list.clear_widgets()

        for note in notes:
            note_card = MDCard(
                size_hint=(1, None),
                height="120dp",
                padding="10dp",
                elevation=3,
                on_release=lambda x, note_id=note[0]: self.edit_note(note_id),
            )
            note_card.add_widget(
                OneLineListItem(text=f"{note[1]} - {note[2][:50]}...")
            )
            notes_list.add_widget(note_card)

    def toggle_dark_mode(self):
        self.theme_cls.theme_style = "Dark" if self.theme_cls.theme_style == "Light" else "Light"


if __name__ == "__main__":
    NoteApp().run()
