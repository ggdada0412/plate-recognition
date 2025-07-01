import os
import time
import boto3
import tkinter as tk
from PIL import Image, ImageTk
import cv2
import csv
from datetime import datetime
import re
from tkinter import messagebox

#AWS 金鑰與區域設定
os.environ['AWS_ACCESS_KEY_ID'] = "你的存取金鑰"
os.environ['AWS_SECRET_ACCESS_KEY'] = "你的私密存取金鑰"
region = "你的儲存桶位置"
bucket_name = "你的儲存桶名稱"

#自訂參數
image_folder = "plates"
csv_file = "results_ui.csv"

#AWS 客戶端初始化
s3 = boto3.client('s3', region_name=region)
rekognition = boto3.client('rekognition', region_name=region)

# 確保資料夾存在
os.makedirs(image_folder, exist_ok=True)

# 初始化 CSV 檔
if not os.path.exists(csv_file):
    with open(csv_file, mode='w', newline='', encoding='utf-8') as file:
        writer = csv.writer(file)
        writer.writerow(["圖片檔名", "車牌文字", "時間"])

#車牌格式化函數
def format_license_plate(text):
    clean = re.sub(r'[^A-Za-z0-9]', '', text).upper()
    if len(clean) >= 7:
        return f"{clean[:3]}-{clean[3:]}"
    return text

#GUI 設定
root = tk.Tk()
root.title("車牌辨識系統")
root.geometry("1000x600")

#紀錄區（唯讀）
log_frame = tk.Frame(root, bd=2, relief="solid")
log_frame.pack(side="left", fill="both", expand=True)
log_text = tk.Text(log_frame, font=("Arial", 14), state="normal")
log_text.insert("end", f"{'車牌號':<15} {'時間':<25}\n")
log_text.config(state="disabled")  #設為唯讀
log_text.pack(fill="both", expand=True)

#即時畫面顯示
image_frame = tk.Frame(root, bd=2, relief="solid", width=400, height=300)
image_frame.pack(padx=10, pady=10)
image_label = tk.Label(image_frame, text="攝影機畫面")
image_label.pack()

#控制按鈕
button_frame = tk.Frame(root)
button_frame.pack(pady=20)

def capture_and_detect():
    ret, frame = cap.read()
    if not ret:
        messagebox.showerror("錯誤", "無法讀取攝影機畫面")
        return

    timestamp_str = datetime.now().strftime("%Y%m%d_%H%M%S")
    filename = f"{timestamp_str}.jpg"
    filepath = os.path.join(image_folder, filename)
    cv2.imwrite(filepath, frame)

    try:
        #上傳到 S3
        s3.upload_file(filepath, bucket_name, filename)

        #Rekognition 辨識
        response = rekognition.detect_text(
            Image={'S3Object': {'Bucket': bucket_name, 'Name': filename}}
        )
        detected_lines = [
            text['DetectedText']
            for text in response['TextDetections']
            if text['Type'] == 'LINE'
        ]
        license_plate = format_license_plate(detected_lines[0]) if detected_lines else "(未偵測)"

        timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

        #寫入 CSV
        with open(csv_file, mode='a', newline='', encoding='utf-8') as file:
            writer = csv.writer(file)
            writer.writerow([filename, license_plate, timestamp])

        #更新紀錄（解除唯讀-更新-再設回唯讀）
        log_text.config(state="normal")
        log_text.insert("end", f"{license_plate:<15} {timestamp:<25}\n")
        log_text.see("end")
        log_text.config(state="disabled")

    except Exception as e:
        messagebox.showerror("錯誤", f"辨識失敗：{e}")

    if os.path.exists(filepath):
        os.remove(filepath)

#拍照按鈕
capture_btn = tk.Button(button_frame, text="拍照辨識", font=("Arial", 16), command=capture_and_detect)
capture_btn.pack(side="left", padx=20)

#停止按鈕
stop_btn = tk.Button(button_frame, text="離開", font=("Arial", 16), command=root.destroy)
stop_btn.pack(side="right", padx=20)

#攝影機即時畫面更新
cap = cv2.VideoCapture(0)
def update_camera():
    ret, frame = cap.read()
    if ret:
        frame_rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
        img = Image.fromarray(frame_rgb).resize((400, 300))
        imgtk = ImageTk.PhotoImage(image=img)
        image_label.imgtk = imgtk
        image_label.config(image=imgtk)
    root.after(30, update_camera)

root.after(0, update_camera)
root.mainloop()
cap.release()
