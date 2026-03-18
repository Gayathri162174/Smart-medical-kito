# Smart-medical-kito
Giving alert messageto the doctors ,family members and gaurdian
import streamlit as st
import pandas as pd
import numpy as np
import time
import smtplib
from email.mime.text import MimeText
from email.mime.multipart import MIMEMultipart
import threading

# Thresholds (customize)
THRESHOLDS = {
    'Heart Rate (bpm)': (60, 100),
    'Blood Pressure Sys (mmHg)': (90, 120),
    'Temperature (°C)': (36, 38),
    'SpO2 (%)': (95, 100)
}

# Recipients (add emails)
RECIPIENTS = ['doctor@example.com', 'family@example.com', 'guardian@example.com']

# Email config (use your Gmail)
EMAIL = 'yourgmail@gmail.com'
PASSWORD = 'your_app_password'  # Generate app password in Google settings

def send_alert(patient_id, vitals):
    msg = MIMEMultipart()
    msg['From'] = EMAIL
    msg['To'] = ', '.join(RECIPIENTS)
    msg['Subject'] = f'EMERGENCY: Patient {patient_id} Abnormal Vitals'
    
    body = f'Patient {patient_id} vitals abnormal:\n'
    for vital, value in vitals.items():
        body += f'{vital}: {value}\n'
    
    msg.attach(MimeText(body, 'plain'))
    
    try:
        server = smtplib.SMTP('smtp.gmail.com', 587)
        server.starttls()
        server.login(EMAIL, PASSWORD)
        text = msg.as_string()
        server.sendmail(EMAIL, RECIPIENTS, text)
        server.quit()
        print("Alert sent!")
    except Exception as e:
        print(f"Email error: {e}")

def check_emergency(vitals):
    alerts = {}
    for vital, (low, high) in THRESHOLDS.items():
        value = vitals[vital]
        if value < low or value > high:
            alerts[vital] = value
    return alerts

# Streamlit App
st.title("Patient Vital Monitor with Alerts")
patient_id = st.text_input("Patient ID", "P001")

if st.button("Start Monitoring"):
    placeholder = st.empty()
    alert_log = st.empty()
    
    for _ in range(100):  # Run 100 cycles
        # Simulate vitals (replace with sensor read)
        vitals = {
            'Heart Rate (bpm)': np.random.normal(80, 10),
            'Blood Pressure Sys (mmHg)': np.random.normal(110, 10),
            'Temperature (°C)': np.random.normal(37, 0.5),
            'SpO2 (%)': np.random.normal(98, 1)
        }
        
        df = pd.DataFrame([vitals])
        placeholder.dataframe(df)
        
        alerts = check_emergency(vitals)
        if alerts:
            alert_log.warning(f"🚨 Emergency for {patient_id}: {alerts}")
            # Thread for non-blocking alert
            threading.Thread(target=send_alert, args=(patient_id, alerts)).start()
        else:
            alert_log.success("Vitals normal.")
        
        time.sleep(2)  # 2s interval
