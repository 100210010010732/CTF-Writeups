## Machine: Confuser

**Platform:** echoCTF 
**Difficulty:** Advanced
**OS:** Linux 
**Points:** 5200
**Date:** 15/04/2026

---

## Summary

Durante la enumeración del panel, se identificó el uso del gestor de archivos **Responsive FileManager**. Mediante investigación en fuentes públicas (Exploit-DB), se determinó que esta versión es vulnerable a **Arbitrary File Creation (CVE-2022-46604)**. En lugar de una explotación manual, se utilizó un exploit público en Python que automatiza la creación de la sesión y el bypass del filtro, enviando un archivo con extensión `.php` en el parámetro `path` que el servidor procesa sin validar.

---

## Enumeration

### 🔸 Nmap

```bash
nmap -sC -sV --top-ports 1000 <IP>
```

**Open Ports:**

* 22 → SSH
* 80 → HTTP

**Key Findings:**

* Subida de archivos php en general bloqueados.
* Vector de ataque por medio de script de python que aprovecha una validación de la subida de archivos.

---

### 🔸 Web / Service Enumeration

**Technologies:**

* PHP / Apache 2.41.6 

**Directories / Endpoints:**

* `/plugins/source
* `/plugins/filemanager

**Observations:**

* Archivos subidos se guardan en /plugins/source por defecto
* /plugins/filemanager/dialog.php interfaz de subida de archivos
* /plugins/filemanager/upload.php procesa los archivos subidos (POST)

---

## Initial Foothold

### 🔸 Vulnerability

* **Type:** Arbitrary File Creation
* **Location:**`/plugins/filemanager/upload.php` (parámetros POST `name` y `path`)
* **Impact:** Permite bypassear la subida de php por medio de parámetros POST obteniendo acceso al servidor.

---

### 🔸 Exploitation

```bash
python script.py ip 
```

**Explanation:**

- El fallo reside en una validación inconsistente entre los parámetros name y path en el endpoint upload.php.
* La única validación está en el name y el path no cuenta con alguna, el servidor confía en su única validación y usa **path** para asignar el nombre y extensión a usar.

**PoC**
1. El script primero busca a dialog.php para obtener una sesión válida (cookie).
2. Envía un POST a upload.php con lo siguientes parámetros:
	- "path": "shell.php" "path_thumb" 
	- "name": "shell.txt" 
	- "new_content": payload
3. Al procesar la petición el servidor solo valida **name** creyendo que es un archivo de texto, pero termina asignando el nombre y extensión del **path**.

---

## Shell Access

* **Method:** (reverse shell)
* **Stabilization:**

```bash
script /dev/null -qc bash
	reset xterm
export TERM=xterm
```

---

## Post-Exploitation

### 🔸 Internal Enumeration

* **Users:** root, ETSCTF
* **Interesting Files:** /usr/local/bin/confuser
* **Permissions:** -rwxr-xr-r

---

## Privilege Escalation

### 🔸 Vector (PATH hijacking)
	
* Un binario compilado que ejecuta /bin/ls -alR && getcaps
* getcaps no se llama por su ruta absoluta lo que permite cambiar 
* `strings /usr/local/bin/confuser` para leer el binario compilado, aquí se encuentra la cadena `getcaps` sin su ruta absoluta 

---

### 🔸 Exploitation

```bash
cd /tmp
echo '/bin/bash -p' > getcaps
chmod +x getcaps
sudo PATH=/tmp:$PATH /usr/local/bin/confuser
sudo /usr/local/bin/confuser
```

---

## Lessons Learned

* **Qué aprendí**: Que la fase de reconocimiento no solo sirve para encontrar puertos, sino para identificar versiones exactas de software (Fingerprinting) que ya cuentan con exploits públicos, ahorrando tiempo de explotación manual.
* **Qué debería haber detectado antes**: Acostumbrarme a interceptar el tráfico con Burp Suite de fondo, incluso cuando uso exploits automáticos, para entender exactamente qué está enviando el script a nivel de peticiones HTTP.

---

## Attack Pattern 

* **Type:** Arbitrary File Upload → RCE → PrivEsc
* **Where it appears:** En gestores de archivos web desactualizados (como FileManager) 
* **How to detect it:** Realizando _Fingerprinting_ del software utilizado en el backend y cruzando esa información con bases de datos de vulnerabilidades conocidas (CVEs) antes de intentar ataques de fuerza bruta.
* **Key indicators:** Binarios locales que llaman a herramientas del sistema sin usar la ruta absoluta

---

## References

* https://www.exploit-db.com/exploits/51251


