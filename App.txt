import cv2
from cvzone.HandTrackingModule import HandDetector
import mouse
import pyautogui
import numpy as np
import tkinter as tk
from tkinter import messagebox
import random
import threading
import time
from PIL import Image, ImageTk


class RandomPointApp:
    def __init__(self, master):
        self.master = master
        master.title("Кликни по точке!")

        
        self.width = 1600
        self.height = 900
        master.geometry(f"{self.width}x{self.height}")

        self.click_count = 0
        self.click_count_label = tk.Label(master, text="Кликов: 0", font=("Arial", 16), bg="black", fg="white")
        self.click_count_label.pack(anchor=tk.NW, padx=20, pady=20, side=tk.LEFT)

        self.time_elapsed = 0
        self.start_time = None
        self.timer_label = tk.Label(master, text="Время: 0.00", font=("Arial", 16), bg="black", fg="white")
        self.timer_label.pack(anchor=tk.NW, padx=20, pady=20, side=tk.LEFT)

        self.canvas = tk.Canvas(master, width=self.width, height=self.height, bg="black")
        self.canvas.pack(fill=tk.BOTH, expand=True)

        self.point_radius = 30  
        self.point_id = None

        self.create_random_point()
        self.update_timer()

    def create_random_point(self):
        """Создает или перемещает точку в случайное место."""
        
        x = random.randint(self.point_radius, self.width - self.point_radius)
        y = random.randint(self.point_radius, self.height - self.point_radius)

        if self.point_id is None:
            self.point_id = self.canvas.create_oval(x - self.point_radius, y - self.point_radius,
                                                    x + self.point_radius, y + self.point_radius,
                                                    fill="red", outline="red", tags="point")
            self.canvas.tag_bind("point", "<Button-1>", self.on_point_click)
        else:
           
            self.canvas.coords(self.point_id, x - self.point_radius, y - self.point_radius,
                               x + self.point_radius, y + self.point_radius)

    def on_point_click(self, event):
        """Обработчик клика по точке."""
        if self.start_time is None:
            self.start_time = time.time()

        self.click_count += 1
        self.click_count_label.config(text=f"Кликов: {self.click_count}")
        self.create_random_point()

        if self.click_count == 5:
            self.stop_timer()
            result = messagebox.showinfo("Игра окончена", f"Вы потратили {self.time_elapsed:.2f} секунд!")
            if result == "ok":  
                self.reset_game()  

    def reset_game(self):
        """Сбрасывает состояние игры и запускает новую."""
        self.click_count = 0
        self.click_count_label.config(text="Кликов: 0")
        self.time_elapsed = 0
        self.start_time = None
        self.timer_label.config(text="Время: 0.00")
        self.create_random_point()
        self.update_timer()

    def update_timer(self):
        """Обновляет таймер каждую сотую секунды."""
        if self.start_time is not None:
            self.time_elapsed = time.time() - self.start_time
            self.timer_label.config(text=f"Время: {self.time_elapsed:.2f}")

        self.master.after(10, self.update_timer)

    def stop_timer(self):
        """Останавливает таймер."""
        self.start_time = None



def show_camera(canvas, detector):
    cap = cv2.VideoCapture(0)
    cam_w, cam_h = 480, 360
    cap.set(cv2.CAP_PROP_FRAME_WIDTH, cam_w)
    cap.set(cv2.CAP_PROP_FRAME_HEIGHT, cam_h)
    cap.set(cv2.CAP_PROP_FPS, 60)

    screen_w, screen_h = pyautogui.size()

    while True:
        success, img = cap.read()
        if not success:
            break

        img = cv2.flip(img, 1)
        hands, img = detector.findHands(img)

        right_hand = None
        left_hand = None

        if hands:
            for hand in hands:
                if hand['type'] == 'Right':
                    right_hand = hand
                elif hand['type'] == 'Left':
                    left_hand = hand

            if right_hand:
                lmlist_right = right_hand['lmList']
                ind_x, ind_y = lmlist_right[8][0], lmlist_right[8][1]
                cv2.circle(img, (ind_x, ind_y), 5, (0, 255, 255), 2)
                conv_x = int(np.interp(ind_x, (0, cam_w), (0, screen_w)))
                conv_y = int(np.interp(ind_y, (0, cam_h), (0, screen_h)))
                mouse.move(conv_x, conv_y)

            if left_hand:
                fingers_left = detector.fingersUp(left_hand)
                if fingers_left == [0, 1, 1, 0, 0]:
                    pyautogui.mouseDown()
                elif fingers_left == [0, 0, 0, 0, 0]:
                    pyautogui.click()
                else:
                    pyautogui.mouseUp()

        
        img_rgb = cv2.cvtColor(img, cv2.COLOR_BGR2RGB)
        img_pil = Image.fromarray(img_rgb)
        img_tk = ImageTk.PhotoImage(image=img_pil)

        
        canvas.create_image(0, 0, anchor=tk.NW, image=img_tk)
        canvas.image = img_tk  

        if cv2.waitKey(1) & 0xFF == ord('q'):
            break

    cap.release()



def main_app():
    root = tk.Tk()
    root.title("Управление курсором и игра")

 
    canvas = tk.Canvas(root, width=800, height=600)
    canvas.grid(row=0, column=0, padx=10, pady=10)

   
    def open_clicker_game():
        clicker_root = tk.Toplevel(root)
        app = RandomPointApp(clicker_root)

    button_frame = tk.Frame(root)
    button_frame.grid(row=0, column=1, padx=10, pady=10, sticky="nsew")

    tk.Button(button_frame, text="Кликер", command=open_clicker_game, font=("Arial", 14)).pack(pady=20)
    
    detector = HandDetector(detectionCon=0.65, maxHands=2)
    threading.Thread(target=lambda: show_camera(canvas, detector), daemon=True).start()

    root.mainloop()


if __name__ == "__main__":
    main_app()
