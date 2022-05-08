# Clonezilla
Tout ce qui concerne ce superbe outil

# USB boot Clonezilla personnalisé
TODO : clarifier le code avec longues lignes

Créée une clé bootable avec la dernière version de CZ (en 32 ou 64 bits), avec clavier FR, rend le système accessible par SSH et par HOSTNAME (Avahi), installe des outils utiles tel vim.

Nota : prend en compte le certificat Let's encrypt expiré (sinon ne télécharge pas avec wget)

Astuce : on peut tester la clé sans rebooter avec QEmu/KVM (voir plus bas)
## Manuel (en attendant le test concluant de la version automatisée)
```sh
# Dépendances pour mettre en place (Debian) en SU
user@host:~$ su -
user@host:~$ apt update && apt install -y rsync libc6-i386 mtools squashfs-tools parted wget dosfstools

# Définir la clé (utiliser aussi `dmesg -w` avant de brancher)
lsblk
DEV=/dev/sdg

# Config
PASS='P455w0rd!*'
HOSTNAME=CZ-LIVE
SCRIPT_PATH=/set_live_session.sh
SERVICE_PATH=/etc/systemd/system/set_live_session.service
ISO_ARCH=64
#ISO_ARCH="(i386|i486|i686)"
ISO_DL_SOURCE=https://sourceforge.net/projects/clonezilla/files/clonezilla_live_stable

# Format
root@host:~# parted -a cylinder $DEV -s "mklabel msdos" -s "mkpart primary fat32 0 100%"
root@host:~# mkfs.vfat -n $HOSTNAME -I ${DEV}1 -F 32

# DL dernier ISO stable réel (Nécessite mtools sur les nouvelles versions)
root@host:~# cz_latest_dl_folder=$( wget --no-check-certificate -qO- $ISO_DL_SOURCE | grep -Eo "https?://[^ \"]*sourceforge[^ \"]*/[-0-9.]+/" | sort -rV | head -n1 )
root@host:~# cz_latest_dl_version=$( echo $cz_latest_dl_folder | grep -Eo "[-0-9.]{2,}" )
root@host:~# cz_latest_iso=$( wget --no-check-certificate -qO- $cz_latest_dl_folder | grep -iEo "https?://[^ \"]*sourceforge[^ \"]*\.iso" | grep -E $cz_latest_dl_version | grep -E "$ISO_ARCH[^-]" | sort -u )
root@host:~# wget --no-check-certificate $cz_latest_iso -O cz_latest.iso

# Appliquer ISO sur USB
root@host:~# mkdir -p /mnt/CLONEZILLA/ /mnt/ISO
root@host:~# mount -o loop cz_latest.iso /mnt/ISO/
root@host:~# mount ${DEV}1 /mnt/CLONEZILLA/
root@host:~# rsync -aP /mnt/ISO/ /mnt/CLONEZILLA/
root@host:~# umount -l /mnt/ISO
root@host:~# rm cz_latest.iso
root@host:~# rmdir /mnt/ISO

# Au boot on definit en FR et on prerun dhclient
root@host:~# sed -i '/^ *append initrd=/  s/locales=/locales=fr_FR.UTF-8/;s/keyboard-layouts=/keyboard-layouts=fr/;s/enforcing=0 /enforcing=0  ocs_prerun="dhclient" /' /mnt/CLONEZILLA/syslinux/syslinux.cfg
root@host:~# mount -t squashfs -o loop /mnt/CLONEZILLA/live/filesystem.squashfs /mnt/
root@host:~# mkdir squashfs
root@host:~# rsync -aP /mnt/ squashfs

# chroot squashfs pour personnaliser
root@host:~# for f in /proc /sys ; do mount -B $f squashfs$f ; done ; mount -t devpts none squashfs/dev/pts
root@host:~# PASS=$PASS HOSTNAME=$HOSTNAME SCRIPT_PATH=$SCRIPT_PATH SERVICE_PATH=$SERVICE_PATH chroot squashfs
root@host:/# echo $HOSTNAME > /etc/hostname
root@host:/# cat << EOF > $SCRIPT_PATH
#!/bin/bash
ln -s /usr/lib/live/mount/medium /root/usb_root
ln -s /usr/lib/live/mount/medium /home/user/usb_root
echo -e '$PASS\n$PASS' | passwd user
echo -e '$PASS\n$PASS' | passwd root
EOF
root@host:/# chmod +x $SCRIPT_PATH
root@host:/# a=$( cat /etc/ssh/sshd_config | grep -vE ^[[:space:]]*PermitRootLogin ; echo PermitRootLogin yes )
root@host:/# echo -e "$a" > /etc/ssh/sshd_config
root@host:/# unset a
# pour le moment je n'arrive pas à faire démarrer correctement SSH que depuis profile (s'exécute à chaque logon : ex : user -> root)
root@host:/# echo "systemctl restart ssh" >> /etc/profile
# utiliser les services (https://unix.stackexchange.com/questions/529111/execute-a-script-before-login-screen/529183#529183)
root@host:/# cat << EOF > $SERVICE_PATH
[Unit]
Description=Configure la live session
[Service]
Type=Simple
ExecStart=/bin/bash $SCRIPT_PATH
[Install]
WantedBy=multi-user.target
EOF
root@host:/# chmod 644 $SERVICE_PATH
root@host:/# systemctl enable ssh
root@host:/# systemctl enable $( basename $SERVICE_PATH .service )
root@host:/# echo nameserver 8.8.8.8 > /etc/resolv.conf
# Les paquets qu'on doit installer...
root@host:/# #apt update && apt install -y vim avahi-daemon dnsutils winbind libnss-winbind libnss-mdns#
root@host:/# apt update && apt install -y vim avahi-daemon dnsutils winbind libnss-winbind libnss-mdns memtester
# ...
# Traitement des actions différées (« triggers ») pour mailcap (3.68) ...
# localepurge: Disk space freed:      0 KiB in /usr/share/locale
# localepurge: Disk space freed:     16 KiB in /usr/share/man
# localepurge: Disk space freed:      0 KiB in /usr/share/tcltk
# localepurge: Disk space freed:      0 KiB in /usr/share/calendar
# localepurge: Disk space freed:   2164 KiB in /usr/share/vim/vim82/lang
# ...
# Total disk space freed by localepurge: 2180 KiB
# => ces lignes surnuméraires sont à priori normales...
# nécessaire ? a priori non ! : root@host:/# update-initramfs -k all -u
root@host:/# exit
root@host:~# for f in /proc /sys /dev/pts ; do umount -lf squashfs$f ; done

# Appliquer la personnalisation
root@host:~# mksquashfs squashfs filesystem.squashfs -info
root@host:~# umount -l /mnt/
root@host:~# chroot squashfs dpkg-query -W --showformat='${Package}  ${Version}\n' > /mnt/CLONEZILLA/filesystem.manifest
root@host:~# rm -rf squashfs
root@host:~# rsync -P filesystem.squashfs /mnt/CLONEZILLA/live
root@host:~# rm filesystem.squashfs
root@host:~# chmod 444 /mnt/CLONEZILLA/live/filesystem.squashfs

# Rendre la clé USB bootable
root@host:~# bash /mnt/CLONEZILLA/utils/linux/makeboot.sh -L $HOSTNAME -b ${DEV}1
root@host:~# umount -l /mnt/CLONEZILLA
root@host:~# rmdir /mnt/CLONEZILLA
```

## Automatisé
TODO : à tester (en particulier le chroot avec l'échappement des $)
TODO : tester avec nouvelle gestion des applications
```sh
# Dépendances pour mettre en place (Debian) en SU
apt update && apt install -y rsync libc6-i386 mtools squashfs-tools parted wget dosfstools

# Définir la clé (utiliser aussi `dmesg -w` avant de brancher)
lsblk
DEV=/dev/sdg

# Config
PASS='P455w0rd!*'
HOSTNAME=CZ-LIVE
SCRIPT_PATH=/set_live_session.sh
SERVICE_PATH=/etc/systemd/system/set_live_session.service
ISO_ARCH=64
#ISO_ARCH="(i386|i486|i686)"
ISO_DL_SOURCE=https://sourceforge.net/projects/clonezilla/files/clonezilla_live_stable
APP="vim avahi-daemon dnsutils winbind libnss-winbind libnss-mdns memtester nbd-server"

# Format
parted -a cylinder $DEV -s "mklabel msdos" -s "mkpart primary fat32 0 100%"
mkfs.vfat -n $HOSTNAME -I ${DEV}1 -F 32

# DL dernier ISO stable réel (Nécessite mtools sur les nouvelles versions)
cz_latest_dl_folder=$( wget --no-check-certificate -qO- $ISO_DL_SOURCE | grep -Eo "https?://[^ \"]*sourceforge[^ \"]*/[-0-9.]+/" | sort -rV | head -n1 )
cz_latest_dl_version=$( echo $cz_latest_dl_folder | grep -Eo "[-0-9.]{2,}" )
cz_latest_iso=$( wget --no-check-certificate -qO- $cz_latest_dl_folder | grep -iEo "https?://[^ \"]*sourceforge[^ \"]*\.iso" | grep -E $cz_latest_dl_version | grep -E "$ISO_ARCH[^-]" | sort -u )
wget --no-check-certificate $cz_latest_iso -O cz_latest.iso

# Appliquer ISO sur USB
mkdir -p /mnt/CLONEZILLA/ /mnt/ISO
mount -o loop cz_latest.iso /mnt/ISO/
mount ${DEV}1 /mnt/CLONEZILLA/
rsync -aP /mnt/ISO/ /mnt/CLONEZILLA/
umount -l /mnt/ISO
rm cz_latest.iso
rmdir /mnt/ISO

# Au boot on definit en FR et on prerun dhclient
sed -i '/^ *append initrd=/  s/locales=/locales=fr_FR.UTF-8/;s/keyboard-layouts=/keyboard-layouts=fr/;s/enforcing=0 /enforcing=0  ocs_prerun="dhclient" /' /mnt/CLONEZILLA/syslinux/syslinux.cfg
mount -t squashfs -o loop /mnt/CLONEZILLA/live/filesystem.squashfs /mnt/
mkdir squashfs
rsync -aP /mnt/ squashfs

# chroot squashfs pour personnaliser
for f in /proc /sys ; do mount -B $f squashfs$f ; done ; mount -t devpts none squashfs/dev/pts
#cat << EOC| PASS=$PASS HOSTNAME=$HOSTNAME SCRIPT_PATH=$SCRIPT_PATH SERVICE_PATH=$SERVICE_PATH chroot squashfs
cat << EOC| APP=$APP PASS=$PASS HOSTNAME=$HOSTNAME SCRIPT_PATH=$SCRIPT_PATH SERVICE_PATH=$SERVICE_PATH chroot squashfs
echo \$HOSTNAME > /etc/hostname
cat << EOF > \$SCRIPT_PATH
#!/bin/bash
ln -s /usr/lib/live/mount/medium /root/usb_root
ln -s /usr/lib/live/mount/medium /home/user/usb_root
echo -e '\$PASS\n\$PASS' | passwd user
echo -e '\$PASS\n\$PASS' | passwd root
EOF
chmod +x \$SCRIPT_PATH
a=\$( cat /etc/ssh/sshd_config | grep -vE ^[[:space:]]*PermitRootLogin ; echo PermitRootLogin yes )
echo -e "\$a" > /etc/ssh/sshd_config
unset a
# pour le moment je n'arrive pas à faire démarrer correctement SSH que depuis profile (s'exécute à chaque logon : ex : user -> root)
echo "systemctl restart ssh" >> /etc/profile
# utiliser les services (https://unix.stackexchange.com/questions/529111/execute-a-script-before-login-screen/529183#529183)
cat << EOF > \$SERVICE_PATH
[Unit]
Description=Configure la live session
[Service]
Type=Simple
ExecStart=/bin/bash \$SCRIPT_PATH
[Install]
WantedBy=multi-user.target
EOF
chmod 644 \$SERVICE_PATH
systemctl enable ssh
systemctl enable \$( basename \$SERVICE_PATH .service )
echo nameserver 8.8.8.8 > /etc/resolv.conf
# Les paquets qu'on doit installer...
# #apt# update && apt install -y vim avahi-daemon dnsutils ...
apt update && apt install -y \$APP
# ...
# Traitement des actions différées (« triggers ») pour mailcap (3.68) ...
# localepurge: Disk space freed:      0 KiB in /usr/share/locale
# localepurge: Disk space freed:     16 KiB in /usr/share/man
# localepurge: Disk space freed:      0 KiB in /usr/share/tcltk
# localepurge: Disk space freed:      0 KiB in /usr/share/calendar
# localepurge: Disk space freed:   2164 KiB in /usr/share/vim/vim82/lang
# ...
# Total disk space freed by localepurge: 2180 KiB
# => ces lignes surnuméraires sont à priori normales...
# nécessaire ? a priori non ! : root@host:/# update-initramfs -k all -u
EOC
for f in /proc /sys /dev/pts ; do umount -lf squashfs$f ; done

# Appliquer la personnalisation
mksquashfs squashfs filesystem.squashfs -info
umount -l /mnt/
chroot squashfs dpkg-query -W --showformat='${Package}  ${Version}\n' > /mnt/CLONEZILLA/filesystem.manifest
rm -rf squashfs
rsync -P filesystem.squashfs /mnt/CLONEZILLA/live
rm filesystem.squashfs
chmod 444 /mnt/CLONEZILLA/live/filesystem.squashfs

# Rendre la clé bootable
bash /mnt/CLONEZILLA/utils/linux/makeboot.sh -L $HOSTNAME -b ${DEV}1
umount -l /mnt/CLONEZILLA
rmdir /mnt/CLONEZILLA
```
## Automatisé avec export NBD
Objectif : booter une clé USB clonezilla sur un système distant et automatiquement exporter ses disques locaux avec un serveur NBD pour pouvoir les exploiter à distance depuis un client NBD. Voir mon travail sur NBD [ici](https://github.com/lenainjaune/nbd_remote_disk/blob/main/README.md) et voir un exemple complet de mise en place d'une installation client/server NBD [là](https://github.com/lenainjaune/nbd_remote_disk/blob/main/README.md#exemple-complet-clientserveur-).

Comment ça marche :
* Formater support
* DL dernier ISO clonezilla stable réel
* Copie du contenu ISO sur support et modifier la configuration du boot (DHCP et AZERTY)
* Extraction du système interne
* Modification du système interne pour ajouter des outils, définir des réglages et permettre d'exécuter au boot un script externe
* Réintégration du système interne
* Ajout du script externe qui permettra d'outrepasser certains réglages et exécutera le script d'export NBD
* Ajout du script d'export de disque NBD
* Rendre support bootable

TODO : NBD permettre d'exporter les disques utilisés (montés ou autres) ou PAS (voir test attacher support USB distant - plus bas)

TODO : gestion PASSWD comme dans le script utilisateur

TODO : /etc/hosts est-ce ça qui donnait les erreur résolution DNS CZ-LIVE ?


TODO : démarrer SSH ailleurs que /etc/profile exécuté à chaque logon

TODO : empêcher de supprimer le script utilisateur avec chattr

TODO : si script utilisateur absent en recreer un par defaut ; si existe deja verifier que shebang en ligne 1 

TODO : parfois on doit redémarrer avahi (hostname-2 !)

TODO : partition persistante avec scripts utilisateurs pour pouvoir modifier en live session

```sh
# Définir la clé (utiliser aussi `dmesg -w` avant de brancher)
lsblk
DEV=/dev/sdb


# Dépendances pour mettre en place (Debian) en SU
apt update && apt install -y rsync libc6-i386 mtools squashfs-tools parted wget dosfstools

# Config
DNS=8.8.8.8
HOSTNAME_USB=CZ-LIVE
## 'PASS' protège PASS d'une expansion non voulue de * ou ! etc.
PASS='P455w0rd!*'
OUTER_1ST_SCRIPT=ran_first.sh
SCRIPT_NAME_NBD_EXPORT=export_all_local_disk.sh
ISO_ARCH=64
#ISO_ARCH="(i386|i486|i686)"


# Format
parted -a cylinder $DEV -s "mklabel msdos" \
 -s "mkpart primary fat32 0 100%"
mkfs.vfat -n $HOSTNAME_USB -I ${DEV}1 -F 32


# Dernier ISO stable réel (Nécessite mtools pour nouvelles versions)
ISO_DL_SOURCE=\
https://sourceforge.net/projects/clonezilla/files/clonezilla_live_stable
cz_latest_dl_folder=$(
 wget --no-check-certificate -qO- $ISO_DL_SOURCE \
 | grep -Eo "https?://[^ \"]*sourceforge[^ \"]*/[-0-9.]+/" \
 | sort -rV | head -n1
)
cz_latest_dl_version=$(
 echo $cz_latest_dl_folder | grep -Eo "[-0-9.]{2,}" 
)
cz_latest_iso=$(
 wget --no-check-certificate -qO- $cz_latest_dl_folder \
 | grep -iEo "https?://[^ \"]*sourceforge[^ \"]*\.iso" \
 | grep -E $cz_latest_dl_version | grep -E "$ISO_ARCH[^-]" | sort -u
)
wget --no-check-certificate $cz_latest_iso -O cz_latest.iso


# Appliquer ISO sur USB
mkdir -p /mnt/CLONEZILLA/ /mnt/ISO
mount -o loop cz_latest.iso /mnt/ISO/
mount ${DEV}1 /mnt/CLONEZILLA/
rsync -aP /mnt/ISO/ /mnt/CLONEZILLA/
umount -l /mnt/ISO
rm cz_latest.iso
rmdir /mnt/ISO


# Au boot on definit en FR et on prerun dhclient
sed -i -e '/^ *append initrd=/s/locales=/locales=fr_FR.UTF-8/' \
 -e 's/keyboard-layouts=/keyboard-layouts=fr/' \
 -e 's/enforcing=0 /enforcing=0  ocs_prerun="dhclient" /' \
 /mnt/CLONEZILLA/syslinux/syslinux.cfg
mount -t squashfs -o loop /mnt/CLONEZILLA/live/filesystem.squashfs /mnt
mkdir squashfs
rsync -aP /mnt/ squashfs


# chroot squashfs pour personnaliser
for f in /proc /sys ; do mount -B $f squashfs$f ; done
mount -t devpts none squashfs/dev/pts
cat << EOC| DNS=$DNS PASS=$PASS HOSTNAME_USB=$HOSTNAME_USB \
  OUTER_1ST_SCRIPT=$OUTER_1ST_SCRIPT chroot squashfs

 INNER_1ST_SCRIPT=inner_1st_script.sh
 INNER_1ST_SERVICE=run_1st_script.service
 APP="vim avahi-daemon dnsutils winbind libnss-winbind libnss-mdns \
  memtester nbd-server"

 echo \$HOSTNAME_USB > /etc/hostname

 # DNS (DHCP au boot dans ocs_prerun du boot)
 a=\$(
  echo nameserver \$DNS
  cat /etc/resolv.conf \\
   | grep -vE ^[[:space:]]*nameserver[[:space:]]+\$DNS[[:space:]]*$
 )
 echo -e "\$a" > /etc/resolv.conf

 # Installer les paquets (nécessite réseau configuré + DNS) ...
 apt update && apt install -y \$APP
 #  ...
 #  Traitement des actions différées (« triggers ») pour mailcap (3.68)
 #  localepurge: Disk space freed:      0 KiB in /usr/share/locale
 #  ... 16 KiB in /usr/share/man
 #  ... 0 KiB in /usr/share/tcltk
 #  ... 0 KiB in /usr/share/calendar
 #  ... 2164 KiB in /usr/share/vim/vim82/lang
 #   ...
 # Total disk space freed by localepurge: 2180 KiB
 #  => ces lignes surnuméraires sont à priori normales...

 # SSH (auto exécution + accès root)
 a=\$(
  cat /etc/profile \\
   | grep -vE ^[[:space:]]*systemctl[[:space:]]+restart[[:space:]]+ssh
  echo systemctl restart ssh
 )
 echo -e "\$a" > /etc/profile
 a=\$(
  cat /etc/ssh/sshd_config | grep -vE ^[[:space:]]*PermitRootLogin
  echo PermitRootLogin yes
 )
 echo -e "\$a" > /etc/ssh/sshd_config
 systemctl enable ssh

 # echo -e "\$(
 #  a=\$(
 #   cat /etc/ssh/sshd_config | grep -vE ^[[:space:]]*PermitRootLogin
 #   echo PermitRootLogin yes
 #  )
 #  echo -e "\$a"
 # )" > /etc/ssh/sshd_config 

cat << EOF > \$INNER_1ST_SCRIPT
#!/bin/bash

 echo -e "127.0.0.1\tlocalhost\n127.0.1.1\t$HOSTNAME_USB" > /etc/hosts
 # => OK

 echo -e '\$PASS\n\$PASS' | passwd user
 echo -e '\$PASS\n\$PASS' | passwd root

 ln -s /usr/lib/live/mount/medium /root/usb_root
 ln -s /usr/lib/live/mount/medium /home/user/usb_root

# /usr/bin/sudo /run/live/medium/\$OUTER_1ST_SCRIPT
 /run/live/medium/\$OUTER_1ST_SCRIPT

EOF

 chmod +x \$INNER_1ST_SCRIPT

 # Configurer service qui démarre le premier script (voir [1])

cat << EOF > /etc/systemd/system/\$INNER_1ST_SERVICE

 [Unit]
 Description=Exécute le premier script interne

 [Service]
 Type=Simple
 ExecStart=/bin/bash /\$INNER_1ST_SCRIPT
 #ExecStart=/usr/bin/sudo /bin/bash /\$INNER_1ST_SCRIPT

 [Install]
 WantedBy=multi-user.target
 
EOF

 chmod 644 /etc/systemd/system/\$INNER_1ST_SERVICE
 systemctl enable \$INNER_1ST_SERVICE

 # nécessaire ? a priori non ! : root@host:/# update-initramfs -k all -u
EOC

for f in /proc /sys /dev/pts ; do umount -lf squashfs$f ; done



# Appliquer la personnalisation
mksquashfs squashfs filesystem.squashfs -info
umount -l /mnt/
chroot squashfs dpkg-query -W --showformat='${Package}  ${Version}\n' \
 > /mnt/CLONEZILLA/filesystem.manifest
rm -rf squashfs
rsync -P filesystem.squashfs /mnt/CLONEZILLA/live \
 && rm filesystem.squashfs
chmod 444 /mnt/CLONEZILLA/live/filesystem.squashfs

# Script à exécuter en 1er ; peut être modifié hors LIVE session
cat << EOF > /mnt/CLONEZILLA/$OUTER_1ST_SCRIPT
#!/bin/bash

 # Attention : ce script peut être modifié que hors LIVE session !

 # c'est bien root qui exécute

 # touch /touched
 # => OK

 # Rédéfinir

 # Host Name pour multiple Clonezilla distants (defaut : $HOSTNAME_USB)
 # Laisser vide pour ne pas redéfinir
 NEW_HOSTNAME_USB=

 # Mot de passe de session (defaut : '$PASS')
 # Laisser '' pour ne pas redéfinir
 # Nota : 'PASS' protège PASS d'une expansion non voulue de * ou ! etc.
 NEW_PASS=''

 if [[ ! -z \$NEW_HOSTNAME_USB ]] ; then
  sed -i "s/$HOSTNAME_USB/\$NEW_HOSTNAME_USB/g" /etc/hosts
  echo \$NEW_HOSTNAME_USB > /etc/hostname
  hostnamectl set-hostname \$NEW_HOSTNAME_USB
  systemctl restart avahi-daemon
 fi

 if [[ ! \$NEW_PASS == '' ]] ; then
  # echo \$NEW_PASS > /touched
  echo -e "\$NEW_PASS\n\$NEW_PASS" | passwd user
  echo -e "\$NEW_PASS\n\$NEW_PASS" | passwd root
  # echo retour passwd : $?
 fi

 /usr/lib/live/mount/medium/$SCRIPT_NAME_NBD_EXPORT

EOF

chmod +x /mnt/CLONEZILLA/$OUTER_1ST_SCRIPT



# Exporter les disques locaux
# Nota : heredoc avec variables non expandées
cat << 'EOF' > /mnt/CLONEZILLA/$SCRIPT_NAME_NBD_EXPORT
#!/bin/bash

 # c'est bien root qui exécute

 # S'il y a encore des connexions établies (disques à distance importés)
 if [[ $( netstat -atnp | grep nbd-server | grep ESTABLISHED ) ]] ; then
  msg="Erreur : les disques sont déjà exportés"
  msg="$msg\nVoici la liste :"
  msg="$msg\n$( netstat -atnp | grep nbd-server | grep ESTABLISHED )"
  echo -e "$msg"
  exit 1
 fi

 modprobe -r nbd
 pkill nbd-server

 # Config
 (
  echo -e "[generic]\nuser=root\ngroup=root\nallowlist=true"
  for e in $( \
    lsblk -nd -o NAME,SIZE | awk '$2 != "0B" { print $1 "|" $2 }' \
  ) ; do
   dev=$( echo $e | cut -d '|' -f 1 )
   id=$( \
    find /dev -type l \
         -exec bash -c 'l={} ; echo $l $( readlink $l )' \; \
          | grep /by-id/.*$dev$ | cut -d ' ' -f 1 \
          | rev | cut -d '/' -f 1 | rev )
   size=$( echo $e | cut -d '|' -f 2 )
   echo -e "[$dev@$id@$size]\nexportname=/dev/$dev"
  done
 ) > nbd.conf
 
 modprobe nbd

 # Je ne sais pas pourquoi mais il faut le démarrer avec -d (daemon ?)
 nbd-server -C /nbd.conf -d

EOF

chmod +x /mnt/CLONEZILLA/$SCRIPT_NAME_NBD_EXPORT



# Rendre la clé bootable
bash /mnt/CLONEZILLA/utils/linux/makeboot.sh -L $HOSTNAME_USB -b ${DEV}1
umount -l /mnt/CLONEZILLA
rmdir /mnt/CLONEZILLA
```
[1] utiliser les services (https://unix.stackexchange.com/questions/529111/execute-a-script-before-login-screen/529183#529183)

## Tester avec QEmu/KVM
```sh
DEV=/dev/sdg

root@host:~# kvm -m 4096 -hda $DEV
# WARNING: Image format was not specified for '/dev/sdg' and probing guessed raw.
#         Automatically detecting the format is dangerous for raw images, write operations on block 0 will be restricted.
#         Specify the 'raw' format explicitly to remove the restrictions.
# => la fenêtre d'accueil devrait s'afficher
```
