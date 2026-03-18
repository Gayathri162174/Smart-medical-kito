# Smart-medical-kito
Giving alert messageto the doctors ,family members and gaurdian
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Patient Vital Monitor Dashboard</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { 
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; 
            background: linear-gradient(135deg, #0f2027, #203a43, #2c5364);
            color: white; 
            min-height: 100vh; 
            padding: 20px;
        }
        .container { max-width: 1200px; margin: 0 auto; }
        h1 { text-align: center; margin-bottom: 30px; color: #00d4ff; }
        
        .controls { display: flex; gap: 20px; justify-content: center; margin-bottom: 30px; flex-wrap: wrap; }
        input, button { padding: 12px 20px; border: none; border-radius: 8px; font-size: 16px; }
        input { background: #2c5364; color: white; width: 200px; }
        button { background: #00d4ff; color: #0f2027; cursor: pointer; font-weight: bold; }
        button:hover { background: #00b8d4; }
        button:disabled { background: #666; cursor: not-allowed; }
        
        .vitals-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(280px, 1fr)); gap: 20px; margin-bottom: 30px; }
        .vital-card { 
            background: rgba(255,255,255,0.1); backdrop-filter: blur(10px); 
            padding: 25px; border-radius: 15px; text-align: center; 
            border: 2px solid rgba(255,255,255,0.2); transition: all 0.3s;
        }
        .vital-card.normal { border-color: #4ecdc4; box-shadow: 0 10px 30px rgba(78,205,196,0.3); }
        .vital-card.emergency { border-color: #ff6b6b; box-shadow: 0 10px 30px rgba(255,107,107,0.5); animation: pulse 1s infinite; }
        @keyframes pulse { 0%,100%{transform:scale(1);} 50%{transform:scale(1.05);} }
        
        .vital-value { font-size: 2.5em; font-weight: bold; margin: 10px 0; }
        .normal .vital-value { color: #4ecdc4; }
        .emergency .vital-value { color: #ff6b6b; }
        .vital-label { font-size: 1.1em; opacity: 0.9; margin-bottom: 5px; }
        .vital-status { font-size: 0.9em; padding: 5px 10px; border-radius: 20px; margin-top: 10px; }
        
        .alert-section { 
            background: rgba(255,107,107,0.2); padding: 20px; border-radius: 10px; 
            display: none; text-align: center; margin-top: 20px;
        }
        .alert-section.active { display: block; animation: slideDown 0.5s; }
        @keyframes slideDown { from { opacity: 0; transform: translateY(-20px); } to { opacity: 1; transform: translateY(0); } }
        
        .notification { 
            position: fixed; top: 20px; right: 20px; background: #ff6b6b; 
            color: white; padding: 15px 20px; border-radius: 8px; 
            box-shadow: 0 5px 20px rgba(255,107,107,0.4); display: none;
        }
        .notification.show { display: block; animation: slideIn 0.3s; }
        @keyframes slideIn { from { transform: translateX(100%); } to { transform: translateX(0); } }
        
        .recipients { display: flex; gap: 10px; flex-wrap: wrap; justify-content: center; margin-top: 10px; }
        .recipient { background: rgba(255,255,255,0.2); padding: 8px 15px; border-radius: 20px; font-size: 0.9em; }
    </style>
</head>
<body>
    <div class="container">
        <h1>🩺 Patient Vital Signs Monitor</h1>
        
        <div class="controls">
            <input type="text" id="patientId" placeholder="Enter Patient ID (e.g., P001)" value="P001">
            <button id="startBtn">▶️ Start Monitoring</button>
            <button id="stopBtn" disabled>⏹️ Stop</button>
        </div>
        
        <div class="vitals-grid" id="vitalsGrid">
            <!-- Vitals cards populated by JS -->
        </div>
        
        <div class="alert-section" id="alertSection">
            <h3>🚨 EMERGENCY ALERT</h3>
            <p id="alertMessage"></p>
            <div class="recipients" id="recipientsList"></div>
        </div>
    </div>
    
    <div class="notification" id="toast">Alert sent to Doctor, Family & Guardian!</div>

    <script>
        const THRESHOLDS = {
            'Heart Rate (bpm)': { min: 60, max: 100, unit: 'bpm' },
            'Blood Pressure Sys (mmHg)': { min: 90, max: 120, unit: 'mmHg' },
            'Temperature (°C)': { min: 36, max: 38, unit: '°C' },
            'SpO2 (%)': { min: 95, max: 100, unit: '%' }
        };

        const RECIPIENTS = ['doctor@hospital.com', 'family@gmail.com', 'guardian@yahoo.com'];
        
        let monitoring = false;
        let intervalId;
        const vitalsGrid = document.getElementById('vitalsGrid');
        const startBtn = document.getElementById('startBtn');
        const stopBtn = document.getElementById('stopBtn');
        const patientIdInput = document.getElementById('patientId');
        const alertSection = document.getElementById('alertSection');
        const alertMessage = document.getElementById('alertMessage');
        const recipientsList = document.getElementById('recipientsList');
        const toast = document.getElementById('toast');

        // Create vital cards
        Object.keys(THRESHOLDS).forEach(vital => {
            const card = document.createElement('div');
            card.className = 'vital-card';
            card.innerHTML = `
                <div class="vital-label">${vital}</div>
                <div class="vital-value">--</div>
                <div class="vital-status">Normal</div>
            `;
            vitalsGrid.appendChild(card);
        });

        function generateVitals() {
            const vitals = {};
            Object.keys(THRESHOLDS).forEach(vital => {
                const { min, max } = THRESHOLDS[vital];
                // 10% chance of emergency
                if (Math.random() < 0.1) {
                    vitals[vital] = Math.random() < 0.5 ? min - 10 : max + 10;
                } else {
                    vitals[vital] = min + Math.random() * (max - min);
                }
            });
            return vitals;
        }

        function updateVitals(vitals) {
            const cards = vitalsGrid.children;
            let emergency = false;
            let emergencyVitals = {};

            Array.from(cards).forEach((card, i) => {
                const vitalName = Object.keys(THRESHOLDS)[i];
                const value = vitals[vitalName].toFixed(1);
                const { min, max } = THRESHOLDS[vitalName];
                const valueNum = parseFloat(value);

                card.querySelector('.vital-value').textContent = value;
                
                if (valueNum < min || valueNum > max) {
                    card.className = 'vital-card emergency';
                    card.querySelector('.vital-status').textContent = 'EMERGENCY';
                    emergency = true;
                    emergencyVitals[vitalName] = value;
                } else {
                    card.className = 'vital-card normal';
                    card.querySelector('.vital-status').textContent = 'Normal';
                }
            });

            if (emergency) {
                triggerEmergency(emergencyVitals);
            }
        }

        function triggerEmergency(vitals) {
            const patientId = patientIdInput.value || 'Unknown';
            alertMessage.innerHTML = `Patient <strong>${patientId}</strong> has critical vitals:<br>` + 
                Object.entries(vitals).map(([key, val]) => `<strong>${key}:</strong> ${val}`).join('<br>');
            
            recipientsList.innerHTML = RECIPIENTS.map(email => 
                `<span class="recipient">📧 ${email}</span>`
            ).join('');
            
            alertSection.classList.add('active');
            showToast();
            
            // Simulate notification sound
            const audio = new Audio('data:audio/wav;base64,UklGRnoGAABXQVZFZm10IBAAAAABAAEAQB8AAEAfAAABAAgAZGF0YQoGAACBhYqFbF1fdJivrJBhNjVgodDbq2EcBj+a2/LDciUFLIHO8tiJNwgZaLvt559NEAxQp+PwtmMcBjiR1/LMeSwFJHfH8N2QQAo');
            audio.play().catch(() => {});
        }

        function showToast() {
            toast.classList.add('show');
            setTimeout(() => toast.classList.remove('show'), 4000);
        }

        startBtn.addEventListener('click', () => {
            if (monitoring) return;
            monitoring = true;
            startBtn.disabled = true;
            stopBtn.disabled = false;
            alertSection.classList.remove('active');
            
            intervalId = setInterval(() => {
                const vitals = generateVitals();
                updateVitals(vitals);
            }, 2000);
        });

        stopBtn.addEventListener('click', () => {
            monitoring = false;
            clearInterval(intervalId);
            startBtn.disabled = false;
            stopBtn.disabled = true;
            alertSection.classList.remove('active');
        });

        // Auto-start demo
        setTimeout(() => {
            if (!monitoring) startBtn.click();
        }, 1000);
    </script>
</body>
</html>
