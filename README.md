import tkinter as tk
from tkinter import ttk, messagebox, scrolledtext
import json
import datetime
import matplotlib.pyplot as plt
from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg
import numpy as np
import os

class BMIFitnessTracker:
    def __init__(self, root):
        self.root = root
        self.root.title("BMI & Fitness Tracker")
        self.root.geometry("1200x800")
        
        # Configure style
        self.root.configure(bg='#f0f2f5')
        self.style = ttk.Style()
        self.style.theme_use('clam')
        
        # Configure colors
        self.colors = {
            'primary': '#3498db',
            'success': '#2ecc71',
            'warning': '#f39c12',
            'danger': '#e74c3c',
            'light': '#ecf0f1',
            'dark': '#2c3e50',
            'background': '#f0f2f5'
        }
        
        # Data storage
        self.entries = []
        self.goals = {'weight': None, 'bmi': None}
        self.current_unit = 'metric'  # 'metric' or 'imperial'
        
        # Load saved data
        self.load_data()
        
        # Setup UI
        self.setup_ui()
        
        # Update displays
        self.update_displays()
        
    def setup_ui(self):
        # Create notebook for tabs
        self.notebook = ttk.Notebook(self.root)
        self.notebook.pack(fill='both', expand=True, padx=10, pady=10)
        
        # Create tabs
        self.calculator_tab = ttk.Frame(self.notebook)
        self.progress_tab = ttk.Frame(self.notebook)
        self.history_tab = ttk.Frame(self.notebook)
        self.charts_tab = ttk.Frame(self.notebook)
        
        self.notebook.add(self.calculator_tab, text='BMI Calculator')
        self.notebook.add(self.progress_tab, text='Progress & Goals')
        self.notebook.add(self.history_tab, text='History')
        self.notebook.add(self.charts_tab, text='Charts')
        
        # Setup each tab
        self.setup_calculator_tab()
        self.setup_progress_tab()
        self.setup_history_tab()
        self.setup_charts_tab()
        
    def setup_calculator_tab(self):
        # Main frame
        main_frame = ttk.Frame(self.calculator_tab)
        main_frame.pack(fill='both', expand=True, padx=20, pady=20)
        
        # Title
        title_label = ttk.Label(main_frame, text="BMI Calculator", 
                                font=('Arial', 24, 'bold'),
                                foreground=self.colors['dark'])
        title_label.pack(pady=(0, 20))
        
        # Unit selection frame
        unit_frame = ttk.LabelFrame(main_frame, text="Unit System", padding=10)
        unit_frame.pack(fill='x', pady=(0, 20))
        
        self.unit_var = tk.StringVar(value='metric')
        ttk.Radiobutton(unit_frame, text="Metric (kg, cm)", 
                       variable=self.unit_var, value='metric',
                       command=self.toggle_units).pack(side='left', padx=20)
        ttk.Radiobutton(unit_frame, text="Imperial (lbs, ft/in)", 
                       variable=self.unit_var, value='imperial',
                       command=self.toggle_units).pack(side='left', padx=20)
        
        # Input frame
        input_frame = ttk.Frame(main_frame)
        input_frame.pack(fill='x', pady=(0, 20))
        
        # Metric inputs
        self.metric_frame = ttk.Frame(input_frame)
        
        ttk.Label(self.metric_frame, text="Weight (kg):").grid(row=0, column=0, sticky='w', pady=5)
        self.weight_kg = ttk.Entry(self.metric_frame, width=20)
        self.weight_kg.grid(row=0, column=1, pady=5, padx=(10, 20))
        
        ttk.Label(self.metric_frame, text="Height (cm):").grid(row=1, column=0, sticky='w', pady=5)
        self.height_cm = ttk.Entry(self.metric_frame, width=20)
        self.height_cm.grid(row=1, column=1, pady=5, padx=(10, 20))
        
        # Imperial inputs
        self.imperial_frame = ttk.Frame(input_frame)
        
        ttk.Label(self.imperial_frame, text="Weight (lbs):").grid(row=0, column=0, sticky='w', pady=5)
        self.weight_lbs = ttk.Entry(self.imperial_frame, width=20)
        self.weight_lbs.grid(row=0, column=1, pady=5, padx=(10, 20))
        
        ttk.Label(self.imperial_frame, text="Height:").grid(row=1, column=0, sticky='w', pady=5)
        
        height_imperial_frame = ttk.Frame(self.imperial_frame)
        height_imperial_frame.grid(row=1, column=1, pady=5, padx=(10, 20))
        
        self.height_ft = ttk.Entry(height_imperial_frame, width=8)
        self.height_ft.pack(side='left')
        ttk.Label(height_imperial_frame, text="ft").pack(side='left', padx=5)
        
        self.height_in = ttk.Entry(height_imperial_frame, width=8)
        self.height_in.pack(side='left', padx=(10, 0))
        ttk.Label(height_imperial_frame, text="in").pack(side='left', padx=5)
        
        # Date input
        date_frame = ttk.Frame(main_frame)
        date_frame.pack(fill='x', pady=(0, 20))
        
        ttk.Label(date_frame, text="Date:").pack(side='left')
        self.date_entry = ttk.Entry(date_frame, width=20)
        self.date_entry.pack(side='left', padx=(10, 20))
        self.date_entry.insert(0, datetime.datetime.now().strftime("%Y-%m-%d"))
        
        # Buttons frame
        button_frame = ttk.Frame(main_frame)
        button_frame.pack(fill='x', pady=(0, 20))
        
        ttk.Button(button_frame, text="Calculate BMI", 
                  command=self.calculate_bmi,
                  style='Primary.TButton').pack(side='left', padx=5)
        ttk.Button(button_frame, text="Save Entry", 
                  command=self.save_entry,
                  style='Success.TButton').pack(side='left', padx=5)
        ttk.Button(button_frame, text="Clear All", 
                  command=self.clear_inputs).pack(side='left', padx=5)
        
        # Result frame
        self.result_frame = ttk.LabelFrame(main_frame, text="Result", padding=20)
        self.result_frame.pack(fill='both', expand=True, pady=(0, 20))
        
        self.bmi_value_label = ttk.Label(self.result_frame, text="--", 
                                        font=('Arial', 48, 'bold'),
                                        foreground=self.colors['dark'])
        self.bmi_value_label.pack()
        
        self.bmi_category_label = ttk.Label(self.result_frame, text="", 
                                           font=('Arial', 18, 'bold'))
        self.bmi_category_label.pack(pady=(10, 5))
        
        self.bmi_message_label = ttk.Label(self.result_frame, text="", 
                                          font=('Arial', 12),
                                          foreground=self.colors['dark'])
        self.bmi_message_label.pack()
        
        # Style configuration
        self.style.configure('Primary.TButton', 
                           background=self.colors['primary'],
                           foreground='white')
        self.style.configure('Success.TButton', 
                           background=self.colors['success'],
                           foreground='white')
        
        # Show metric by default
        self.metric_frame.pack()
        
    def setup_progress_tab(self):
        main_frame = ttk.Frame(self.progress_tab)
        main_frame.pack(fill='both', expand=True, padx=20, pady=20)
        
        # Title
        ttk.Label(main_frame, text="Progress & Goals", 
                 font=('Arial', 24, 'bold'),
                 foreground=self.colors['dark']).pack(pady=(0, 20))
        
        # Stats frame
        stats_frame = ttk.Frame(main_frame)
        stats_frame.pack(fill='x', pady=(0, 20))
        
        stats_data = [
            ("Current BMI", "current_bmi", "--"),
            ("Best BMI", "best_bmi", "--"),
            ("Total Entries", "total_entries", "0"),
            ("7-Day Trend", "week_trend", "--")
        ]
        
        self.stats_labels = {}
        
        for i, (title, key, default) in enumerate(stats_data):
            frame = ttk.Frame(stats_frame, relief='ridge', padding=15)
            frame.grid(row=0, column=i, padx=5, sticky='nsew')
            stats_frame.columnconfigure(i, weight=1)
            
            ttk.Label(frame, text=title, 
                     font=('Arial', 10, 'bold'),
                     foreground=self.colors['dark']).pack()
            
            label = ttk.Label(frame, text=default,
                            font=('Arial', 24, 'bold'),
                            foreground=self.colors['primary'])
            label.pack(pady=(5, 0))
            
            self.stats_labels[key] = label
        
        # Goals frame
        goals_frame = ttk.LabelFrame(main_frame, text="Set Goals", padding=20)
        goals_frame.pack(fill='both', expand=True, pady=(0, 20))
        
        # Weight goal
        goal_weight_frame = ttk.Frame(goals_frame)
        goal_weight_frame.pack(fill='x', pady=(0, 10))
        
        ttk.Label(goal_weight_frame, text="Target Weight (kg):").pack(side='left')
        self.target_weight_entry = ttk.Entry(goal_weight_frame, width=15)
        self.target_weight_entry.pack(side='left', padx=(10, 20))
        
        # BMI goal
        goal_bmi_frame = ttk.Frame(goals_frame)
        goal_bmi_frame.pack(fill='x', pady=(0, 10))
        
        ttk.Label(goal_bmi_frame, text="Target BMI:").pack(side='left')
        self.target_bmi_entry = ttk.Entry(goal_bmi_frame, width=15)
        self.target_bmi_entry.pack(side='left', padx=(10, 20))
        
        ttk.Button(goals_frame, text="Set Goals", 
                  command=self.set_goals,
                  style='Success.TButton').pack(pady=(10, 0))
        
        # Progress bars frame
        self.progress_frame = ttk.LabelFrame(main_frame, text="Goal Progress", padding=20)
        self.progress_frame.pack(fill='both', expand=True)
        
        self.weight_progress_label = ttk.Label(self.progress_frame, text="Weight Goal: -- / -- kg")
        self.weight_progress_label.pack(anchor='w', pady=(0, 5))
        
        self.weight_progress = ttk.Progressbar(self.progress_frame, length=300, mode='determinate')
        self.weight_progress.pack(fill='x', pady=(0, 15))
        
        self.bmi_progress_label = ttk.Label(self.progress_frame, text="BMI Goal: -- / --")
        self.bmi_progress_label.pack(anchor='w', pady=(0, 5))
        
        self.bmi_progress = ttk.Progressbar(self.progress_frame, length=300, mode='determinate')
        self.bmi_progress.pack(fill='x')
        
    def setup_history_tab(self):
        main_frame = ttk.Frame(self.history_tab)
        main_frame.pack(fill='both', expand=True, padx=20, pady=20)
        
        # Title and controls
        header_frame = ttk.Frame(main_frame)
        header_frame.pack(fill='x', pady=(0, 20))
        
        ttk.Label(header_frame, text="History Log", 
                 font=('Arial', 24, 'bold'),
                 foreground=self.colors['dark']).pack(side='left')
        
        ttk.Button(header_frame, text="Clear All History", 
                  command=self.clear_history,
                  style='Danger.TButton').pack(side='right')
        
        # History table
        columns = ('Date', 'Weight', 'Height', 'BMI', 'Category')
        self.history_tree = ttk.Treeview(main_frame, columns=columns, show='headings', height=15)
        
        # Configure columns
        col_widths = {'Date': 100, 'Weight': 100, 'Height': 100, 'BMI': 80, 'Category': 100}
        for col in columns:
            self.history_tree.heading(col, text=col)
            self.history_tree.column(col, width=col_widths.get(col, 100))
        
        # Add scrollbar
        scrollbar = ttk.Scrollbar(main_frame, orient='vertical', command=self.history_tree.yview)
        self.history_tree.configure(yscrollcommand=scrollbar.set)
        
        # Pack tree and scrollbar
        self.history_tree.pack(side='left', fill='both', expand=True)
        scrollbar.pack(side='right', fill='y')
        
        # Configure danger button style
        self.style.configure('Danger.TButton', 
                           background=self.colors['danger'],
                           foreground='white')
        
    def setup_charts_tab(self):
        main_frame = ttk.Frame(self.charts_tab)
        main_frame.pack(fill='both', expand=True, padx=20, pady=20)
        
        ttk.Label(main_frame, text="Progress Charts", 
                 font=('Arial', 24, 'bold'),
                 foreground=self.colors['dark']).pack(pady=(0, 20))
        
        # Create figure for charts
        self.figure = plt.Figure(figsize=(10, 8), dpi=100)
        self.canvas = FigureCanvasTkAgg(self.figure, main_frame)
        self.canvas.get_tk_widget().pack(fill='both', expand=True)
        
    def toggle_units(self):
        unit = self.unit_var.get()
        if unit == 'metric':
            self.metric_frame.pack()
            self.imperial_frame.pack_forget()
        else:
            self.imperial_frame.pack()
            self.metric_frame.pack_forget()
        
    def calculate_bmi(self):
        try:
            if self.unit_var.get() == 'metric':
                weight = float(self.weight_kg.get())
                height = float(self.height_cm.get()) / 100  # Convert to meters
            else:
                weight_lbs = float(self.weight_lbs.get())
                height_ft = float(self.height_ft.get())
                height_in = float(self.height_in.get() or 0)
                
                # Convert to metric
                weight = weight_lbs * 0.453592
                height = (height_ft * 12 + height_in) * 0.0254
            
            if weight <= 0 or height <= 0:
                messagebox.showerror("Error", "Please enter valid weight and height values")
                return
            
            bmi = weight / (height * height)
            self.display_result(bmi)
            
        except ValueError:
            messagebox.showerror("Error", "Please enter valid numeric values")
            
    def display_result(self, bmi):
        bmi_rounded = round(bmi, 1)
        self.bmi_value_label.config(text=str(bmi_rounded))
        
        if bmi < 18.5:
            category = "Underweight"
            color = self.colors['primary']
            message = "Consider gaining weight through a balanced diet"
        elif bmi < 25:
            category = "Normal"
            color = self.colors['success']
            message = "Healthy weight! Maintain your lifestyle"
        elif bmi < 30:
            category = "Overweight"
            color = self.colors['warning']
            message = "Consider light exercise and diet adjustments"
        else:
            category = "Obese"
            color = self.colors['danger']
            message = "Consult a healthcare provider for guidance"
        
        self.bmi_category_label.config(text=category, foreground=color)
        self.bmi_message_label.config(text=message)
        
    def save_entry(self):
        try:
            # Get values based on unit system
            if self.unit_var.get() == 'metric':
                weight = float(self.weight_kg.get())
                height = float(self.height_cm.get())
                unit = 'metric'
            else:
                weight = float(self.weight_lbs.get())
                height_ft = float(self.height_ft.get())
                height_in = float(self.height_in.get() or 0)
                height = height_ft * 12 + height_in  # Store in inches
                unit = 'imperial'
            
            date = self.date_entry.get()
            
            # Calculate BMI
            if self.unit_var.get() == 'metric':
                bmi = weight / ((height / 100) ** 2)
            else:
                # Convert to metric for BMI calculation
                weight_kg = weight * 0.453592
                height_m = height * 0.0254
                bmi = weight_kg / (height_m ** 2)
            
            bmi_rounded = round(bmi, 1)
            
            # Determine category
            if bmi < 18.5:
                category = "Underweight"
            elif bmi < 25:
                category = "Normal"
            elif bmi < 30:
                category = "Overweight"
            else:
                category = "Obese"
            
            # Create entry
            entry = {
                'id': len(self.entries) + 1,
                'date': date,
                'weight': weight,
                'height': height,
                'bmi': bmi_rounded,
                'category': category,
                'unit': unit
            }
            
            self.entries.append(entry)
            self.save_data()
            self.update_displays()
            
            messagebox.showinfo("Success", "Entry saved successfully!")
            
        except ValueError:
            messagebox.showerror("Error", "Please calculate BMI first and ensure all fields are filled correctly")
            
    def set_goals(self):
        try:
            weight = self.target_weight_entry.get()
            bmi = self.target_bmi_entry.get()
            
            if weight:
                self.goals['weight'] = float(weight)
            if bmi:
                self.goals['bmi'] = float(bmi)
            
            self.save_data()
            self.update_goal_progress()
            messagebox.showinfo("Success", "Goals updated successfully!")
            
        except ValueError:
            messagebox.showerror("Error", "Please enter valid numeric values")
            
    def clear_inputs(self):
        self.weight_kg.delete(0, tk.END)
        self.height_cm.delete(0, tk.END)
        self.weight_lbs.delete(0, tk.END)
        self.height_ft.delete(0, tk.END)
        self.height_in.delete(0, tk.END)
        self.date_entry.delete(0, tk.END)
        self.date_entry.insert(0, datetime.datetime.now().strftime("%Y-%m-%d"))
        
        self.bmi_value_label.config(text="--")
        self.bmi_category_label.config(text="")
        self.bmi_message_label.config(text="")
        
    def clear_history(self):
        if messagebox.askyesno("Confirm", "Are you sure you want to clear all history? This cannot be undone."):
            self.entries = []
            self.save_data()
            self.update_displays()
            
    def update_displays(self):
        self.update_history_table()
        self.update_stats()
        self.update_charts()
        self.update_goal_progress()
        
    def update_history_table(self):
        # Clear existing items
        for item in self.history_tree.get_children():
            self.history_tree.delete(item)
        
        # Add entries
        for entry in sorted(self.entries, key=lambda x: x['date'], reverse=True):
            if entry['unit'] == 'metric':
                weight_display = f"{entry['weight']} kg"
                height_display = f"{entry['height']} cm"
            else:
                weight_display = f"{entry['weight']} lbs"
                feet = int(entry['height'] // 12)
                inches = int(entry['height'] % 12)
                height_display = f"{feet}'{inches}\""
            
            values = (
                entry['date'],
                weight_display,
                height_display,
                entry['bmi'],
                entry['category']
            )
            self.history_tree.insert('', 'end', values=values)
            
    def update_stats(self):
        if not self.entries:
            self.stats_labels['current_bmi'].config(text="--")
            self.stats_labels['best_bmi'].config(text="--")
            self.stats_labels['total_entries'].config(text="0")
            self.stats_labels['week_trend'].config(text="--")
            return
        
        # Current BMI (latest entry)
        latest = self.entries[-1]
        self.stats_labels['current_bmi'].config(text=str(latest['bmi']))
        
        # Best BMI (closest to 22)
        best_entry = min(self.entries, key=lambda x: abs(x['bmi'] - 22))
        self.stats_labels['best_bmi'].config(text=str(best_entry['bmi']))
        
        # Total entries
        self.stats_labels['total_entries'].config(text=str(len(self.entries)))
        
        # 7-day trend
        recent_entries = self.entries[-7:]
        if len(recent_entries) >= 2:
            first = recent_entries[0]['bmi']
            last = recent_entries[-1]['bmi']
            trend = last - first
            trend_symbol = "↗️" if trend > 0 else "↘️" if trend < 0 else "➡️"
            self.stats_labels['week_trend'].config(text=f"{trend_symbol} {abs(trend):.1f}")
        else:
            self.stats_labels['week_trend'].config(text="--")
            
    def update_charts(self):
        if not self.entries:
            return
        
        # Clear figure
        self.figure.clear()
        
        # Prepare data
        dates = [entry['date'] for entry in self.entries]
        bmis = [entry['bmi'] for entry in self.entries]
        weights = [entry['weight'] for entry in self.entries]
        
        # Create subplots
        ax1 = self.figure.add_subplot(211)
        ax2 = self.figure.add_subplot(212)
        
        # Plot BMI trend
        ax1.plot(dates, bmis, 'b-o', linewidth=2, markersize=4)
        ax1.axhline(y=18.5, color='r', linestyle='--', alpha=0.5, label='Underweight')
        ax1.axhline(y=25, color='g', linestyle='--', alpha=0.5, label='Normal')
        ax1.axhline(y=30, color='orange', linestyle='--', alpha=0.5, label='Overweight')
        ax1.fill_between(dates, 18.5, 25, alpha=0.1, color='green')
        ax1.set_title('BMI Trend Over Time', fontsize=14, fontweight='bold')
        ax1.set_ylabel('BMI')
        ax1.legend()
        ax1.grid(True, alpha=0.3)
        
        # Rotate x-axis labels for better visibility
        ax1.set_xticklabels(dates, rotation=45, ha='right')
        
        # Plot weight trend
        ax2.plot(dates, weights, 'g-s', linewidth=2, markersize=4)
        ax2.set_title('Weight Trend', fontsize=14, fontweight='bold')
        ax2.set_ylabel('Weight (kg)' if self.entries[0]['unit'] == 'metric' else 'Weight (lbs)')
        ax2.set_xlabel('Date')
        ax2.grid(True, alpha=0.3)
        ax2.set_xticklabels(dates, rotation=45, ha='right')
        
        # Adjust layout
        self.figure.tight_layout()
        
        # Update canvas
        self.canvas.draw()
        
    def update_goal_progress(self):
        if not self.entries:
            self.weight_progress_label.config(text="Weight Goal: -- / -- kg")
            self.bmi_progress_label.config(text="BMI Goal: -- / --")
            self.weight_progress['value'] = 0
            self.bmi_progress['value'] = 0
            return
        
        latest = self.entries[-1]
        
        # Update weight progress
        if self.goals['weight']:
            current_weight = latest['weight']
            if latest['unit'] == 'imperial':
                current_weight = current_weight * 0.453592  # Convert to kg
            
            progress = (current_weight / self.goals['weight']) * 100
            progress = min(100, progress)  # Cap at 100%
            
            self.weight_progress_label.config(
                text=f"Weight Goal: {current_weight:.1f} / {self.goals['weight']} kg"
            )
            self.weight_progress['value'] = progress
        else:
            self.weight_progress_label.config(text="Weight Goal: Not set")
            self.weight_progress['value'] = 0
        
        # Update BMI progress
        if self.goals['bmi']:
            current_bmi = latest['bmi']
            progress = (current_bmi / self.goals['bmi']) * 100
            progress = min(100, progress)  # Cap at 100%
            
            self.bmi_progress_label.config(
                text=f"BMI Goal: {current_bmi} / {self.goals['bmi']}"
            )
            self.bmi_progress['value'] = progress
        else:
            self.bmi_progress_label.config(text="BMI Goal: Not set")
            self.bmi_progress['value'] = 0
            
    def load_data(self):
        try:
            if os.path.exists('bmi_data.json'):
                with open('bmi_data.json', 'r') as f:
                    data = json.load(f)
                    self.entries = data.get('entries', [])
                    self.goals = data.get('goals', {'weight': None, 'bmi': None})
        except Exception as e:
            print(f"Error loading data: {e}")
            
    def save_data(self):
        try:
            data = {
                'entries': self.entries,
                'goals': self.goals
            }
            with open('bmi_data.json', 'w') as f:
                json.dump(data, f, indent=2)
        except Exception as e:
            print(f"Error saving data: {e}")

def main():
    root = tk.Tk()
    app = BMIFitnessTracker(root)
    root.mainloop()

if __name__ == "__main__":
    main()
