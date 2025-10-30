# 📱 WiFi Portal "Redmi Note 13" para Raspberry Pi Zero 2W

Portal cautivo completo con sistema de tiempo limitado y cooldowns para Raspberry Pi.

## 🎯 Descripción

Este proyecto proporciona un comando único que genera automáticamente todo un portal cautivo WiFi para Raspberry Pi Zero 2W. Cuando un dispositivo se conecta al WiFi "Redmi Note 13", es redirigido automáticamente a un portal web donde debe aceptar términos y condiciones para obtener 1 hora de acceso a internet.

## ⚡ Uso Rápido

Todo el proyecto se genera con un único comando bash documentado en `command.md`.

1. **Lee el archivo `command.md`** - contiene el comando completo
2. **Copia el comando bash** completo 
3. **Pégalo en tu Raspberry Pi** y ejecútalo
4. **Ejecuta el script de instalación** generado

## 🚀 Características Principales

- ✅ **Portal cautivo automático**: Redirige todo el tráfico HTTP al portal
- ⏱️ **1 hora de acceso**: Por dispositivo identificado por MAC address
- ⏳ **Cooldown de 4 horas**: Entre sesiones para uso justo
- 🎁 **Sistema de códigos**: Para extender tiempo o reducir cooldown
- 🔧 **Panel administrativo**: Con código secreto "femboy"
- 📊 **Monitoreo en tiempo real**: De usuarios activos y códigos
- 🌐 **Interfaz en español**: Completamente localizada
- 🔒 **Seguridad**: Códigos de un solo uso, seguimiento por MAC

## 📋 Requisitos

- Raspberry Pi Zero 2W (u otros modelos con WiFi)
- Raspberry Pi OS Lite (Debian-based)
- Tarjeta SD con al menos 4GB
- Conexión a internet (vía Ethernet o segundo adaptador WiFi/wlan1)

## 🛠️ Tecnologías

- **Backend**: Python 3 + Flask
- **AP**: hostapd
- **DHCP/DNS**: dnsmasq
- **Firewall**: iptables
- **Autostart**: systemd
- **Storage**: JSON files (sin base de datos)

## 📁 Estructura Generada

El comando crea la siguiente estructura:

```
wifi-portal/
├── app/
│   ├── main.py              # Backend Flask
│   ├── static/
│   │   └── style.css        # Estilos CSS
│   └── templates/
│       ├── index.html       # Página de inicio
│       ├── connected.html   # Sesión activa
│       ├── cooldown.html    # Período de espera
│       └── admin.html       # Panel administrativo
├── data/
│   ├── users.json          # Base de datos de usuarios
│   └── codes.json          # Códigos generados
├── setup.sh                # Script de instalación
└── README.md              # Documentación
```

## 🔑 Credenciales

- **WiFi SSID**: Redmi Note 13
- **WiFi Password**: pezpezpezpez
- **Portal URL**: http://192.168.4.1
- **Admin Code**: femboy

## 🎫 Sistema de Códigos

El panel administrativo permite generar dos tipos de códigos:

1. **+1 Hora Extra**: Agrega 60 minutos adicionales a la sesión actual o futura
2. **Cooldown Reducido**: Reduce el tiempo de espera de 4h a 2h

Los códigos son:
- Alfanuméricos de 8 caracteres
- De un solo uso
- Rastreables (quién los usó y cuándo)

## 📊 Panel Administrativo

Para acceder:
1. Conéctate al WiFi "Redmi Note 13"
2. En el portal, ingresa el código: `femboy`
3. Serás redirigido al panel admin

Funciones del panel:
- Generar nuevos códigos
- Ver usuarios activos (MAC, IP, tiempo restante)
- Ver historial de códigos (usados/disponibles)

## 🔧 Personalización

Todas las configuraciones importantes están centralizadas en `main.py`:

```python
ADMIN_CODE = "femboy"           # Código del admin
SESSION_DURATION = 3600         # 1 hora en segundos
COOLDOWN_DURATION = 14400       # 4 horas en segundos
REDUCED_COOLDOWN = 7200         # 2 horas en segundos
```

Para cambiar el SSID y contraseña WiFi, edita el script `setup.sh` antes de ejecutarlo.

## 📝 Términos y Condiciones

Los usuarios deben aceptar términos que incluyen:
1. Posibles problemas de compatibilidad con IMT Lazarus
2. Sin devoluciones
3. Derecho a vetar usuarios sin previo aviso
4. Recolección de datos anónimos
5. Prohibición de actividades ilegales
6. Tiempo no pausable
7. Fecha de actualización: 29/10/2025

## 🤝 Contribuciones

Este es un proyecto open source. Siéntete libre de:
- Modificar el código según tus necesidades
- Agregar nuevas características
- Mejorar la interfaz
- Reportar bugs

## 📜 Licencia

MIT License - Úsalo libremente para tus proyectos.

## ⚠️ Notas Importantes

- El servicio Flask corre como root para usar el puerto 80
- La Raspberry Pi debe tener conexión a internet por otra interfaz (eth0 o wlan1)
- Los dispositivos son identificados por MAC address
- El tiempo de sesión no se puede pausar una vez iniciado
- Se recomienda reiniciar la Pi después de la instalación

## 🐛 Troubleshooting

Ver el archivo `command.md` para información detallada sobre resolución de problemas.

---

**Desarrollado para Raspberry Pi Zero 2W** | **Portal WiFi "Redmi Note 13"** ✨
