# 🚀 WiFi Portal "Redmi Note 13" - Instalación en Un Solo Comando

Copia y pega este comando completo en tu Raspberry Pi Zero 2W para crear todo el proyecto:

```bash
cd ~ && mkdir -p wifi-portal/app/{static,templates} wifi-portal/data && \
cat > wifi-portal/app/main.py << 'EOF'
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
import os
import json
import time
import random
import string
from datetime import datetime
from threading import Thread
from flask import Flask, render_template, request, redirect, url_for, jsonify
import subprocess

app = Flask(__name__)

DATA_DIR = os.path.join(os.path.dirname(os.path.dirname(__file__)), 'data')
USERS_FILE = os.path.join(DATA_DIR, 'users.json')
CODES_FILE = os.path.join(DATA_DIR, 'codes.json')

ADMIN_CODE = "femboy"
SESSION_DURATION = 3600  # 1 hora en segundos
COOLDOWN_DURATION = 14400  # 4 horas en segundos
REDUCED_COOLDOWN = 7200  # 2 horas en segundos

def load_json(filepath):
    if not os.path.exists(filepath):
        return {}
    try:
        with open(filepath, 'r') as f:
            return json.load(f)
    except:
        return {}

def save_json(filepath, data):
    with open(filepath, 'w') as f:
        json.dump(data, f, indent=2)

def get_mac_from_ip(ip):
    try:
        result = subprocess.run(['arp', '-n', ip], capture_output=True, text=True)
        for line in result.stdout.split('\n'):
            if ip in line:
                parts = line.split()
                if len(parts) >= 3:
                    return parts[2]
    except:
        pass
    return None

def allow_mac_internet(mac):
    try:
        subprocess.run(['sudo', 'iptables', '-t', 'nat', '-A', 'PREROUTING', '-m', 'mac', '--mac-source', mac, '-j', 'ACCEPT'], check=False)
        subprocess.run(['sudo', 'iptables', '-I', 'FORWARD', '1', '-m', 'mac', '--mac-source', mac, '-j', 'ACCEPT'], check=False)
    except:
        pass

def block_mac_internet(mac):
    try:
        subprocess.run(['sudo', 'iptables', '-t', 'nat', '-D', 'PREROUTING', '-m', 'mac', '--mac-source', mac, '-j', 'ACCEPT'], check=False)
        subprocess.run(['sudo', 'iptables', '-D', 'FORWARD', '-m', 'mac', '--mac-source', mac, '-j', 'ACCEPT'], check=False)
    except:
        pass

def cleanup_expired_sessions():
    users = load_json(USERS_FILE)
    now = time.time()
    changed = False
    
    for mac, data in list(users.items()):
        if data.get('active') and data.get('session_end', 0) < now:
            data['active'] = False
            data['cooldown_end'] = now + data.get('cooldown_duration', COOLDOWN_DURATION)
            block_mac_internet(mac)
            changed = True
        
        # Limpiar cooldowns expirados
        if not data.get('active') and data.get('cooldown_end', 0) < now:
            del users[mac]
            changed = True
    
    if changed:
        save_json(USERS_FILE, users)

def get_user_status(mac):
    users = load_json(USERS_FILE)
    now = time.time()
    
    if mac not in users:
        return {'status': 'new', 'remaining': 0}
    
    user = users[mac]
    
    if user.get('active'):
        remaining = max(0, user.get('session_end', 0) - now)
        if remaining > 0:
            return {'status': 'active', 'remaining': int(remaining)}
        else:
            user['active'] = False
            user['cooldown_end'] = now + user.get('cooldown_duration', COOLDOWN_DURATION)
            save_json(USERS_FILE, users)
            block_mac_internet(mac)
            return {'status': 'cooldown', 'remaining': int(user['cooldown_end'] - now)}
    
    cooldown_remaining = user.get('cooldown_end', 0) - now
    if cooldown_remaining > 0:
        return {'status': 'cooldown', 'remaining': int(cooldown_remaining)}
    
    return {'status': 'ready', 'remaining': 0}

@app.route('/')
def index():
    cleanup_expired_sessions()
    client_ip = request.remote_addr
    mac = get_mac_from_ip(client_ip)
    
    if not mac:
        return render_template('index.html', error=None)
    
    status = get_user_status(mac)
    
    if status['status'] == 'active':
        return redirect(url_for('connected'))
    elif status['status'] == 'cooldown':
        return redirect(url_for('cooldown'))
    
    return render_template('index.html', error=None)

@app.route('/connect', methods=['POST'])
def connect():
    if not request.form.get('accept_terms'):
        return redirect(url_for('index'))
    
    client_ip = request.remote_addr
    mac = get_mac_from_ip(client_ip)
    
    if not mac:
        return render_template('index.html', error="No se pudo identificar tu dispositivo")
    
    status = get_user_status(mac)
    
    if status['status'] == 'cooldown':
        return redirect(url_for('cooldown'))
    
    if status['status'] == 'active':
        return redirect(url_for('connected'))
    
    # Iniciar nueva sesión
    users = load_json(USERS_FILE)
    now = time.time()
    
    session_time = SESSION_DURATION
    # Verificar si hay tiempo extra almacenado
    if mac in users and users[mac].get('extra_time', 0) > 0:
        session_time += users[mac]['extra_time']
        extra_time = 0
    else:
        extra_time = 0
    
    users[mac] = {
        'active': True,
        'session_start': now,
        'session_end': now + session_time,
        'session_duration': session_time,
        'cooldown_duration': users.get(mac, {}).get('cooldown_duration', COOLDOWN_DURATION),
        'extra_time': extra_time,
        'ip': client_ip
    }
    
    save_json(USERS_FILE, users)
    allow_mac_internet(mac)
    
    return redirect(url_for('connected'))

@app.route('/connected')
def connected():
    cleanup_expired_sessions()
    client_ip = request.remote_addr
    mac = get_mac_from_ip(client_ip)
    
    if not mac:
        return redirect(url_for('index'))
    
    status = get_user_status(mac)
    
    if status['status'] != 'active':
        return redirect(url_for('index'))
    
    hours = status['remaining'] // 3600
    minutes = (status['remaining'] % 3600) // 60
    seconds = status['remaining'] % 60
    
    return render_template('connected.html', 
                         hours=hours, 
                         minutes=minutes, 
                         seconds=seconds,
                         total_seconds=status['remaining'])

@app.route('/cooldown')
def cooldown():
    cleanup_expired_sessions()
    client_ip = request.remote_addr
    mac = get_mac_from_ip(client_ip)
    
    if not mac:
        return redirect(url_for('index'))
    
    status = get_user_status(mac)
    
    if status['status'] == 'active':
        return redirect(url_for('connected'))
    
    if status['status'] != 'cooldown':
        return redirect(url_for('index'))
    
    hours = status['remaining'] // 3600
    minutes = (status['remaining'] % 3600) // 60
    
    return render_template('cooldown.html', hours=hours, minutes=minutes)

@app.route('/redeem', methods=['POST'])
def redeem():
    code = request.form.get('code', '').strip()
    
    if code == ADMIN_CODE:
        return redirect(url_for('admin'))
    
    client_ip = request.remote_addr
    mac = get_mac_from_ip(client_ip)
    
    if not mac:
        return redirect(url_for('index'))
    
    codes = load_json(CODES_FILE)
    
    if code not in codes:
        return render_template('index.html', error="Código inválido")
    
    if codes[code].get('used'):
        return render_template('index.html', error="Este código ya fue utilizado")
    
    # Marcar código como usado
    codes[code]['used'] = True
    codes[code]['used_by'] = mac
    codes[code]['used_at'] = datetime.now().isoformat()
    save_json(CODES_FILE, codes)
    
    users = load_json(USERS_FILE)
    code_type = codes[code]['type']
    
    if code_type == 'extra_time':
        # Agregar 1 hora extra
        if mac in users and users[mac].get('active'):
            users[mac]['session_end'] += 3600
            users[mac]['session_duration'] += 3600
        else:
            # Guardar para la próxima sesión
            if mac not in users:
                users[mac] = {}
            users[mac]['extra_time'] = users[mac].get('extra_time', 0) + 3600
        message = "¡Código canjeado! Se agregó 1 hora extra"
    else:  # reduced_cooldown
        # Reducir cooldown a 2 horas
        if mac not in users:
            users[mac] = {}
        users[mac]['cooldown_duration'] = REDUCED_COOLDOWN
        message = "¡Código canjeado! Tu próximo cooldown será de solo 2 horas"
    
    save_json(USERS_FILE, users)
    
    status = get_user_status(mac)
    if status['status'] == 'active':
        return redirect(url_for('connected'))
    
    return render_template('index.html', success=message)

@app.route('/admin')
def admin():
    users = load_json(USERS_FILE)
    codes = load_json(CODES_FILE)
    
    active_users = []
    now = time.time()
    
    for mac, data in users.items():
        if data.get('active'):
            remaining = max(0, data.get('session_end', 0) - now)
            active_users.append({
                'mac': mac,
                'ip': data.get('ip', 'N/A'),
                'remaining': remaining
            })
    
    code_list = []
    for code, data in codes.items():
        code_list.append({
            'code': code,
            'type': data['type'],
            'used': data.get('used', False),
            'used_by': data.get('used_by', 'N/A'),
            'created_at': data.get('created_at', 'N/A')
        })
    
    return render_template('admin.html', 
                         active_users=active_users, 
                         codes=code_list,
                         total_codes=len(code_list))

@app.route('/admin/generate', methods=['POST'])
def generate_code():
    code_type = request.form.get('type', 'extra_time')
    
    codes = load_json(CODES_FILE)
    
    # Generar código aleatorio de 8 caracteres
    new_code = ''.join(random.choices(string.ascii_uppercase + string.digits, k=8))
    
    # Asegurar que sea único
    while new_code in codes:
        new_code = ''.join(random.choices(string.ascii_uppercase + string.digits, k=8))
    
    codes[new_code] = {
        'type': code_type,
        'used': False,
        'created_at': datetime.now().isoformat()
    }
    
    save_json(CODES_FILE, codes)
    
    return redirect(url_for('admin'))

@app.route('/status')
def status():
    client_ip = request.remote_addr
    mac = get_mac_from_ip(client_ip)
    
    if not mac:
        return jsonify({'error': 'No MAC found'}), 404
    
    return jsonify(get_user_status(mac))

def background_cleanup():
    """Tarea en segundo plano para limpiar sesiones expiradas cada 5 minutos"""
    while True:
        try:
            cleanup_expired_sessions()
        except Exception:
            pass
        time.sleep(300)  # 5 minutos

if __name__ == '__main__':
    # Crear archivos de datos si no existen
    os.makedirs(DATA_DIR, exist_ok=True)
    if not os.path.exists(USERS_FILE):
        save_json(USERS_FILE, {})
    if not os.path.exists(CODES_FILE):
        save_json(CODES_FILE, {})
    
    # Limpieza inicial y arranque de tarea en segundo plano
    cleanup_expired_sessions()
    cleanup_thread = Thread(target=background_cleanup, daemon=True)
    cleanup_thread.start()
    
    app.run(host='0.0.0.0', port=80, debug=False)
EOF

cat > wifi-portal/app/static/style.css << 'EOF'
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

body {
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    min-height: 100vh;
    display: flex;
    justify-content: center;
    align-items: center;
    padding: 20px;
}

.container {
    background: white;
    border-radius: 20px;
    box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
    max-width: 500px;
    width: 100%;
    padding: 40px;
    text-align: center;
}

h1 {
    color: #333;
    margin-bottom: 10px;
    font-size: 28px;
}

.subtitle {
    color: #666;
    margin-bottom: 30px;
    font-size: 14px;
}

.alert {
    padding: 15px;
    border-radius: 10px;
    margin-bottom: 20px;
    font-size: 14px;
}

.alert-error {
    background: #fee;
    color: #c33;
    border: 1px solid #fcc;
}

.alert-success {
    background: #efe;
    color: #3c3;
    border: 1px solid #cfc;
}

.terms {
    background: #f5f5f5;
    border-radius: 10px;
    padding: 20px;
    margin: 20px 0;
    max-height: 300px;
    overflow-y: auto;
    text-align: left;
}

.terms h3 {
    color: #333;
    margin-bottom: 15px;
    font-size: 16px;
}

.terms ol {
    margin-left: 20px;
}

.terms li {
    color: #555;
    margin-bottom: 10px;
    font-size: 13px;
    line-height: 1.6;
}

.checkbox-container {
    display: flex;
    align-items: center;
    justify-content: center;
    margin: 20px 0;
    gap: 10px;
}

.checkbox-container input[type="checkbox"] {
    width: 20px;
    height: 20px;
    cursor: pointer;
}

.checkbox-container label {
    color: #333;
    font-size: 14px;
    cursor: pointer;
}

.btn {
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    color: white;
    border: none;
    padding: 15px 30px;
    border-radius: 50px;
    font-size: 16px;
    font-weight: bold;
    cursor: pointer;
    width: 100%;
    margin: 10px 0;
    transition: transform 0.2s, box-shadow 0.2s;
}

.btn:hover {
    transform: translateY(-2px);
    box-shadow: 0 10px 20px rgba(0, 0, 0, 0.2);
}

.btn:active {
    transform: translateY(0);
}

.btn:disabled {
    opacity: 0.5;
    cursor: not-allowed;
}

.input-group {
    margin: 20px 0;
}

.input-group label {
    display: block;
    color: #666;
    font-size: 14px;
    margin-bottom: 8px;
    text-align: left;
}

.input-group input[type="text"] {
    width: 100%;
    padding: 12px;
    border: 2px solid #ddd;
    border-radius: 10px;
    font-size: 14px;
    transition: border 0.3s;
}

.input-group input[type="text"]:focus {
    outline: none;
    border-color: #667eea;
}

.time-display {
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    color: white;
    padding: 40px;
    border-radius: 15px;
    margin: 30px 0;
}

.time-display h2 {
    font-size: 48px;
    margin-bottom: 10px;
    font-weight: bold;
}

.time-display p {
    font-size: 16px;
    opacity: 0.9;
}

.info-box {
    background: #f0f4ff;
    border-left: 4px solid #667eea;
    padding: 15px;
    margin: 20px 0;
    border-radius: 5px;
    text-align: left;
}

.info-box p {
    color: #555;
    font-size: 13px;
    line-height: 1.6;
}

.admin-section {
    margin: 30px 0;
    padding: 20px;
    background: #f9f9f9;
    border-radius: 10px;
}

.admin-section h2 {
    color: #333;
    margin-bottom: 20px;
    font-size: 20px;
}

table {
    width: 100%;
    border-collapse: collapse;
    margin: 20px 0;
    background: white;
    border-radius: 10px;
    overflow: hidden;
}

th, td {
    padding: 12px;
    text-align: left;
    border-bottom: 1px solid #eee;
    font-size: 13px;
}

th {
    background: #667eea;
    color: white;
    font-weight: bold;
}

tr:hover {
    background: #f5f5f5;
}

.badge {
    display: inline-block;
    padding: 4px 12px;
    border-radius: 20px;
    font-size: 11px;
    font-weight: bold;
}

.badge-success {
    background: #d4edda;
    color: #155724;
}

.badge-danger {
    background: #f8d7da;
    color: #721c24;
}

.badge-info {
    background: #d1ecf1;
    color: #0c5460;
}

select {
    padding: 10px;
    border: 2px solid #ddd;
    border-radius: 10px;
    font-size: 14px;
    margin-right: 10px;
}

.footer {
    margin-top: 30px;
    padding-top: 20px;
    border-top: 1px solid #eee;
    color: #999;
    font-size: 12px;
}

@media (max-width: 600px) {
    .container {
        padding: 25px;
    }
    
    h1 {
        font-size: 24px;
    }
    
    .time-display h2 {
        font-size: 36px;
    }
}
EOF

cat > wifi-portal/app/templates/index.html << 'EOF'
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Redmi Note 13 WiFi</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
</head>
<body>
    <div class="container">
        <h1>📱 Redmi Note 13</h1>
        <p class="subtitle">WiFi Gratuito - 1 hora de acceso</p>
        
        {% if error %}
        <div class="alert alert-error">
            {{ error }}
        </div>
        {% endif %}
        
        {% if success %}
        <div class="alert alert-success">
            {{ success }}
        </div>
        {% endif %}
        
        <div class="terms">
            <h3>📋 Términos y Condiciones</h3>
            <ol>
                <li>Esta wifi puede que no funcione correctamente con el IMT Lazarus.</li>
                <li>NO aceptamos devoluciones.</li>
                <li>Estamos en total derecho de vetar a cualquier usuario sin previo aviso.</li>
                <li>Podemos recoger datos anónimos.</li>
                <li>No se permite el uso de la red para actividades ilegales.</li>
                <li>El tiempo de conexión no se puede pausar bajo ningún concepto.</li>
                <li>Última actualización: 29/10/2025.</li>
            </ol>
        </div>
        
        <form method="POST" action="/connect" id="connectForm">
            <div class="checkbox-container">
                <input type="checkbox" id="accept_terms" name="accept_terms" required>
                <label for="accept_terms">Acepto los términos y condiciones</label>
            </div>
            
            <button type="submit" class="btn" id="connectBtn">
                🚀 Conectar
            </button>
        </form>
        
        <form method="POST" action="/redeem">
            <div class="input-group">
                <label for="code">¿Tienes un código?</label>
                <input type="text" id="code" name="code" placeholder="Ingresa tu código aquí">
            </div>
            <button type="submit" class="btn" style="background: linear-gradient(135deg, #f093fb 0%, #f5576c 100%);">
                🎁 Canjear Código
            </button>
        </form>
        
        <div class="footer">
            <p>Conexión segura y privada</p>
        </div>
    </div>
    
    <script>
        const checkbox = document.getElementById('accept_terms');
        const btn = document.getElementById('connectBtn');
        
        checkbox.addEventListener('change', function() {
            btn.disabled = !this.checked;
        });
        
        // Inicializar estado del botón
        btn.disabled = !checkbox.checked;
    </script>
</body>
</html>
EOF

cat > wifi-portal/app/templates/connected.html << 'EOF'
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Conectado - Redmi Note 13</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
    <meta http-equiv="refresh" content="30">
</head>
<body>
    <div class="container">
        <h1>✅ ¡Conectado!</h1>
        <p class="subtitle">Disfruta tu navegación</p>
        
        <div class="time-display">
            <h2 id="timeRemaining">{{ "%02d:%02d:%02d" | format(hours, minutes, seconds) }}</h2>
            <p>Tiempo restante</p>
        </div>
        
        <div class="info-box">
            <p><strong>ℹ️ Información importante:</strong></p>
            <p>• Tu sesión está activa y el tiempo no se puede pausar.</p>
            <p>• Esta página se actualiza automáticamente cada 30 segundos.</p>
            <p>• Al finalizar tu tiempo, entrarás en período de espera de 4 horas.</p>
        </div>
        
        <div class="footer">
            <p>Conexión activa desde tu dispositivo</p>
        </div>
    </div>
    
    <script>
        let secondsRemaining = {{ total_seconds }};
        
        function updateTimer() {
            if (secondsRemaining <= 0) {
                window.location.reload();
                return;
            }
            
            const hours = Math.floor(secondsRemaining / 3600);
            const minutes = Math.floor((secondsRemaining % 3600) / 60);
            const seconds = secondsRemaining % 60;
            
            document.getElementById('timeRemaining').textContent = 
                String(hours).padStart(2, '0') + ':' + 
                String(minutes).padStart(2, '0') + ':' + 
                String(seconds).padStart(2, '0');
            
            secondsRemaining--;
        }
        
        setInterval(updateTimer, 1000);
    </script>
</body>
</html>
EOF

cat > wifi-portal/app/templates/cooldown.html << 'EOF'
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>En Espera - Redmi Note 13</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
    <meta http-equiv="refresh" content="60">
</head>
<body>
    <div class="container">
        <h1>⏳ Período de Espera</h1>
        <p class="subtitle">Tu tiempo de conexión ha terminado</p>
        
        <div class="time-display" style="background: linear-gradient(135deg, #f093fb 0%, #f5576c 100%);">
            <h2>Vuelve en {{ "%d:%02d" | format(hours, minutes) }}</h2>
            <p>{{ hours }} hora{{ 's' if hours != 1 else '' }} y {{ minutes }} minuto{{ 's' if minutes != 1 else '' }}</p>
        </div>
        
        <div class="info-box">
            <p><strong>⏰ ¿Por qué debo esperar?</strong></p>
            <p>Para garantizar que todos los usuarios tengan acceso justo a la red, implementamos un período de espera de 4 horas entre sesiones.</p>
            <p><strong>💡 Consejo:</strong> Puedes obtener un código especial para reducir este tiempo de espera a solo 2 horas.</p>
        </div>
        
        <form method="POST" action="/redeem">
            <div class="input-group">
                <label for="code">¿Tienes un código para reducir el tiempo?</label>
                <input type="text" id="code" name="code" placeholder="Ingresa tu código aquí">
            </div>
            <button type="submit" class="btn" style="background: linear-gradient(135deg, #f093fb 0%, #f5576c 100%);">
                🎁 Canjear Código
            </button>
        </form>
        
        <div class="footer">
            <p>Esta página se actualiza automáticamente cada minuto</p>
        </div>
    </div>
</body>
</html>
EOF

cat > wifi-portal/app/templates/admin.html << 'EOF'
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Panel Admin - Redmi Note 13</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
    <meta http-equiv="refresh" content="30">
</head>
<body>
    <div class="container" style="max-width: 900px;">
        <h1>🔧 Panel de Administración</h1>
        <p class="subtitle">Control total del WiFi Portal</p>
        
        <div class="admin-section">
            <h2>➕ Generar Nuevo Código</h2>
            <form method="POST" action="/admin/generate" style="display: flex; align-items: center; justify-content: center; gap: 10px;">
                <select name="type" required>
                    <option value="extra_time">+1 Hora Extra</option>
                    <option value="reduced_cooldown">Cooldown Reducido (2h)</option>
                </select>
                <button type="submit" class="btn" style="width: auto; padding: 10px 30px; margin: 0;">
                    Generar Código
                </button>
            </form>
        </div>
        
        <div class="admin-section">
            <h2>👥 Usuarios Activos ({{ active_users|length }})</h2>
            {% if active_users %}
            <table>
                <thead>
                    <tr>
                        <th>MAC Address</th>
                        <th>IP</th>
                        <th>Tiempo Restante</th>
                    </tr>
                </thead>
                <tbody>
                    {% for user in active_users %}
                    <tr>
                        <td>{{ user.mac }}</td>
                        <td>{{ user.ip }}</td>
                        <td>
                            {% set hours = user.remaining // 3600 %}
                            {% set minutes = (user.remaining % 3600) // 60 %}
                            {{ "%d:%02d" | format(hours, minutes) }}
                        </td>
                    </tr>
                    {% endfor %}
                </tbody>
            </table>
            {% else %}
            <p style="color: #999; text-align: center; padding: 20px;">No hay usuarios conectados actualmente</p>
            {% endif %}
        </div>
        
        <div class="admin-section">
            <h2>🎫 Códigos Generados ({{ total_codes }})</h2>
            {% if codes %}
            <table>
                <thead>
                    <tr>
                        <th>Código</th>
                        <th>Tipo</th>
                        <th>Estado</th>
                        <th>Usado Por</th>
                        <th>Creado</th>
                    </tr>
                </thead>
                <tbody>
                    {% for code in codes %}
                    <tr>
                        <td><strong>{{ code.code }}</strong></td>
                        <td>
                            {% if code.type == 'extra_time' %}
                            <span class="badge badge-info">+1 Hora</span>
                            {% else %}
                            <span class="badge badge-success">Cooldown 2h</span>
                            {% endif %}
                        </td>
                        <td>
                            {% if code.used %}
                            <span class="badge badge-danger">Usado</span>
                            {% else %}
                            <span class="badge badge-success">Disponible</span>
                            {% endif %}
                        </td>
                        <td>{{ code.used_by }}</td>
                        <td>{{ code.created_at[:10] if code.created_at != 'N/A' else 'N/A' }}</td>
                    </tr>
                    {% endfor %}
                </tbody>
            </table>
            {% else %}
            <p style="color: #999; text-align: center; padding: 20px;">No hay códigos generados aún</p>
            {% endif %}
        </div>
        
        <a href="/" class="btn" style="display: block; text-decoration: none; margin-top: 20px;">
            ← Volver al Portal
        </a>
        
        <div class="footer">
            <p>Panel actualizado automáticamente cada 30 segundos</p>
        </div>
    </div>
</body>
</html>
EOF

cat > wifi-portal/data/users.json << 'EOF'
{}
EOF

cat > wifi-portal/data/codes.json << 'EOF'
{}
EOF

cat > wifi-portal/setup.sh << 'EOF'
#!/bin/bash

echo "=========================================="
echo "  Instalación WiFi Portal Redmi Note 13  "
echo "=========================================="
echo ""

# Verificar que se ejecuta como root
if [ "$EUID" -ne 0 ]; then 
    echo "❌ Por favor ejecuta como root (usa sudo)"
    exit 1
fi

# Obtener el usuario real (no root)
REAL_USER=${SUDO_USER:-$USER}
PROJECT_DIR="/home/$REAL_USER/wifi-portal"

echo "📦 Actualizando sistema..."
apt-get update -qq

echo "📦 Instalando dependencias..."
apt-get install -y hostapd dnsmasq iptables python3 python3-pip python3-flask net-tools

echo "🛑 Deteniendo servicios..."
systemctl stop hostapd 2>/dev/null
systemctl stop dnsmasq 2>/dev/null

echo "⚙️  Configurando hostapd..."
cat > /etc/hostapd/hostapd.conf << 'HOSTAPD_EOF'
interface=wlan0
driver=nl80211
ssid=Redmi Note 13
hw_mode=g
channel=7
wmm_enabled=0
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=pezpezpezpez
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
HOSTAPD_EOF

echo "⚙️  Configurando dnsmasq..."
mv /etc/dnsmasq.conf /etc/dnsmasq.conf.backup 2>/dev/null
cat > /etc/dnsmasq.conf << 'DNSMASQ_EOF'
interface=wlan0
dhcp-range=192.168.4.2,192.168.4.20,255.255.255.0,24h
domain=wifiportal
address=/#/192.168.4.1
DNSMASQ_EOF

echo "⚙️  Configurando interfaz wlan0..."
cat >> /etc/dhcpcd.conf << 'DHCPCD_EOF'

# WiFi Portal Configuration
interface wlan0
static ip_address=192.168.4.1/24
nohook wpa_supplicant
DHCPCD_EOF

echo "🔥 Configurando firewall..."
# Habilitar IP forwarding
sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
sysctl -w net.ipv4.ip_forward=1

# Limpiar reglas existentes
iptables -F
iptables -t nat -F
iptables -X

# NAT para compartir internet (wlan1 o eth0 según disponibilidad)
if ip link show wlan1 &> /dev/null; then
    WAN_IFACE="wlan1"
elif ip link show eth0 &> /dev/null; then
    WAN_IFACE="eth0"
else
    WAN_IFACE="wlan1"  # Default fallback
fi

echo "🌐 Usando $WAN_IFACE como interfaz WAN"

iptables -t nat -A POSTROUTING -o $WAN_IFACE -j MASQUERADE
iptables -A FORWARD -i $WAN_IFACE -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT

# Por defecto bloquear todo desde wlan0
iptables -A FORWARD -i wlan0 -o $WAN_IFACE -j DROP

# Redirigir todo el tráfico HTTP al portal (excepto el que ya está autorizado)
iptables -t nat -A PREROUTING -i wlan0 -p tcp --dport 80 -j DNAT --to-destination 192.168.4.1:80
iptables -t nat -A PREROUTING -i wlan0 -p tcp --dport 443 -j DNAT --to-destination 192.168.4.1:80

# Guardar reglas de iptables
iptables-save > /etc/iptables.rules

# Restaurar reglas en boot
cat > /etc/network/if-pre-up.d/iptables << 'IPTABLES_RESTORE_EOF'
#!/bin/sh
iptables-restore < /etc/iptables.rules
exit 0
IPTABLES_RESTORE_EOF
chmod +x /etc/network/if-pre-up.d/iptables

echo "🐍 Configurando servicio systemd..."
cat > /etc/systemd/system/wifi-portal.service << SYSTEMD_EOF
[Unit]
Description=WiFi Portal Redmi Note 13
After=network.target hostapd.service dnsmasq.service

[Service]
Type=simple
User=root
WorkingDirectory=$PROJECT_DIR/app
ExecStart=/usr/bin/python3 $PROJECT_DIR/app/main.py
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
SYSTEMD_EOF

echo "🔁 Recargando systemd..."
systemctl daemon-reload

echo "✅ Habilitando servicios..."
systemctl unmask hostapd
systemctl enable hostapd
systemctl enable dnsmasq
systemctl enable wifi-portal

echo "🚀 Iniciando servicios..."
systemctl start hostapd
systemctl start dnsmasq
systemctl start wifi-portal

echo ""
echo "=========================================="
echo "  ✅ Instalación Completada               "
echo "=========================================="
echo ""
echo "📱 SSID: Redmi Note 13"
echo "🔐 Contraseña: pezpezpezpez"
echo "🌐 Portal: http://192.168.4.1"
echo "🔧 Panel Admin: Usa el código 'femboy'"
echo ""
echo "📊 Estado de servicios:"
systemctl status hostapd --no-pager | grep Active
systemctl status dnsmasq --no-pager | grep Active
systemctl status wifi-portal --no-pager | grep Active
echo ""
echo "💡 Reinicia la Raspberry Pi para asegurar que todo funcione correctamente:"
echo "   sudo reboot"
echo ""
EOF

chmod +x wifi-portal/setup.sh && \

cat > wifi-portal/README.md << 'EOF'
# 📱 WiFi Portal "Redmi Note 13" para Raspberry Pi Zero 2W

Portal cautivo completo con sistema de tiempo y cooldowns para Raspberry Pi.

## 🚀 Características

- ✅ Portal cautivo automático
- ⏱️ 1 hora de acceso por dispositivo (MAC)
- ⏳ 4 horas de cooldown entre sesiones
- 🎁 Sistema de códigos:
  - Códigos de +1 hora extra
  - Códigos de cooldown reducido (2h en lugar de 4h)
- 🔧 Panel de administración completo
- 📊 Seguimiento en tiempo real de usuarios
- 🌐 Interfaz completamente en español

## 📋 Requisitos

- Raspberry Pi Zero 2W (u otros modelos con WiFi)
- Raspberry Pi OS Lite (Debian-based)
- Tarjeta SD con al menos 4GB
- Conexión a internet (vía Ethernet o segundo adaptador WiFi)

## ⚡ Instalación Rápida

```bash
# Clonar o descargar este proyecto
cd ~/wifi-portal

# Ejecutar instalación
sudo bash setup.sh

# Reiniciar
sudo reboot
```

## 🔧 Configuración

### Cambiar SSID y Contraseña

Edita `/etc/hostapd/hostapd.conf`:

```bash
sudo nano /etc/hostapd/hostapd.conf
```

Modifica:
- `ssid=Redmi Note 13` → tu nombre deseado
- `wpa_passphrase=pezpezpezpez` → tu contraseña

Reinicia hostapd:
```bash
sudo systemctl restart hostapd
```

### Cambiar Código de Admin

Edita `app/main.py` y modifica:
```python
ADMIN_CODE = "femboy"  # Cambia esto
```

Reinicia el servicio:
```bash
sudo systemctl restart wifi-portal
```

## 📊 Panel de Administración

1. Conéctate al WiFi "Redmi Note 13"
2. En el portal, ingresa el código: `femboy`
3. Accederás al panel donde puedes:
   - Generar códigos de +1 hora
   - Generar códigos de cooldown reducido
   - Ver usuarios activos
   - Ver historial de códigos

## 🛠️ Comandos Útiles

```bash
# Ver estado de servicios
sudo systemctl status wifi-portal
sudo systemctl status hostapd
sudo systemctl status dnsmasq

# Ver logs del portal
sudo journalctl -u wifi-portal -f

# Reiniciar portal
sudo systemctl restart wifi-portal

# Ver usuarios conectados
cat ~/wifi-portal/data/users.json

# Ver códigos generados
cat ~/wifi-portal/data/codes.json
```

## 🔍 Troubleshooting

### El portal no abre automáticamente

Algunos dispositivos no redirigen automáticamente. Abre manualmente:
- http://192.168.4.1
- http://captive.apple.com (iOS)
- http://connectivitycheck.gstatic.com (Android)

### No hay internet después de conectar

Verifica que la interfaz WAN tenga conexión:
```bash
ping -I wlan1 8.8.8.8  # o eth0 según tu configuración
```

### El WiFi no aparece

Verifica hostapd:
```bash
sudo systemctl status hostapd
sudo journalctl -u hostapd
```

## 📁 Estructura del Proyecto

```
wifi-portal/
├── app/
│   ├── main.py              # Backend Flask
│   ├── static/
│   │   └── style.css        # Estilos
│   └── templates/
│       ├── index.html       # Página principal
│       ├── connected.html   # Sesión activa
│       ├── cooldown.html    # Período de espera
│       └── admin.html       # Panel admin
├── data/
│   ├── users.json          # Base de datos de usuarios
│   └── codes.json          # Base de datos de códigos
├── setup.sh                # Script de instalación
└── README.md              # Este archivo
```

## 🔒 Seguridad

- Los códigos son de un solo uso
- El panel admin requiere código secreto
- Los datos se almacenan localmente
- NAT configurado correctamente
- Firewall con iptables

## 📝 Términos y Condiciones

Los términos mostrados a los usuarios incluyen:
1. Posibles problemas de compatibilidad
2. Sin devoluciones
3. Derecho a vetar usuarios
4. Recolección de datos anónimos
5. Prohibición de actividades ilegales
6. Tiempo no pausable

## 🤝 Contribuciones

Este proyecto es open source. Siéntete libre de modificarlo según tus necesidades.

## 📜 Licencia

MIT License - Úsalo como quieras

## ✨ Créditos

Desarrollado para Raspberry Pi Zero 2W
Portal WiFi "Redmi Note 13"
EOF

echo "✅ ¡Proyecto creado exitosamente en ~/wifi-portal!"
echo ""
echo "📁 Contenido generado:"
echo "   - Aplicación Flask completa"
echo "   - Plantillas HTML en español"
echo "   - Estilos CSS modernos"
echo "   - Script de instalación automática"
echo "   - Documentación completa"
echo ""
echo "🚀 Para instalar en tu Raspberry Pi:"
echo "   cd ~/wifi-portal"
echo "   sudo bash setup.sh"
echo "   sudo reboot"
echo ""
echo "📱 Después podrás conectarte a 'Redmi Note 13' con contraseña 'pezpezpezpez'"
echo "🔧 Código admin: 'femboy'"
```

---

## 🎯 Cómo usar este comando

1. **Copia todo el bloque de código** desde `cd ~` hasta el final
2. **Pégalo en la terminal** de tu Raspberry Pi
3. **Presiona Enter** y espera unos segundos
4. **Ejecuta la instalación:**
   ```bash
   cd ~/wifi-portal
   sudo bash setup.sh
   sudo reboot
   ```

¡Y listo! El portal estará funcionando después del reinicio. 🎉

---

## 📝 Notas Importantes

- **Puerto 80 requiere sudo:** El servicio Flask se ejecuta como root para usar el puerto 80
- **Internet compartido:** La Pi debe tener internet vía Ethernet o segundo adaptador WiFi (wlan1)
- **Compatibilidad:** Probado en Raspberry Pi Zero 2W con Raspberry Pi OS Lite
- **Personalización:** Todos los tiempos, códigos y textos son modificables en el código

---

¿Quieres alguna modificación o característica adicional? 🚀
