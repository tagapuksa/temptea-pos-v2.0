import customtkinter as ctk
import database as db
import pyttsx3
import threading
import time
import os
from tkinter import messagebox, ttk
from login import LoginWindow

# Ensure separate ticket counters exist in database or initialize them here
if not hasattr(db, 'student_ticket_counter'):
    db.student_ticket_counter = 1
if not hasattr(db, 'parent_ticket_counter'):
    db.parent_ticket_counter = 1


def speak_text(text):
    """Safely runs voice generation in an independent background thread so it never freezes the UI."""
    def run_speech():
        try:
            local_engine = pyttsx3.init()
            local_engine.setProperty('rate', 145)
            local_engine.say(text)
            local_engine.runAndWait()
            local_engine.stop()
            del local_engine
        except Exception as e:
            print(f"[Voice Engine Error]: {e}")
            
    threading.Thread(target=run_speech, daemon=True).start()


def get_ticket_voice_label(ticket_info):
    """Helper function to parse whether a ticket belongs to a Student or Guardian/Parent for TTS."""
    student_name = ticket_info.get("student", "")
    if "[Parent]" in student_name:
        return f"guardian number {ticket_info['id']:03d}"
    else:
        return f"student number {ticket_info['id']:03d}"


# ================================================================
# 📋 DATABASE MANAGER POPUP (SEPARATED STUDENT & PARENT TABS)
# ================================================================
class DatabaseWindow(ctk.CTkToplevel):
    """Pop-up panel showing live active transactions with separate Student and Parent database views."""
    def __init__(self, master, update_callback):
        super().__init__(master)
        self.title("Live Queue Database Manager")
        self.geometry("750x520")
        self.update_callback = update_callback
        
        self.attributes("-topmost", True)
        
        ctk.CTkLabel(self, text="📋 Live Queue Database Records", font=("Helvetica", 18, "bold")).pack(pady=10)
        
        # Tabview for separating Student and Parent databases
        self.tabview = ctk.CTkTabview(self)
        self.tabview.pack(fill="both", expand=True, padx=15, pady=5)
        
        self.tab_student = self.tabview.add("🎓 Student Queue")
        self.tab_parent = self.tabview.add("👨‍👩‍👧 Parent / Guardian Queue")
        
        style = ttk.Style()
        style.theme_use("clam")
        style.configure("Treeview", background="#2d3436", fieldbackground="#2d3436", foreground="white", rowheight=25)
        style.map("Treeview", background=[('selected', '#1a73e8')])

        # Build Student Tree
        self.stud_tree = ttk.Treeview(self.tab_student, columns=("Ticket", "ID_Name", "Purpose", "Phase"), show="headings")
        self.setup_tree_columns(self.stud_tree, "Student ID / Name")
        
        # Build Parent Tree
        self.parent_tree = ttk.Treeview(self.tab_parent, columns=("Ticket", "ID_Name", "Purpose", "Phase"), show="headings")
        self.setup_tree_columns(self.parent_tree, "Parent Name")
        
        # Bottom Control Action Buttons
        btn_frame = ctk.CTkFrame(self, fg_color="transparent")
        btn_frame.pack(fill="x", pady=10, padx=20)
        
        ctk.CTkButton(btn_frame, text="❌ Delete Selected Ticket", fg_color="#e74c3c", hover_color="#c0392b", command=self.delete_selected).pack(side="left", padx=5)
        ctk.CTkButton(btn_frame, text="📊 Open Tab Excel File", fg_color="#27ae60", hover_color="#219a52", command=self.open_excel_app).pack(side="right", padx=5)
        ctk.CTkButton(btn_frame, text="🔄 Refresh Tables", fg_color="#34495e", command=self.load_data).pack(side="right", padx=5)
        
        self.load_data()

    def setup_tree_columns(self, tree_widget, name_header):
        tree_widget.heading("Ticket", text="Ticket No.")
        tree_widget.heading("ID_Name", text=name_header)
        tree_widget.heading("Purpose", text="Purpose")
        tree_widget.heading("Phase", text="Queue Status")
        
        tree_widget.column("Ticket", width=110, anchor="center")
        tree_widget.column("ID_Name", width=220, anchor="w")
        tree_widget.column("Purpose", width=150, anchor="center")
        tree_widget.column("Phase", width=150, anchor="center")
        tree_widget.pack(fill="both", expand=True, padx=5, pady=5)

    def load_data(self):
        # Clear existing rows
        for item in self.stud_tree.get_children():
            self.stud_tree.delete(item)
        for item in self.parent_tree.get_children():
            self.parent_tree.delete(item)
            
        # Populate Student & Parent Tables independently
        for ticket in db.paying_queue:
            is_parent = "[Parent]" in ticket['student']
            phase = "Paying Phase"
            if is_parent:
                self.parent_tree.insert("", "end", values=(f"G-{ticket['id']:03d}", ticket['student'], ticket['purpose'], phase))
            else:
                self.stud_tree.insert("", "end", values=(f"S-{ticket['id']:03d}", ticket['student'], ticket['purpose'], phase))

        for ticket in db.preparation_queue:
            is_parent = "[Parent]" in ticket['student']
            phase = "Prep Phase"
            if is_parent:
                self.parent_tree.insert("", "end", values=(f"G-{ticket['id']:03d}", ticket['student'], ticket['purpose'], phase))
            else:
                self.stud_tree.insert("", "end", values=(f"S-{ticket['id']:03d}", ticket['student'], ticket['purpose'], phase))

    def delete_selected(self):
        current_tab = self.tabview.get()
        is_parent_tab = ("Parent" in current_tab)
        tree = self.parent_tree if is_parent_tab else self.stud_tree
        
        selected_item = tree.selection()
        if not selected_item:
            messagebox.showwarning("Selection Missing", "Please select a ticket row from the active tab table first.")
            return
            
        values = tree.item(selected_item, "values")
        target_ticket_str = values[0]
        prefix, id_str = target_ticket_str.split("-")
        target_id = int(id_str)
        is_parent_target = (prefix == "G")
        
        removed = False
        for ticket in db.paying_queue:
            ticket_is_parent = "[Parent]" in ticket['student']
            if ticket['id'] == target_id and ticket_is_parent == is_parent_target:
                db.paying_queue.remove(ticket)
                db.log_to_excel(ticket['id'], ticket['student'], ticket['purpose'], "Absent - Removed by Cashier", is_parent=is_parent_target)
                removed = True
                break
                
        if not removed:
            for ticket in db.preparation_queue:
                ticket_is_parent = "[Parent]" in ticket['student']
                if ticket['id'] == target_id and ticket_is_parent == is_parent_target:
                    db.preparation_queue.remove(ticket)
                    db.log_to_excel(ticket['id'], ticket['student'], ticket['purpose'], "Absent - Removed by Cashier", is_parent=is_parent_target)
                    break
                    
        messagebox.showinfo("Success", f"Ticket {target_ticket_str} has been cleared out.")
        self.load_data()
        self.update_callback()

    def open_excel_app(self):
        current_tab = self.tabview.get()
        is_parent = ("Parent" in current_tab)
        file_path = getattr(db, 'PARENT_EXCEL_FILE', 'parent_queue_database.xlsx') if is_parent else getattr(db, 'STUDENT_EXCEL_FILE', 'student_queue_database.xlsx')
        
        if os.path.exists(file_path):
            try:
                os.startfile(file_path)
            except Exception as e:
                messagebox.showerror("Error", f"Could not launch Excel: {e}")
        else:
            messagebox.showwarning("File Missing", f"No Excel records found yet for this queue. Print a ticket first!")


# ================================================================
# 📺 LOBBY TV MONITOR WINDOW (DUAL SIDE-BY-SIDE: STUDENT & PARENTS)
# ================================================================
class PublicTVMonitor(ctk.CTkToplevel):
    """The upgraded output display window for the Lobby TV Screen Monitor."""
    def __init__(self, master):
        super().__init__(master)
        self.title("Lobby TV Display")
        self.geometry("1100x700+100+100")
        self.configure(fg_color="#1e272e")
        self.setup_tv_layout()

    def setup_tv_layout(self):
        ctk.CTkLabel(self, text="CAMPUS QUEUE HUB", font=("Helvetica", 32, "bold"), text_color="#f5cd79").pack(pady=(20, 10))
        
        # Main Dual Container
        self.main_container = ctk.CTkFrame(self, fg_color="transparent")
        self.main_container.pack(fill="both", expand=True, padx=20, pady=10)
        self.main_container.grid_columnconfigure((0, 1), weight=1)
        self.main_container.grid_rowconfigure(0, weight=1)

        # ================= LEFT SIDE: STUDENTS =================
        self.stud_side = ctk.CTkFrame(self.main_container, fg_color="#2c3e50", corner_radius=15)
        self.stud_side.grid(row=0, column=0, padx=10, pady=5, sticky="nsew")

        ctk.CTkLabel(self.stud_side, text="STUDENTS QUEUE", font=("Helvetica", 22, "bold"), text_color="#3498db").pack(pady=(15, 5))
        ctk.CTkLabel(self.stud_side, text="NOW SERVING", font=("Helvetica", 18, "bold"), text_color="#ffffff").pack(pady=(5, 0))
        
        self.tv_stud_serving_lbl = ctk.CTkLabel(self.stud_side, text="S---", font=("Helvetica", 70, "bold"), text_color="#2ecc71")
        self.tv_stud_serving_lbl.pack(pady=10)

        # Student Prep Panel
        stud_prep_panel = ctk.CTkFrame(self.stud_side, fg_color="#151d24", corner_radius=12)
        stud_prep_panel.pack(fill="x", padx=15, pady=(10, 15))
        
        ctk.CTkLabel(stud_prep_panel, text="⏱️ UPCOMING STUDENT QUEUE", font=("Helvetica", 12, "bold"), text_color="#bdc3c7").pack(pady=(10, 5))
        
        self.stud_badge_container = ctk.CTkFrame(stud_prep_panel, fg_color="transparent")
        self.stud_badge_container.pack(pady=(0, 10), padx=10, fill="x")
        
        self.stud_prep_badges = []
        for i in range(4):
            self.stud_badge_container.grid_columnconfigure(i, weight=1)
            badge = ctk.CTkLabel(
                self.stud_badge_container, text="--", font=("Helvetica", 16, "bold"),
                text_color="#7f8c8d", fg_color="#1e272e", height=40, corner_radius=6
            )
            badge.grid(row=0, column=i, padx=4, sticky="ew")
            self.stud_prep_badges.append(badge)

        # ================= RIGHT SIDE: PARENTS =================
        self.parent_side = ctk.CTkFrame(self.main_container, fg_color="#2c3e50", corner_radius=15)
        self.parent_side.grid(row=0, column=1, padx=10, pady=5, sticky="nsew")

        ctk.CTkLabel(self.parent_side, text="PARENTS QUEUE", font=("Helvetica", 22, "bold"), text_color="#e67e22").pack(pady=(15, 5))
        ctk.CTkLabel(self.parent_side, text="NOW SERVING", font=("Helvetica", 18, "bold"), text_color="#ffffff").pack(pady=(5, 0))
        
        self.tv_parent_serving_lbl = ctk.CTkLabel(self.parent_side, text="G---", font=("Helvetica", 70, "bold"), text_color="#e74c3c")
        self.tv_parent_serving_lbl.pack(pady=10)

        # Parent Prep Panel
        parent_prep_panel = ctk.CTkFrame(self.parent_side, fg_color="#151d24", corner_radius=12)
        parent_prep_panel.pack(fill="x", padx=15, pady=(10, 15))
        
        ctk.CTkLabel(parent_prep_panel, text="⏱️ UPCOMING PARENTS QUEUE", font=("Helvetica", 12, "bold"), text_color="#bdc3c7").pack(pady=(10, 5))
        
        self.parent_badge_container = ctk.CTkFrame(parent_prep_panel, fg_color="transparent")
        self.parent_badge_container.pack(pady=(0, 10), padx=10, fill="x")
        
        self.parent_prep_badges = []
        for i in range(4):
            self.parent_badge_container.grid_columnconfigure(i, weight=1)
            badge = ctk.CTkLabel(
                self.parent_badge_container, text="--", font=("Helvetica", 16, "bold"),
                text_color="#7f8c8d", fg_color="#1e272e", height=40, corner_radius=6
            )
            badge.grid(row=0, column=i, padx=4, sticky="ew")
            self.parent_prep_badges.append(badge)

    def update_tv_screen(self):
        # Filter Queues separately
        all_queue = db.paying_queue + db.preparation_queue
        
        stud_queue = [t for t in all_queue if "[Parent]" not in t['student']]
        parent_queue = [t for t in all_queue if "[Parent]" in t['student']]

        # Update Student Side
        if stud_queue:
            self.tv_stud_serving_lbl.configure(text=f"S-{stud_queue[0]['id']:03d}")
            stud_upcoming = stud_queue[1:5]
        else:
            self.tv_stud_serving_lbl.configure(text="S---")
            stud_upcoming = []

        for idx in range(4):
            if idx < len(stud_upcoming):
                self.stud_prep_badges[idx].configure(
                    text=f"S-{stud_upcoming[idx]['id']:03d}",
                    text_color="#2ecc71", fg_color="#273c75"
                )
            else:
                self.stud_prep_badges[idx].configure(text="--", text_color="#7f8c8d", fg_color="#1e272e")

        # Update Parent Side
        if parent_queue:
            self.tv_parent_serving_lbl.configure(text=f"G-{parent_queue[0]['id']:03d}")
            parent_upcoming = parent_queue[1:5]
        else:
            self.tv_parent_serving_lbl.configure(text="G---")
            parent_upcoming = []

        for idx in range(4):
            if idx < len(parent_upcoming):
                self.parent_prep_badges[idx].configure(
                    text=f"G-{parent_upcoming[idx]['id']:03d}",
                    text_color="#e67e22", fg_color="#273c75"
                )
            else:
                self.parent_prep_badges[idx].configure(text="--", text_color="#7f8c8d", fg_color="#1e272e")


# ================================================================
# 🎫 COMBINED KIOSK TERMINAL WINDOW (STUDENT & PARENTS SIDE-BY-SIDE)
# ================================================================
class CombinedKioskWindow(ctk.CTkToplevel):
    """One single window containing BOTH Student and Parent Kiosk Terminals side-by-side."""
    def __init__(self, master_app):
        super().__init__(master_app)
        self.master_app = master_app
        self.title("Kiosk Ticketing Terminal")
        self.geometry("800x550+50+100")
        
        self.grid_columnconfigure((0, 1), weight=1)
        self.grid_rowconfigure(0, weight=1)
        
        # ================= PANEL 1: STUDENT KIOSK =================
        self.student_panel = ctk.CTkFrame(self, corner_radius=15)
        self.student_panel.grid(row=0, column=0, padx=15, pady=20, sticky="nsew")
        
        ctk.CTkLabel(self.student_panel, text="🎫 Student Kiosk Terminal", font=("Helvetica", 18, "bold")).pack(pady=15)
        
        self.stud_num_entry = ctk.CTkEntry(self.student_panel, placeholder_text="Enter ID Number or Name", width=220)
        self.stud_num_entry.pack(pady=15)
        
        ctk.CTkLabel(self.student_panel, text="Select Purpose:", font=("Helvetica", 12)).pack(pady=5)
        self.stud_purpose_var = ctk.StringVar(value="Tuition")
        self.stud_purpose_menu = ctk.CTkOptionMenu(self.student_panel, variable=self.stud_purpose_var, values=["Tuition", "Books", "Documents", "Other"])
        self.stud_purpose_menu.pack(pady=5)
        
        ctk.CTkLabel(self.student_panel, text="", font=("Helvetica", 12)).pack(pady=10)
        
        ctk.CTkButton(self.student_panel, text="🎟️ Get Ticket", font=("Helvetica", 14, "bold"), command=self.generate_student_ticket, fg_color="#2ecc71", hover_color="#27ae60", height=40).pack(pady=15)

        # ================= PANEL 2: PARENTS KIOSK =================
        self.parent_panel = ctk.CTkFrame(self, corner_radius=15)
        self.parent_panel.grid(row=0, column=1, padx=15, pady=20, sticky="nsew")
        
        ctk.CTkLabel(self.parent_panel, text="👨‍👩‍👧 Parents Kiosk Terminal", font=("Helvetica", 18, "bold")).pack(pady=15)
        
        self.parent_num_entry = ctk.CTkEntry(self.parent_panel, placeholder_text="Enter Parent / Guardian Name", width=220)
        self.parent_num_entry.pack(pady=15)
        
        ctk.CTkLabel(self.parent_panel, text="Select Purpose:", font=("Helvetica", 12)).pack(pady=5)
        self.parent_purpose_var = ctk.StringVar(value="Tuition")
        self.parent_purpose_menu = ctk.CTkOptionMenu(self.parent_panel, variable=self.parent_purpose_var, values=["Tuition", "Books", "Documents", "Other"])
        self.parent_purpose_menu.pack(pady=5)
        
        ctk.CTkLabel(self.parent_panel, text="", font=("Helvetica", 12)).pack(pady=10)
        
        ctk.CTkButton(self.parent_panel, text="🎟️ Get Ticket", font=("Helvetica", 14, "bold"), command=self.generate_parent_ticket, fg_color="#2ecc71", hover_color="#27ae60", height=40).pack(pady=15)

    def generate_student_ticket(self):
        student_input = self.stud_num_entry.get().strip() or "Walk-In Student"
        purpose = self.stud_purpose_var.get()
        ticket_num = db.student_ticket_counter
        db.student_ticket_counter += 1
        
        ticket_str = f"S-{ticket_num:03d}"
        ticket_data = {"id": ticket_num, "student": f"[Student] {student_input}", "purpose": purpose}
        
        active_stud_queue = [t for t in db.paying_queue if "[Parent]" not in t['student']]
        if len(active_stud_queue) < 3:
            db.paying_queue.append(ticket_data)
            db.log_to_excel(ticket_num, student_input, purpose, "Queued for Payment (Student)", is_parent=False)
        else:
            db.preparation_queue.append(ticket_data)
            db.log_to_excel(ticket_num, student_input, purpose, "In Preparation Pool (Student)", is_parent=False)
            
        # 🖨️ AUTO-PRINT THERMAL RECEIPT HERE
        if hasattr(db, 'print_thermal_receipt'):
            db.print_thermal_receipt(ticket_str, student_input, purpose, is_parent=False)

        self.stud_num_entry.delete(0, 'end')
        self.master_app.update_ui_displays()
        
        print(f"[Ticket Printed] {ticket_str} -> Registered to: [Student] {student_input}")
        speak_text(f"Ticket printed. Student number {ticket_num:03d}.")

    def generate_parent_ticket(self):
        parent_input = self.parent_num_entry.get().strip() or "Walk-In Parent"
        purpose = self.parent_purpose_var.get()
        ticket_num = db.parent_ticket_counter
        db.parent_ticket_counter += 1
        
        ticket_str = f"G-{ticket_num:03d}"
        ticket_data = {"id": ticket_num, "student": f"[Parent] {parent_input}", "purpose": purpose}
        
        active_parent_queue = [t for t in db.paying_queue if "[Parent]" in t['student']]
        if len(active_parent_queue) < 3:
            db.paying_queue.append(ticket_data)
            db.log_to_excel(ticket_num, parent_input, purpose, "Queued for Payment (Parent)", is_parent=True)
        else:
            db.preparation_queue.append(ticket_data)
            db.log_to_excel(ticket_num, parent_input, purpose, "In Preparation Pool (Parent)", is_parent=True)
            
        # 🖨️ AUTO-PRINT THERMAL RECEIPT HERE
        if hasattr(db, 'print_thermal_receipt'):
            db.print_thermal_receipt(ticket_str, parent_input, purpose, is_parent=True)

        self.parent_num_entry.delete(0, 'end')
        self.master_app.update_ui_displays()
        
        print(f"[Ticket Printed] {ticket_str} -> Registered to: [Parent] {parent_input}")
        speak_text(f"Ticket printed. Guardian number {ticket_num:03d}.")


# ================================================================
# 💼 CASHIER OPERATION DESK MAIN APP
# ================================================================
class CashierDeskApp(ctk.CTk):
    """The Admin Control Station App running locally on the Cashier Desk."""
    def __init__(self):
        super().__init__()
        self.title("Cashier Operation Desk")
        self.geometry("820x550+850+100")
        
        db.init_db()
        self.last_completed_student = None 
        self.last_completed_parent = None
        
        self.grid_columnconfigure((0, 1), weight=1)
        self.grid_rowconfigure(0, weight=1)
        
        # ================= DESK 1: STUDENT CASHIER DESK =================
        self.student_cashier_panel = ctk.CTkFrame(self, corner_radius=15)
        self.student_cashier_panel.grid(row=0, column=0, padx=15, pady=20, sticky="nsew")
        
        ctk.CTkLabel(self.student_cashier_panel, text="💼 Student Cashier Desk", font=("Helvetica", 18, "bold")).pack(pady=15)
        
        self.student_handling_lbl = ctk.CTkLabel(self.student_cashier_panel, text="No Student Active\nReady for next call.", font=("Helvetica", 14), text_color="#3498db")
        self.student_handling_lbl.pack(pady=20)
        
        ctk.CTkButton(self.student_cashier_panel, text="📢 Call Current Ticket", command=lambda: self.call_ticket("Student"), fg_color="#3498db", hover_color="#2980b9").pack(pady=8, padx=30, fill="x")
        ctk.CTkButton(self.student_cashier_panel, text="⏭️ Next Ticket", command=lambda: self.next_ticket("Student"), fg_color="#e67e22", hover_color="#d35400").pack(pady=8, padx=30, fill="x")
        ctk.CTkButton(self.student_cashier_panel, text="🔄 Recall Last Ticket", command=lambda: self.recall_ticket("Student"), fg_color="#2c3e50", hover_color="#1a252f").pack(pady=8, padx=30, fill="x")
        ctk.CTkButton(self.student_cashier_panel, text="📊 Open Queue Database", command=self.open_database_view, fg_color="#9b59b6", hover_color="#8e44ad").pack(pady=20, padx=30, fill="x")

        # ================= DESK 2: PARENTS CASHIER DESK =================
        self.parent_cashier_panel = ctk.CTkFrame(self, corner_radius=15)
        self.parent_cashier_panel.grid(row=0, column=1, padx=15, pady=20, sticky="nsew")
        
        ctk.CTkLabel(self.parent_cashier_panel, text="💼 Parents Cashier Desk", font=("Helvetica", 18, "bold")).pack(pady=15)
        
        self.parent_handling_lbl = ctk.CTkLabel(self.parent_cashier_panel, text="No Parent Active\nReady for next call.", font=("Helvetica", 14), text_color="#3498db")
        self.parent_handling_lbl.pack(pady=20)
        
        ctk.CTkButton(self.parent_cashier_panel, text="📢 Call Current Ticket", command=lambda: self.call_ticket("Parent"), fg_color="#3498db", hover_color="#2980b9").pack(pady=8, padx=30, fill="x")
        ctk.CTkButton(self.parent_cashier_panel, text="⏭️ Next Ticket", command=lambda: self.next_ticket("Parent"), fg_color="#e67e22", hover_color="#d35400").pack(pady=8, padx=30, fill="x")
        ctk.CTkButton(self.parent_cashier_panel, text="🔄 Recall Last Ticket", command=lambda: self.recall_ticket("Parent"), fg_color="#2c3e50", hover_color="#1a252f").pack(pady=8, padx=30, fill="x")
        ctk.CTkButton(self.parent_cashier_panel, text="📊 Open Queue Database", command=self.open_database_view, fg_color="#9b59b6", hover_color="#8e44ad").pack(pady=20, padx=30, fill="x")

        # Launch TV Monitor and the COMBINED Kiosks Window
        self.tv_window = PublicTVMonitor(self)
        self.kiosk_window = CombinedKioskWindow(self)
        
        self.update_ui_displays()

    def get_first_ticket_of_type(self, target_type):
        """Finds the active ticket matching the specified type ('Student' or 'Parent')."""
        for ticket in db.paying_queue:
            is_parent = "[Parent]" in ticket['student']
            if (target_type == "Parent" and is_parent) or (target_type == "Student" and not is_parent):
                return ticket
        return None

    def update_ui_displays(self):
        stud_ticket = self.get_first_ticket_of_type("Student")
        parent_ticket = self.get_first_ticket_of_type("Parent")

        # Update Student Desk Display
        if stud_ticket:
            self.student_handling_lbl.configure(
                text=f"Active Queue: S-{stud_ticket['id']:03d}\nID/Name: {stud_ticket['student']}\nPurpose: {stud_ticket['purpose']}"
            )
        else:
            self.student_handling_lbl.configure(text="No Student Active\nReady for next call.")

        # Update Parents Desk Display
        if parent_ticket:
            self.parent_handling_lbl.configure(
                text=f"Active Queue: G-{parent_ticket['id']:03d}\nID/Name: {parent_ticket['student']}\nPurpose: {parent_ticket['purpose']}"
            )
        else:
            self.parent_handling_lbl.configure(text="No Parent Active\nReady for next call.")
            
        self.tv_window.update_tv_screen()

    def call_ticket(self, queue_type):
        current = self.get_first_ticket_of_type(queue_type)
        if current:
            label = get_ticket_voice_label(current)
            speak_text(f"Calling {label}, Calling {label}")
        else:
            speak_text(f"There are no active {queue_type.lower()} numbers to call.")

    def next_ticket(self, queue_type):
        is_parent_type = (queue_type == "Parent")
        current = self.get_first_ticket_of_type(queue_type)
        if current:
            db.paying_queue.remove(current)
            db.log_to_excel(current['id'], current['student'], current['purpose'], f"{queue_type} Payment Completed", is_parent=is_parent_type)
            
            if queue_type == "Student":
                self.last_completed_student = current
            else:
                self.last_completed_parent = current
            
            # Promote next waiting item of this type from prep queue if any
            for prep in list(db.preparation_queue):
                is_parent = "[Parent]" in prep['student']
                if (queue_type == "Parent" and is_parent) or (queue_type == "Student" and not is_parent):
                    db.preparation_queue.remove(prep)
                    db.paying_queue.append(prep)
                    db.log_to_excel(prep['id'], prep['student'], prep['purpose'], f"Promoted to Paying ({queue_type})", is_parent=is_parent_type)
                    break
                
            self.update_ui_displays()
            
            next_one = self.get_first_ticket_of_type(queue_type)
            if next_one:
                label = get_ticket_voice_label(next_one)
                speak_text(f"Next {label}, Next {label}")
            else:
                speak_text(f"{queue_type} queue line is now empty.")
        else:
            speak_text(f"No {queue_type.lower()}s are currently waiting.")

    def recall_ticket(self, queue_type):
        is_parent_type = (queue_type == "Parent")
        last_ticket = self.last_completed_student if queue_type == "Student" else self.last_completed_parent
        if last_ticket:
            db.paying_queue.insert(0, last_ticket)
            db.log_to_excel(last_ticket['id'], last_ticket['student'], last_ticket['purpose'], f"Recalled to Desk ({queue_type})", is_parent=is_parent_type)
            
            label = get_ticket_voice_label(last_ticket)
            if queue_type == "Student":
                self.last_completed_student = None
            else:
                self.last_completed_parent = None

            self.update_ui_displays()
            speak_text(f"Recall {label}, Recall {label}")
        else:
            messagebox.showinfo("Recall Empty", f"No recently closed {queue_type.lower()} transactions to recall.")

    def open_database_view(self):
        DatabaseWindow(self, self.update_ui_displays)


def launch_main_system():
    app = CashierDeskApp()
    app.mainloop()

if __name__ == "__main__":
    login_screen = LoginWindow(on_success_bridge=launch_main_system)
    login_screen.mainloop()
    
