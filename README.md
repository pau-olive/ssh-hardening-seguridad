# SSH Hardening: Medidas de Seguridad contra Ataques de Fuerza Bruta

![Portada del proyecto](cover.png)

## Descripción del Proyecto
Proyecto de **Administración de Sistemas y Ciberseguridad** enfocado en la auditoría y endurecimiento del protocolo SSH contra vulnerabilidades críticas.

El proyecto documenta el análisis completo de un servicio SSH sin proteger, la ejecución de un ataque de fuerza bruta exitoso y la implementación de múltiples capas de seguridad para mitigar riesgos.

---

## Características Principales
- **Auditoría de Vulnerabilidades**: Reconocimiento con nmap y análisis de servicios expuestos.
- **Ataque de Fuerza Bruta**: Implementación de hydra para compromiso exitoso de credenciales.
- **Hardening SSH**:
  - Cambio de puerto (22 → 2222)
  - Deshabilitar acceso root
  - Autenticación basada en clave pública
  - Desactivar autenticación por contraseña
- **Protección contra Brute Force**: Instalación y configuración de fail2ban (máx 3 intentos, 5 min ban).
- **Análisis Forense**: Revisión de logs de auditoría (/var/log/auth.log).
- **20+ capturas** del proceso completo: reconocimiento → ataque → hardening.
- Entorno **100% reproducible** en Linux.

---

## Tecnologías y Herramientas Utilizadas
- **Auditoría y Explotación**
  - nmap (reconocimiento de servicios)
  - Hydra (ataque de fuerza bruta)
  - SSH OpenSSH 9.6p1
- **Hardening**
  - sshd_config (configuración SSH)
  - fail2ban (limitación de intentos)
  - ssh-keygen (generación de claves)
- **Monitoreo**
  - /var/log/auth.log (logs de autenticación)
  - systemctl (control de servicios)
- **Entorno**
  - Ubuntu Server 22.04
  - Ubuntu Cliente
  - Kali Linux

---

## Objetivos Alcanzados
- Identificación de vulnerabilidades en SSH sin proteger (puerto predeterminado, sin límites de intentos).
- Recopilación de información con nmap: versión exacta del servidor, puerto abierto, MAC, latencia.
- Ataque de fuerza bruta exitoso: extracción de credenciales (usuario `gato`, contraseña `miau`).
- Acceso al servidor comprometido e inspección de logs de ataque.
- Implementación y verificación de todas las medidas de seguridad.
- Demostración de que el servidor hardened rechaza ataques idénticos.
- Análisis forense completo: auditoría de cambios, eventos de seguridad, logs de intrusión.

---

## Procesos Documentados

### PARTE 1: Análisis y Explotación de Vulnerabilidades

#### 1.1 Reconocimiento con nmap
```bash
nmap -sS -sV -p 22 [IP_SERVIDOR]
```
**Información extraída:**
- Estado del host: Up (activo)
- Puerto abierto: 22/tcp
- Servicio: SSH OpenSSH 9.6p1 Ubuntu 3ubuntu13.11
- Dirección MAC: 08:00:27:16:B9:1B
- Latencia: 0.00049s

#### 1.2 Creación de Usuario Vulnerable
```bash
sudo useradd -m -s /bin/bash gato
sudo passwd gato  # Contraseña: miau
```

#### 1.3 Diccionario de Contraseñas
```bash
cat > passwords_comunes.txt << 'EOF'
123456
password
12345678
qwerty
123456789
12345
1234567
111111
1234567890
123123
miau
EOF
```

#### 1.4 Ataque de Fuerza Bruta con Hydra
```bash
hydra -l gato -P passwords_comunes.txt ssh://[IP_SERVIDOR]:22 -v
```
**Resultado:** ✅ Acceso exitoso `gato:miau`

#### 1.5 Evidencias del Ataque en Logs
```bash
sudo tail -f /var/log/auth.log
```
**Hallazgos:**
- Múltiples intentos fallidos desde IP atacante
- Acceso exitoso registrado
- IP, puerto, usuario y timestamp de compromiso documentados

---

### PARTE 2: Implementación de Medidas de Seguridad

#### 2.1 Cambio de Puerto SSH
**Ubicación:** `/etc/ssh/sshd_config`

```bash
# ANTES
#Port 22

# DESPUÉS
Port 2222

sudo systemctl restart sshd
```
**Prueba:** Acceso desde puerto 22 → rechazado. Acceso desde puerto 2222 → exitoso.

#### 2.2 Deshabilitar Acceso Root
**En sshd_config:**

```
PermitRootLogin no
```

**Prueba:**
```bash
ssh -p 2222 root@[IP]
# Permission denied (public key)
```

#### 2.3 Instalación y Configuración de fail2ban

**Instalación:**
```bash
sudo apt-get install fail2ban
sudo systemctl enable fail2ban
```

**Configuración en `/etc/fail2ban/jail.local`:**
```ini
[DEFAULT]
bantime = 300          # 5 minutos
findtime = 600         # ventana 10 min
maxretry = 3

[sshd]
enabled = true
port = 2222
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 300
```

**Prueba:** 3 intentos fallidos → IP baneada → Conexión rechazada en intento 4.

#### 2.4 Autenticación basada en Clave Pública

**Generación de clave (cliente):**
```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa_gato -N ""
```

**Copiar clave al servidor:**
```bash
ssh-copy-id -i ~/.ssh/id_rsa_gato.pub -p 2222 gato@[IP]
```

**Ubicación en servidor:**
```
~/.ssh/authorized_keys
```

#### 2.5 Deshabilitar Autenticación por Contraseña

**En sshd_config:**
```
PasswordAuthentication no
PubkeyAuthentication yes
```

**Prueba:**
```bash
# Intento con contraseña → rechazado
ssh -p 2222 gato@[IP]
# Permission denied (publickey)

# Acceso con clave → exitoso
ssh -p 2222 -i ~/.ssh/id_rsa_gato gato@[IP]
```

---

## Análisis de Resultados

### Estado ANTES de Hardening
| Aspecto | Vulnerabilidad |
|---------|-----------------|
| **Puerto SSH** | Predeterminado (22) - evidente |
| **Intentos de login** | Sin límite |
| **Root login** | Permitido |
| **Contraseñas** | Aceptadas |
| **Protección brute-force** | Ninguna |
| **Resultado Hydra** | ✗ Acceso exitoso |

### Estado DESPUÉS de Hardening
| Aspecto | Medida Implementada |
|---------|---------------------|
| **Puerto SSH** | 2222 (oscuridad) |
| **Intentos de login** | Máx 3 antes de ban |
| **Root login** | Deshabilidado |
| **Contraseñas** | Deshabilitadas |
| **Protección brute-force** | fail2ban (5 min ban) |
| **Resultado Hydra** | ✓ IP baneada rápidamente |

---

## Prueba Final: "Algodón" (Reproducción del Ataque Post-Hardening)

**Intento 1: nmap**
```bash
nmap -sS -sV -p 2222 [IP]
# Detecta puerto pero acceso denegado
```

**Intento 2: Hydra**
```bash
hydra -l gato -P passwords_test.txt ssh://[IP]:2222
# Connection refused (fail2ban)
```

**Intento 3: SSH directo**
```bash
ssh -p 2222 gato@[IP]
# Permission denied (publickey)
```

**Intento 4: root**
```bash
ssh -p 2222 root@[IP]
# Permission denied (publickey)
```

**Conclusión:** Todos los vectores de ataque bloqueados ✓

---

## Capturas de Evidencia
Las siguientes capturas documentan el proceso completo:
- Reconocimiento con nmap
- Ejecución de hydra
- Acceso exitoso pre-hardening
- Modificación de sshd_config
- Estado de fail2ban
- Bloqueo de acceso post-hardening
- Análisis de logs (/var/log/auth.log)
- Generación de claves SSH
- Transferencia de claves públicas
- Acceso exitoso con autenticación por clave

*Ver carpeta `/screenshots` para todas las imágenes.*

---

## Recomendaciones de Seguridad Posteriores
1. Implementar certificados TLS/SSL para otras aplicaciones.
2. Configurar 2FA (autenticación de dos factores) adicional.
3. Auditoría continua de logs con herramientas centralizadas (ELK, Splunk).
4. Rotación periódica de claves SSH (cada 90 días).
5. Implementar IDS/IPS en la red perimetral.
6. Backup automático de configuraciones críticas.
7. Monitoreo de intentos de acceso sospechosos con alertas.

---

## Conclusión
Este proyecto demuestra de manera práctica cómo un servidor SSH vulnerable puede ser comprometido en segundos mediante técnicas estándar, y cómo múltiples capas de seguridad (defensa en profundidad) hacen que el mismo servidor sea prácticamente inmune a ataques similares.

**Postura de Seguridad:** CRÍTICA → MEDIA-ALTA

---

**📝 Nota Académica**

**Este proyecto es una actividad evaluada del ciclo ASIX en CEV (Centro de Estudios Vicens Vives).** Realizado como parte de la formación en administración de sistemas y ciberseguridad.

---

**Autor:** Pau Olivé Moreno  
**Institución:** CEV - ASIX  
**Fecha:** Marzo 2026  
**Versión:** 1.0
