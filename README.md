# Fast_Whisper_GUI

import os
import threading
import time
from tkinter import Tk, Label, Button, Entry, StringVar, Checkbutton, IntVar, filedialog, messagebox
from tkinter.ttk import Progressbar
from moviepy.editor import VideoFileClip
from faster_whisper import WhisperModel
import psutil
import GPUtil
from tkinter import Text, Scrollbar
from tkinter.constants import END, RIGHT, LEFT, Y
from moviepy.editor import AudioFileClip
from tkinter import Toplevel, Label, Button


def save_paths(input_file_path, model_folder_path):
    with open("last_paths.txt", "w") as f:
        f.write(f"{input_file_path}\n")
        f.write(f"{model_folder_path}\n")

def read_paths():
    try:
        with open("last_paths.txt", "r") as f:
            input_file_path = f.readline().strip()
            model_folder_path = f.readline().strip()
    except FileNotFoundError:
        input_file_path = ""
        model_folder_path = ""
    return input_file_path, model_folder_path

class TranscriptionGUI(Tk):
    def show_nonmodal_info(self, title, message):
        info_window = Toplevel(self)
        info_window.title(title)

        label = Label(info_window, text=message, wraplength=300)
        label.pack(padx=20, pady=20)

        ok_button = Button(info_window, text="OK", command=info_window.destroy)
        ok_button.pack(pady=(0, 20))

        info_window.focus_set()

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)

        self.input_file_list = []

        self.end_time_value = StringVar()
        self.transcription_running = False

        self.audio_process = None
        self.status_log_label = Label(self, text="Status Log:")
        self.status_log_label.grid(row=10, column=0, padx=10, pady=10, sticky="w")

        self.status_log_text = Text(self, wrap="word", width=60, height=10)
        self.status_log_text.grid(row=11, column=0, columnspan=3, padx=10, pady=10)

        self.status_log_scrollbar = Scrollbar(self, orient="vertical", command=self.status_log_text.yview)
        self.status_log_scrollbar.grid(row=11, column=3, sticky="ns")

        self.status_log_text.configure(yscrollcommand=self.status_log_scrollbar.set)


        self.title("Whisper Transcription")
        self.geometry("630x800")

        self.input_file_label = Label(self, text="Input File:")
        self.input_file_label.grid(row=0, column=0, padx=10, pady=10, sticky="w")

        self.input_file_path = StringVar()
        self.input_file_entry = Entry(self, textvariable=self.input_file_path, width=40)
        self.input_file_entry.grid(row=0, column=1, padx=10, pady=10)

        self.browse_input_button = Button(self, text="Browse", command=self.browse_input)
        self.browse_input_button.grid(row=0, column=2, padx=10, pady=10)

        self.model_folder_label = Label(self, text="Model Folder:")
        self.model_folder_label.grid(row=1, column=0, padx=10, pady=10, sticky="w")

        self.model_folder_path = StringVar()
        self.model_folder_entry = Entry(self, textvariable=self.model_folder_path, width=40)
        self.model_folder_entry.grid(row=1, column=1, padx=10, pady=10)

        self.browse_model_button = Button(self, text="Browse", command=self.browse_model)
        self.browse_model_button.grid(row=1, column=2, padx=10, pady=10)

        self.cpu_gpu = IntVar()
        self.cpu_gpu_checkbox = Checkbutton(self, text="Use GPU", variable=self.cpu_gpu)
        self.cpu_gpu_checkbox.grid(row=2, column=1, padx=10, pady=10, sticky="w")
        self.cpu_gpu.set(1)

        self.transcribe_button = Button(self, text="Transcribe", command=self.transcribe)
        self.transcribe_button.grid(row=3, column=1, padx=10, pady=10)

        self.stop_event = threading.Event()
        self.cancel_button = Button(self, text="Cancel", command=self.cancel)
        self.cancel_button.grid(row=3, column=2, padx=10, pady=10)

        self.progress = IntVar()
        self.progress_bar = Progressbar(self, length=400, mode="determinate", variable=self.progress)
        self.progress_bar.grid(row=4, column=0, padx=10, pady=10, columnspan=3)

        self.file_duration_label = Label(self, text="File duration:")
        self.file_duration_label.grid(row=5, column=0, padx=10, pady=10, sticky="w")
        self.file_duration_value = StringVar()
        self.file_duration_entry = Entry(self, textvariable=self.file_duration_value, width=12)
        self.file_duration_entry.grid(row=5, column=1, padx=10, pady=10, sticky="w")

        self.start_time_label = Label(self, text="Start time:")
        self.start_time_label.grid(row=6, column=0, padx=10, pady=10, sticky="w")
        self.start_time_value = StringVar()
        self.start_time_value.set("00:00:00")
        self.start_time_entry = Entry(self, textvariable=self.start_time_value, width=12)
        self.start_time_entry.grid(row=6, column=1, padx=10, pady=10, sticky="w")

        self.total_time_label = Label(self, text="Total time:")
        self.total_time_label.grid(row=7, column=0, padx=10, pady=10, sticky="w")
        self.total_time_value = StringVar()
        self.total_time_value.set("00:00:00")
        self.total_time_entry = Entry(self, textvariable=self.total_time_value, width=12)
        self.total_time_entry.grid(row=7, column=1, padx=10, pady=10, sticky="w")


        # Add these lines after the total_time_entry
        self.stopwatch_label = Label(self, text="Stopwatch:")
        self.stopwatch_label.grid(row=8, column=0, padx=10, pady=10, sticky="w")

        self.stopwatch_value = StringVar()
        self.stopwatch_value.set("00:00:00")
        self.stopwatch_entry = Entry(self, textvariable=self.stopwatch_value, width=12)
        self.stopwatch_entry.grid(row=8, column=1, padx=10, pady=10, sticky="w")

        
        # CPU&GPU Usage
        self.cpu_usage_label = Label(self, text="CPU Usage:")
        self.cpu_usage_label.grid(row=8, column=0, padx=10, pady=10, sticky="w")

        self.cpu_usage_value = StringVar()
        self.cpu_usage_value.set("0%")
        self.cpu_usage_entry = Entry(self, textvariable=self.cpu_usage_value, width=12)
        self.cpu_usage_entry.grid(row=8, column=1, padx=10, pady=10, sticky="w")

        self.gpu_usage_label = Label(self, text="GPU Usage:")
        self.gpu_usage_label.grid(row=9, column=0, padx=10, pady=10, sticky="w")

        self.gpu_usage_value = StringVar()
        self.gpu_usage_value.set("0%")
        self.gpu_usage_entry = Entry(self, textvariable=self.gpu_usage_value, width=12)
        self.gpu_usage_entry.grid(row=9, column=1, padx=10, pady=10, sticky="w")
        self.update_resource_usage()
        self.update_start_time()
        

        # Load last used paths
        input_file_path, model_folder_path = read_paths()
        self.input_file_path.set(input_file_path)
        self.model_folder_path.set(model_folder_path)

    def update_resource_usage(self):
        # Update CPU usage
        cpu_percent = psutil.cpu_percent()
        self.cpu_usage_value.set(f"{cpu_percent:.1f}%")

        # Update GPU usage
        gpus = GPUtil.getGPUs()
        if len(gpus) > 0:
            gpu = gpus[0]
            self.gpu_usage_value.set(f"{gpu.load * 100:.1f}%")
        else:
            self.gpu_usage_value.set("N/A")

        # Schedule the next update in 1000ms (1 second)
        self.after(1000, self.update_resource_usage)
    
    def browse_input(self):
        file_names = filedialog.askopenfilenames(title="Open Audio/Video Files", filetypes=[("Audio/Video Files", "*.mp4;*.mp3;*.wav;*.mkv"),
                                                                                            ("All Files", "*.*")])
        self.input_file_list = list(file_names)
        self.input_file_path.set(", ".join(file_names))  # Display the list of files in the Entry widget
        self.status_log_text.insert(END, f"{len(self.input_file_list)} file(s) loaded.\n")
        self.status_log_text.see(END)
        save_paths(self.input_file_path.get(), self.model_folder_path.get())

        # Calculate the total duration of the selected files
        total_duration = 0
        for file_name in self.input_file_list:
            video_extensions = (".mp4", ".mkv", ".avi", ".mov", ".flv", ".wmv")
            audio_extensions = (".mp3", ".wav", ".aac", ".m4a", ".flac", ".ogg")

            if file_name.lower().endswith(video_extensions):
                clip = VideoFileClip(file_name)
            elif file_name.lower().endswith(audio_extensions):
                clip = AudioFileClip(file_name)
            else:
                messagebox.showerror("Invalid File Type", "The selected file is not a supported audio or video file.")
                return

            duration = clip.duration
            total_duration += duration

        total_duration_str = time.strftime("%H:%M:%S", time.gmtime(total_duration))
        self.file_duration_value.set(total_duration_str)  # Update the file_duration_value

        # Display the list of files in the Entry widget
        self.input_file_path.set(", ".join(self.input_file_list))

        # Save the paths to the input file and model folder
        save_paths(self.input_file_path.get(), self.model_folder_path.get())

        # Calculate the total duration of the selected files
        total_duration = 0
        for file_name in self.input_file_list:
            video_extensions = (".mp4", ".mkv", ".avi", ".mov", ".flv", ".wmv")
            audio_extensions = (".mp3", ".wav", ".aac", ".m4a", ".flac", ".ogg")

            if file_name.lower().endswith(video_extensions):
                clip = VideoFileClip(file_name)
            elif file_name.lower().endswith(audio_extensions):
                clip = AudioFileClip(file_name)
            else:
                messagebox.showerror("Invalid File Type", "The selected file is not a supported audio or video file.")
                return

            duration = clip.duration
            total_duration += duration

        total_duration_str = time.strftime("%H:%M:%S", time.gmtime(total_duration))
        self.file_duration_value.set(total_duration_str)  # Update the file_duration_value

    def update_file_duration(self):
            file_name = self.input_file_path.get()
            if file_name:
                video_extensions = (".mp4", ".mkv", ".avi", ".mov", ".flv", ".wmv")
                audio_extensions = (".mp3", ".wav", ".aac", ".m4a", ".flac", ".ogg")
                if file_name.lower().endswith(video_extensions):
                    clip = VideoFileClip(file_name)
                elif file_name.lower().endswith(audio_extensions):
                    clip = AudioFileClip(file_name)
                else:
                    messagebox.showerror("Invalid File Type", "The selected file is not a supported audio or video file.")
                    return
                duration = clip.duration
                duration_str = time.strftime("%H:%M:%S", time.gmtime(duration))
                self.file_duration_value.set(duration_str)


            duration = clip.duration
            duration_str = time.strftime("%H:%M:%S", time.gmtime(duration))
            self.file_duration_value.set(duration_str)  # Update the file_duration_value

    def browse_model(self):
        folder_name = filedialog.askdirectory(title="Open Model Folder")
        self.model_folder_path.set(folder_name)
        save_paths(self.input_file_path.get(), self.model_folder_path.get())

    def transcribe_thread(self):
        successful_transcriptions = 0  # Initialize the counter before the for loop
        self.start_time = time.time()  # Move start time outside the for loop
        for input_file in self.input_file_list:
            input_file_path = input_file
            if input_file.endswith(".mp4") or input_file.endswith(".mkv"):
                try:
                    audio_file = self.extract_audio_from_video(input_file_path, self.stop_event)
                except Exception as e:
                    self.status_log_text.insert(END, str(e) + "\n")
                    self.status_log_text.see(END)
                    return
            elif input_file.endswith(".mp3") or input_file.endswith(".wav"):
                audio_file = input_file_path
            else:
                messagebox.showerror("Invalid File Type", "The selected file is not a supported audio file.")
                return

            model_folder = self.model_folder_path.get()
            config_path = os.path.join(model_folder, "config.json")
            model_path = os.path.join(model_folder, "model.bin")
            tokenizer_path = os.path.join(model_folder, "tokenizer.json")
            vocabulary_path = os.path.join(model_folder, "vocabulary.txt")

            device = "cuda" if self.cpu_gpu.get() == 1 else "cpu"
            compute_type = "float16" if device == "cuda" else "int8"

            try:
                model_size = "large-v2"
                model = WhisperModel(model_size, device=device, compute_type=compute_type)
            except Exception as e:
                self.status_log_text.insert(END, str(e) + "\n")
                self.status_log_text.see(END)
                return

            segments, info = model.transcribe(audio_file, beam_size=5)

            # Export results as .srt file
            srt_file = os.path.splitext(audio_file)[0] + ".srt"
            with open(srt_file, "w") as f:
                for i, segment in enumerate(segments):
                    f.write(f"{i + 1}\n")
                    f.write(f"{segment.start} --> {segment.end}\n")
                    f.write(f"{segment.text}\n\n")

            if input_file.endswith(".mp4") or input_file.endswith(".mkv"):
                os.remove(audio_file)  # Delete temporary audio file

            self.stopwatch_value.set("00:00:00")

            self.progress_bar.stop()
            self.progress.set(100)

            successful_transcriptions += 1 
            self.status_log_text.insert(END, f"{successful_transcriptions} file(s) transcribed successfully.\n")
            self.status_log_text.see(END)

            end_time = time.time()
            elapsed_time = end_time - self.start_time
            self.end_time_value.set(time.strftime("%H:%M:%S", time.gmtime(elapsed_time)))

        if successful_transcriptions == len(self.input_file_list):
            end_time = time.time()
            total_time = end_time - self.start_time
            total_time_str = time.strftime("%H:%M:%S", time.gmtime(total_time))
            self.total_time_value.set(total_time_str)

            # Update the progress bar after each file is transcribed
            progress_percentage = (self.input_file_list.index(input_file) + 1) * 100 / len(self.input_file_list)
            self.progress.set(progress_percentage)

        # Display a single message box after all files have been transcribed
        if successful_transcriptions == len(self.input_file_list):
            self.show_nonmodal_info("Success", "Transcription process completed successfully.")
            self.transcription_running = False

    def transcribe_audio(self):
        transcription_thread = threading.Thread(target=self.transcribe_thread)
        transcription_thread.start()
            
    def transcribe(self):
        self.status_log_text.delete(1.0, END)
        self.status_log_text.insert(END, "Transcribing...\n")
        self.progress.set(0)
        self.progress_bar.start()
        self.start_time = time.time()
        self.start_time_value.set("00:00:00")
        self.stopwatch_start_time = time.time()
        self.stop_event.clear()

        self.transcription_thread = threading.Thread(target=self.transcribe_thread)
        self.transcription_thread.start()
        self.stopwatch_start_time = time.time()
        self.update_stopwatch()
        self.transcription_running = True

        self.update_start_time()
      
    def update_start_time(self):
        if self.transcription_running:
            elapsed_time = time.time() - self.start_time
            elapsed_time_str = time.strftime("%H:%M:%S", time.gmtime(elapsed_time))
            self.start_time_value.set(elapsed_time_str)
            self.after(1000, self.update_start_time)

    def update_stopwatch(self):
        if not self.stop_event.is_set():
            elapsed_time = time.time() - self.stopwatch_start_time
            elapsed_time_str = time.strftime("%H:%M:%S", time.gmtime(elapsed_time))
            self.stopwatch_value.set(elapsed_time_str)
            self.after(1000, self.update_stopwatch)

    def cancel(self):
        self.stop_event.set()
        self.progress_bar.stop()
        self.progress.set(0)
        if self.transcription_thread.is_alive():
            self.status_log_text.insert(END, "Cancelling transcription...\n")
            if self.audio_process:
                self.audio_process.terminate()
            self.transcribe_thread.join()
            self.status_log_text.insert(END, "Transcription cancelled.\n")
            self.stopwatch_value.set("00:00:00")

    def update_progress(self):
        if not self.stop_event.is_set():
            self.progress_bar.step(1)
            if self.progress_bar['value'] < 100:
                self.after(100, self.update_progress)

    def extract_audio_from_video(self, video_path, stop_event):
        video = VideoFileClip(video_path)
        audio_path = os.path.splitext(video_path)[0] + ".wav"
        video.audio.write_audiofile(audio_path, codec='pcm_s16le')
        return audio_path

       
def run():
    app = TranscriptionGUI()
    app.mainloop()

  
if __name__ == "__main__":
    run()
