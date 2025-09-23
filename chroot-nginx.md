# 🧪 **LAB CHROOT – Nginx isolé sous Ubuntu Server 24.04**

## **1. Pré-requis**

* Ubuntu Server 24.04
* Accès root ou sudo
* Paquets nécessaires :

```bash
sudo apt update && sudo apt install nginx debootstrap coreutils -y
```

⚠️ **Arrêter Nginx sur l’hôte pour libérer le port 80 :**

```bash
sudo systemctl stop nginx
sudo systemctl disable nginx
```

---

## **2. Préparation du répertoire chroot**

```bash
sudo mkdir -p /var/chroot/nginx
```

---

## **3. Création d’un mini-système de base**

```bash
sudo debootstrap --variant=minbase noble /var/chroot/nginx http://archive.ubuntu.com/ubuntu/
```

---

## **4. Préparation du chroot**

Monter les répertoires nécessaires :

```bash
sudo mount --bind /dev /var/chroot/nginx/dev
sudo mount --bind /proc /var/chroot/nginx/proc
sudo mount --bind /sys /var/chroot/nginx/sys
```

Copier la résolution DNS :

```bash
sudo cp /etc/resolv.conf /var/chroot/nginx/etc/
```

---

## **5. Installation de Nginx dans le chroot**

Entrer dans le chroot :

```bash
sudo chroot /var/chroot/nginx /bin/bash
```

Installer Nginx :

```bash
apt update
apt install nginx -y
```

---

## **6. Configuration du site web**

Créer une page test :

```bash
echo "<h1>Bienvenue dans le CHROOT Nginx</h1>" > /var/www/html/index.html
```

Modifier la config pour écouter sur le **port 8080** (au lieu de 80) :
Fichier `/etc/nginx/sites-enabled/default` → remplacer :

```nginx
server {
    listen 8080;
    root /var/www/html;
    index index.html;
    server_name localhost;
}
```

Tester la configuration :

```bash
nginx -t
```

---

## **7. Démarrage du Nginx chrooté**

Toujours dans le chroot :

```bash
nginx
```

---

## **8. Test depuis l’hôte**

Sortir du chroot :

```bash
exit
```

Puis tester :

```bash
curl http://127.0.0.1:8080
```

Résultat attendu :

```
<h1>Bienvenue dans le CHROOT Nginx</h1>
```

---

## **9. Script de gestion (facultatif)**

Créez un script pour automatiser le lancement/arrêt :

`/usr/local/bin/nginx-chroot.sh`

```bash
#!/bin/bash
CHROOT_DIR="/var/chroot/nginx"

case "$1" in
  start)
    mount --bind /dev $CHROOT_DIR/dev
    mount --bind /proc $CHROOT_DIR/proc
    mount --bind /sys $CHROOT_DIR/sys
    chroot $CHROOT_DIR /usr/sbin/nginx
    ;;
  stop)
    chroot $CHROOT_DIR /usr/sbin/nginx -s stop
    umount $CHROOT_DIR/dev
    umount $CHROOT_DIR/proc
    umount $CHROOT_DIR/sys
    ;;
  restart)
    $0 stop
    sleep 2
    $0 start
    ;;
  *)
    echo "Usage: $0 {start|stop|restart}"
    ;;
esac
```

Rendre exécutable :

```bash
sudo chmod +x /usr/local/bin/nginx-chroot.sh
```

Utilisation :

```bash
sudo nginx-chroot.sh start
sudo nginx-chroot.sh stop
```


---

