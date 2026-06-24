import tkinter as tk
from tkinter import ttk, messagebox, filedialog
import mysql.connector
import csv  # Added for exporting
from datetime import datetime

# --- DATABASE CONNECTION ---
def get_db_connection():
    try:
        return mysql.connector.connect(
            host="localhost",
            user="root",
            password="    OPEN",  # Change this
            database="event_planner"
        )
    except mysql.connector.Error as err:
        messagebox.showerror("Database Error", f"Connection failed: {err}")
        return None

class EventPlannerApp:
    def __init__(self, root):
        self.root = root
        self.root.title("Professional Event Planner")
        self.root.geometry("1000x600")
        self.current_user_id = None
        self.show_login()

    def clear_window(self):
        for widget in self.root.winfo_children():
            widget.destroy()

    def show_login(self):
        self.clear_window()
        frame = tk.Frame(self.root, padx=30, pady=30, relief="groove", borderwidth=2)
        frame.place(relx=0.5, rely=0.5, anchor="center")
        tk.Label(frame, text="Login", font=("Arial", 22, "bold")).grid(row=0, columnspan=2, pady=10)
        tk.Label(frame, text="Username:").grid(row=1, column=0, sticky="e", pady=5)
        self.user_ent = tk.Entry(frame); self.user_ent.grid(row=1, column=1, pady=5)
        tk.Label(frame, text="Password:").grid(row=2, column=0, sticky="e", pady=5)
        self.pass_ent = tk.Entry(frame, show="*"); self.pass_ent.grid(row=2, column=1, pady=5)
        tk.Button(frame, text="Login", bg="#4CAF50", fg="black", width=15, command=self.handle_login).grid(row=3, columnspan=2, pady=10)
        tk.Button(frame, text="Create account", command=self.show_registration, borderwidth=0).grid(row=4, columnspan=2)

    def show_registration(self):
        self.clear_window()
        frame = tk.Frame(self.root, padx=30, pady=30, relief="groove", borderwidth=2)
        frame.place(relx=0.5, rely=0.5, anchor="center")
        tk.Label(frame, text="Register", font=("Arial", 22, "bold")).grid(row=0, columnspan=2, pady=10)
        tk.Label(frame, text="New Username:").grid(row=1, column=0, sticky="e", pady=5)
        self.reg_user = tk.Entry(frame); self.reg_user.grid(row=1, column=1, pady=5)
        tk.Label(frame, text="New Password:").grid(row=2, column=0, sticky="e", pady=5)
        self.reg_pass = tk.Entry(frame, show="*"); self.reg_pass.grid(row=2, column=1, pady=5)
        tk.Button(frame, text="Register Now", bg="#2196F3", fg="black", width=15, command=self.handle_registration).grid(row=3, columnspan=2, pady=10)
        tk.Button(frame, text="Back to Login", command=self.show_login, borderwidth=0).grid(row=4, columnspan=2)

    def show_main_app(self):
        self.clear_window()
        nav = tk.Frame(self.root, bg="#333", height=50)
        nav.pack(fill="x")
        tk.Label(nav, text="Event Dashboard", bg="#333", fg="white", font=("Arial", 14)).pack(side="left", padx=20)
        tk.Button(nav, text="Logout", bg="#f44336", fg="black", command=self.show_login).pack(side="right", padx=10, pady=10)

        body = tk.Frame(self.root, padx=20, pady=20)
        body.pack(fill="both", expand=True)

        form = tk.LabelFrame(body, text="Manage Event", padx=10, pady=10)
        form.pack(side="left", fill="y", padx=10)

        tk.Label(form, text="Title:").pack(anchor="w")
        self.title_ent = tk.Entry(form); self.title_ent.pack(fill="x", pady=2)
        tk.Label(form, text="Date:").pack(anchor="w")
        self.date_ent = tk.Entry(form); self.date_ent.pack(fill="x", pady=2)
        tk.Label(form, text="Location:").pack(anchor="w")
        self.loc_ent = tk.Entry(form); self.loc_ent.pack(fill="x", pady=2)
        
        # --- CLIENT REMARKS ---
        tk.Label(form, text="Client Remarks:").pack(anchor="w")
        self.remarks_ent = tk.Text(form, height=4, width=25)
        self.remarks_ent.pack(fill="x", pady=5)

        tk.Button(form, text="Add Event", bg="#4CAF50", fg="black", command=self.add_event).pack(fill="x", pady=5)
        tk.Button(form, text="Delete Selected", bg="#f44336", fg="black", command=self.delete_event).pack(fill="x", pady=5)
        
        # --- EXPORT BUTTON ---
        tk.Button(form, text="Export to CSV", bg="#2196F3", fg="black", command=self.export_data).pack(fill="x", pady=10)

        self.tree = ttk.Treeview(body, columns=("ID", "Title", "Date", "Location", "Remarks"), show="headings")
        for col in ("ID", "Title", "Date", "Location", "Remarks"):
            self.tree.heading(col, text=col)
            self.tree.column(col, width=120)
        self.tree.pack(side="right", fill="both", expand=True)
        self.load_events()

    def handle_login(self):
        conn = get_db_connection()
        if conn:
            cursor = conn.cursor()
            cursor.execute("SELECT user_id FROM users WHERE username=%s AND password=%s", (self.user_ent.get(), self.pass_ent.get()))
            row = cursor.fetchone()
            if row:
                self.current_user_id = row[0]
                self.show_main_app()
            else:
                messagebox.showerror("Error", "Invalid credentials")
            conn.close()

    def handle_registration(self):
        u, p = self.reg_user.get(), self.reg_pass.get()
        conn = get_db_connection()
        if conn:
            try:
                cursor = conn.cursor()
                cursor.execute("INSERT INTO users (username, password) VALUES (%s, %s)", (u, p))
                conn.commit()
                messagebox.showinfo("Success", "Account created!")
                self.show_login()
            except: messagebox.showerror("Error", "Username taken")
            finally: conn.close()

    def add_event(self):
        remarks = self.remarks_ent.get("1.0", tk.END).strip()
        date_str = self.date_ent.get()
        try:
            date_obj = datetime.strptime(date_str, '%Y-%m-%d')
        except ValueError:
            try:
                date_obj = datetime.strptime(date_str, '%m/%d/%Y')
            except ValueError:
                try:
                    date_obj = datetime.strptime(date_str, '%d/%m/%Y')
                except ValueError:
                    messagebox.showerror("Error", "Invalid date format. Please use YYYY-MM-DD, MM/DD/YYYY, or DD/MM/YYYY")
                    return
        formatted_date = date_obj.strftime('%Y-%m-%d')
        conn = get_db_connection()
        if conn:
            cursor = conn.cursor()
            cursor.execute("INSERT INTO events (user_id, title, date, location, remarks) VALUES (%s, %s, %s, %s, %s)", 
                           (self.current_user_id, self.title_ent.get(), formatted_date, self.loc_ent.get(), remarks))
            conn.commit(); conn.close()
            self.load_events()

    def load_events(self):
        for i in self.tree.get_children(): self.tree.delete(i)
        conn = get_db_connection()
        if conn:
            cursor = conn.cursor()
            cursor.execute("SELECT event_id, title, date, location, remarks FROM events WHERE user_id=%s", (self.current_user_id,))
            for row in cursor.fetchall(): self.tree.insert("", "end", values=row)
            conn.close()

    def delete_event(self):
        selected = self.tree.selection()
        if not selected: return
        event_id = self.tree.item(selected)['values'][0]
        conn = get_db_connection()
        if conn:
            cursor = conn.cursor()
            cursor.execute("DELETE FROM events WHERE event_id=%s", (event_id,))
            conn.commit(); conn.close()
            self.load_events()

    def export_data(self):
        file_path = filedialog.asksaveasfilename(defaultextension=".csv", filetypes=[("CSV files", "*.csv")])
        if file_path:
            with open(file_path, mode="w", newline="") as file:
                writer = csv.writer(file)
                writer.writerow(["ID", "Title", "Date", "Location", "Remarks"])
                for row_id in self.tree.get_children():
                    writer.writerow(self.tree.item(row_id)['values'])
            messagebox.showinfo("Export", "Data saved successfully!")

if __name__ == "__main__":
    root = tk.Tk()
    app = EventPlannerApp(root)
    root.mainloop()
