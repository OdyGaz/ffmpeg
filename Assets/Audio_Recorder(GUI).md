# FFmpeg Screen & Audio Recorder (GUI)

Αυτή η εφαρμογή δημιουργήθηκε για να επιτρέπει την εύκολη καταγραφή της οθόνης (desktop) των Windows μαζί με μιξαρισμένο ήχο από εικονική κάρτα ήχου (VB-Audio Virtual Cable) και εξωτερικό μικρόφωνο (External Microphone), χωρίς την ανάγκη χρήσης του τερματικού (CMD).

---

## 💡 Η Ιδέα (The Concept)
Η χρήση του FFmpeg μέσω γραμμής εντολών είναι ισχυρή αλλά παρουσιάζει δύο βασικά προβλήματα στην καθημερινή χρήση:
1. **Πολυπλοκότητα**: Η πληκτρολόγηση μεγάλων εντολών κάθε φορά είναι κουραστική και χρονοβόρα.
2. **Κατεστραμμένα Βίντεο (Corrupted MP4)**: Αν κλείσουμε το παράθυρο του τερματικού βίαια ή πατήσουμε `Ctrl+C`, το FFmpeg δεν προλαβαίνει να γράψει το "ευρετήριο" (metadata index / moov atom) στο τέλος του αρχείου `.mp4`. Αυτό έχει ως αποτέλεσμα το βίντεο να μην μπορεί να αναπαραχθεί από κανέναν player.

Αυτή η εφαρμογή επιλύει και τα δύο προβλήματα προσφέροντας ένα απλό γραφικό περιβάλλον (GUI) που αναλαμβάνει όλη τη διαχείριση της διεργασίας του FFmpeg.

---

## ⚙️ Πώς Λειτουργεί (How it Works)
- **Γραφικό Περιβάλλον (Tkinter)**: Χρησιμοποιεί την ενσωματωμένη βιβλιοθήκη `tkinter` της Python για την εμφάνιση των κουμπιών (Browse, Record, Stop) χωρίς εξωτερικές εξαρτήσεις.
- **Multithreading (Πολυνηματική εκτέλεση)**: Το FFmpeg εκτελείται σε ξεχωριστό "νήμα" (background thread) ώστε το παράθυρο της εφαρμογής να παραμένει ενεργό και να μην "παγώνει" κατά τη διάρκεια της εγγραφής.
- **Ασφαλής Τερματισμός (Graceful Stop)**: Όταν πατάτε το κουμπί **Stop**, η εφαρμογή στέλνει τον χαρακτήρα `q\n` απευθείας στο `stdin` (standard input) της διεργασίας του FFmpeg. Αυτός είναι ο επίσημος τρόπος του FFmpeg για να σταματήσει την εγγραφή και να ολοκληρώσει σωστά το αρχείο βίντεο, κάνοντάς το απόλυτα λειτουργικό.
- **Διαχείριση Σφαλμάτων (Error Logging)**: Όλα τα μηνύματα του FFmpeg γράφονται σε ένα προσωρινό αρχείο `ffmpeg_log.txt`. Αν η εγγραφή αποτύχει (π.χ. επειδή δεν βρέθηκε το μικρόφωνο), η εφαρμογή διαβάζει το αρχείο και σας εμφανίζει παράθυρο με το ακριβές σφάλμα. Αν πετύχει, το αρχείο καταγραφής διαγράφεται αυτόματα.

---

## 💻 Ο Κώδικας Python (The Code)
Αποθηκεύστε τον παρακάτω κώδικα ως `recorder.py`:

```python
import tkinter as tk
from tkinter import filedialog, messagebox
import subprocess
import threading
import os

class FFmpegRecorderGUI:
    def __init__(self, root):
        self.root = root
        self.root.title("FFmpeg Screen Recorder")
        self.root.geometry("400x220")
        self.root.resizable(False, False)
        
        self.save_path = ""
        self.process = None
        
        # Label για την επιλογή διαδρομής
        self.lbl_path = tk.Label(root, text="Δεν έχει επιλεγεί τοποθεσία αποθήκευσης", wraplength=350, fg="gray")
        self.lbl_path.pack(pady=10)
        
        self.btn_browse = tk.Button(root, text="Browse (Επιλογή Φακέλου)", command=self.browse_file, width=25)
        self.btn_browse.pack(pady=5)
        
        # Κουμπιά ελέγχου
        self.btn_record = tk.Button(root, text="Record (Έναρξη)", bg="green", fg="white", state=tk.DISABLED, command=self.start_recording, width=25)
        self.btn_record.pack(pady=5)
        
        self.btn_stop = tk.Button(root, text="Stop (Διακοπή)", bg="red", fg="white", state=tk.DISABLED, command=self.stop_recording, width=25)
        self.btn_stop.pack(pady=5)
        
        self.lbl_status = tk.Label(root, text="Κατάσταση: Ανενεργό", fg="blue")
        self.lbl_status.pack(pady=10)

    def browse_file(self):
        path = filedialog.asksaveasfilename(
            defaultextension=".mp4",
            filetypes=[("MP4 Video", "*.mp4")],
            title="Αποθήκευση βίντεο ως..."
        )
        if path:
            self.save_path = path
            self.lbl_path.config(text=f"Αποθήκευση σε: {os.path.basename(path)}", fg="black")
            self.btn_record.config(state=tk.NORMAL)

    def start_recording(self):
        if not self.save_path:
            return
            
        self.btn_record.config(state=tk.DISABLED)
        self.btn_browse.config(state=tk.DISABLED)
        self.btn_stop.config(state=tk.NORMAL)
        self.lbl_status.config(text="Κατάσταση: Καταγραφή...", fg="red")
        
        # Εκκίνηση σε ξεχωριστό thread
        self.thread = threading.Thread(target=self.run_ffmpeg, daemon=True)
        self.thread.start()

    def run_ffmpeg(self):
        cmd = [
            "ffmpeg",
            "-y",
            "-f", "gdigrab",
            "-framerate", "30",
            "-i", "desktop",
            "-f", "dshow", "-i", "audio=CABLE Output (VB-Audio Virtual Cable)",
            "-f", "dshow", "-i", "audio=External Mic (Realtek(R) Audio)",
            "-filter_complex", "[1:a][2:a]amix=inputs=2[a]",
            "-map", "0:v",
            "-map", "[a]",
            "-c:v", "libx264",
            "-pix_fmt", "yuv420p",
            "-c:a", "aac",
            "-b:a", "192k",
            self.save_path
        ]
        
        # Αποθήκευση των μηνυμάτων του FFmpeg σε προσωρινό αρχείο καταγραφής για περίπτωση σφάλματος
        log_path = os.path.join(os.path.dirname(self.save_path), "ffmpeg_log.txt")
        
        try:
            with open(log_path, "w", encoding="utf-8") as log_file:
                self.process = subprocess.Popen(
                    cmd,
                    stdin=subprocess.PIPE,
                    stdout=log_file,
                    stderr=log_file,
                    creationflags=subprocess.CREATE_NO_WINDOW if os.name == 'nt' else 0
                )
                self.process.wait()  # Αναμονή μέχρι να κλείσει η διεργασία φυσιολογικά
            
            # Έλεγχος αν το FFmpeg έκλεισε με σφάλμα (return code διάφορο του 0)
            if self.process.returncode != 0:
                with open(log_path, "r", encoding="utf-8", errors="ignore") as f:
                    log_content = f.read()
                # Απομόνωση των τελευταίων γραμμών του σφάλματος
                error_lines = "\n".join(log_content.splitlines()[-10:])
                self.root.after(0, lambda: messagebox.showerror(
                    "Σφάλμα FFmpeg", 
                    f"Η εγγραφή απέτυχε (Exit code: {self.process.returncode}).\n\nΠιθανή αιτία:\n{error_lines}"
                ))
            else:
                # Αν όλα πήγαν καλά, σβήνουμε το log αρχείο και δείχνουμε επιτυχία
                if os.path.exists(log_path):
                    try: os.remove(log_path)
                    except: pass
                self.root.after(0, lambda: messagebox.showinfo("Ολοκληρώθηκε", "Η εγγραφή αποθηκεύτηκε επιτυχώς!"))
                
        except Exception as e:
            self.root.after(0, lambda: messagebox.showerror("Σφάλμα", f"Παρουσιάστηκε πρόβλημα:\n{str(e)}"))
        finally:
            self.root.after(0, self.reset_gui)
            
    def stop_recording(self):
        self.btn_stop.config(state=tk.DISABLED)  # Απενεργοποίηση για αποφυγή διπλού κλικ
        if self.process and self.process.poll() is None:
            try:
                # Στέλνουμε το 'q' και αλλαγή γραμμής για να κλείσει σωστά το MP4
                self.process.stdin.write(b'q\n')
                self.process.stdin.flush()
                self.process.stdin.close()
            except Exception:
                self.process.terminate()

    def reset_gui(self):
        self.btn_record.config(state=tk.NORMAL)
        self.btn_browse.config(state=tk.NORMAL)
        self.btn_stop.config(state=tk.DISABLED)
        self.lbl_status.config(text="Κατάσταση: Έτοιμο", fg="blue")

if __name__ == "__main__":
    root = tk.Tk()
    app = FFmpegRecorderGUI(root)
    root.mainloop()
```

---

## 🛠️ Πώς Τρέχει και Πώς Χτίζεται (How to Run & Build)

### Προαπαιτούμενα
1. Εγκατεστημένη έκδοση της **Python 3**.
2. Εγκατεστημένο το **FFmpeg** και περασμένο στα System PATH του υπολογιστή σας.

### 1. Απλή εκτέλεση με Python
Ανοίξτε το τερματικό στον φάκελο του αρχείου και τρέξτε:
```bash
python recorder.py
```

### 2. Χτίσιμο σε αυτόνομο αρχείο `.exe` (Build)
Για να μην χρειάζεται να ανοίγετε τερματικά, μπορείτε να μετατρέψετε το script σε ένα αυτόνομο αρχείο `.exe` με τη χρήση του **PyInstaller**:

1. Εγκαταστήστε το PyInstaller:
   ```bash
   pip install pyinstaller
   ```
2. Δημιουργήστε το `.exe` τρέχοντας:
   ```bash
   pyinstaller --onefile --noconsole recorder.py
   ```
   * `--onefile`: Συμπυκνώνει όλη την εφαρμογή σε ένα μόνο αρχείο `.exe`.
   * `--noconsole`: Αποτρέπει την εμφάνιση του μαύρου παραθύρου CMD όταν εκτελείτε την εφαρμογή.

3. Το έτοιμο αρχείο θα βρίσκεται μέσα στον φάκελο `dist` που θα δημιουργηθεί αυτόματα.
