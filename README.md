# Documentation

The purpose of this document?
Enable SNMP support on IMUNES nodes / docker image.

What is IMUNES?

What is OpenNMS?

## Περιβάλλον Εγκατάστασης

Το λειτουργικό σύστημα που χρησιμοποιήθηκε είναι το *Xubuntu 20.04.1 LTS (Focal Fossa) 64-bit* σε εικονικό μηχάνημα (virtual machine) μέσω *VirtualBox*.

## Εγκατάσταση IMUNES

Αρχικά, ενημερώνουμε και εγκαθιστούμε τυχόν αναβαθμίσεις στο σύστημα.

```console
user@msnlab:~$ sudo apt update && sudo apt dist-upgrade -y
```

Εγκαθιστούμε τα απαραίτητα πακέτα για την εγκατάσταση IMUNES.

```console
user@msnlab:~$ sudo apt install git openvswitch-switch docker.io xterm wireshark make imagemagick tk tcllib util-linux
```

Κατεβάζουμε ένα αντίγραφο του πηγαίου κώδικα του IMUNES από το GitHub.

```console
user@msnlab:~$ git clone https://github.com/imunes/imunes.git
```

Μεταβαίνουμε στον φάκελο ```imunes```
```console
user@msnlab:~$ cd imunes
```

και εκτελούμε ```sudo make install``` για να κάνουμε εγκατάσταση του IMUNES στο σύστημά μας.
```console
user@msnlab:~$ sudo make install
```

Έπειτα αρχικοποιούμε το IMUNES με την παρακάτω εντολή. Η παράμετρος ```-p``` θα προετοιμάσει το *virtual root file system* και θα κατεβάσει το ```imunes/template``` docker image τοπικά, ένα image το οποίο είναι αναγκαίο και εκτελείται σε κάθε κόμβο πείραματικού δικτύου μέσα στο IMUNES.
```console
user@msnlab:~$ sudo imunes -p
```

Πλέον, μπορούμε να τρέξουμε το IMUNES
```console
user@msnlab:~$ sudo imunes
```

## Εγκατάσταση Quagga με υποστήριξη πρωτοκόλλου SNMP

Στο σημείο αυτό έχουμε ολοκληρώσει την εγκατάσταση του IMUNES και το επόμενο βήμα είναι να ρυθμίσουμε το Quagga έτσι ώστε υποστηρίζει το πρωτόκολλο SNMP. Για να το πετύχουμε αυτό θα πρέπει:
- να επεξεργαστούμε το ```imunes/template``` docker image με σκοπό
- να εγκαταστήσουμε τα απαραίτητα πακέτα λογισμικού
- να εγκαταστήσουμε εκ νέου το Quagga με υποστήριξη SNMP από τον πηγαίο κώδικα
- να ρυθμίσουμε σωστά την υπηρεσία SNMP καθώς και να ρυθμίσουμε τις παγίδες SNMP (SNMP Traps) καθώς και
- να αποθηκεύσουμε τις αλλαγές στο ```imunes/template``` docker image ώστε να μπορούμε να χρησιμοποιούμε πλέον το Quagga με SNMP υποστήριξη σε κάθε κόμβο δικτύου μέσα από το IMUNES.


### Επιβεβαιώνουμε πως διαθέτουμε τοπικά το ```imunes/template``` docker image
```console
user@msnlab:~$ sudo docker image ls
REPOSITORY      TAG     IMAGE ID        CREATED SIZE
imunes/template latest  28ab347bef71    934MB
```

### Εκτελούμε το image αυτό ως ένα container
```console
user@msnlab:~$ sudo docker run --detach --tty --net='host' imunes/template
```
Το container αυτό θα έχει πρόσβαση στο ίδιο δίκτυο με τον host, σε αυτή τη περίπτωση θέλουμε στο container να έχουμε πρόσβαση στο διαδίκτυο και αυτό το επιτυχγάνουμε με τη παράμετρο ```--net='host'```.

Επιβεβαιώνουμε πως το ```container``` δημιουργήθηκε επιτυχώς
```console
user@msnlab:~$ sudo docker container ls
```

Από το αποτέλεσμα της παραπάνω εντολής θα πρέπει να λάβουμε πληροφορίες σχετικά με το ```container``` που δημιουργήθηκε, ιδιάιτερα, χρειαζόμαστε το αναγνωριστικό ```CONTAINER-ID```.

### Πρόσβαση στο shell του ```container```
```console
user@msnlab:~$ sudo docker exec -u root -t -i CONTAINER-ID /bin/bash
root@msnlab:~#
```

Εκτελώντας τη παραπάνω εντολή έχει ως αποτέλεσμα να αποκτήσουμε ```root``` πρόσβαση στο shell του container.

## Ρύθμιση Quagga με υποστήριξη SNMP

Ενημερώνουμε τα τοπικά αποθετήρια και εγκαθιστούμε τυχόν ενημερώσεις στο σύστημα
```console
root@msnlab:~# apt update && apt dist-upgrade -y 
```

Εγκαθιστούμε όλα τα απαραίτητα πακέτα λογισμικού (dependencies)
```console
root@msnlab:/# apt install git make snmp snmpd snmptrapd snmp-mibs-downloader automake autoconf libtool texinfo gawk pkg-config libreadline-dev libc-ares-dev libsnmp-dev
```

Κατεβάζουμε το πηγαίο κώδικα του Quagga από το αποθετήριο μεταβαίνουμε στον φάκελο
```console
root@msnlab:/# git clone https://git.savannah.gnu.org/git/quagga.git && cd quagga/
```
Θέτουμε την μεταβλητή περιβάλλοντος ```LD_LIBRARY_PATH``` με σκοπό ο μεταγγλωτιστής να γνωρίζει πως οι βιβλιοθήκες σε αυτό το ```container``` βρίσκονται στο φάκελο ```/usr/local/lib```
```console
root@msnlab:/quagga# export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib
```
Βάσει οδηγιών από το [INSTALL.quagga.txt](http://git.savannah.gnu.org/cgit/quagga.git/tree/INSTALL.quagga.txt) εκτελούμε τα εξής ώστε να αρχικοποιήσουμε τα απαραίτητα αρχεία για την εγκατάσταση.
```console
root@msnlab:/quagga# automake --add-missing
root@msnlab:/quagga# ./bootstrap.sh
```
Επίσης, πρέπει να προσθέσουμε επιπλέον μια μεταβλητή περιβάλλοντος με σκοπό να ρυθμίσουμε το ```Quagga``` να χρησιμοποιεί τις βιβλιοθήκες που βρίσκονται στο φάκελο ```/usr/local/lib```.
```console
root@msnlab:/quagga# export LDFLAGS='-L/usr/local/lib' && ldconfig
```
Έπειτα ρυθμίζουμε το Quagga να κάνει χρήση του πρωτοκόλλου ```SNMP``` και να τρέχει ως χρήστης ```root``` καθώς και να διαβάζει τις ρυθμίσεις από το φάκελο ```/etc/quagga```.
```console
root@msnlab:/quagga# ./configure --enable-snmp=agentx --enable-user=root --enable-group=root --sysconfdir=/etc/quagga
```
Μετά την επιτυχή ρύθμιση, προχωράμε στην εγκατάσταση.
```console
root@msnlab:/quagga# make install
```
## Ρύθμιση υπηρεσίας πρωτοκόλλου SNMP
Μετά την εγκατάσταση του Quagga προχωράμε στις ρυθμίσεις της υπηρεσίας SNMP.

Το πρώτο αρχείο που θα πρέπει να επεξεργαστούμε είναι το ```/etc/snmp/snmpd.conf```, όπου και βρίσκονται οι κύριες ρυθμίσεις της υπηρεσίας SNMP.
Ανοίγουμε το αρχείο αυτό με το ```nano```.
```console
root@msnlab:/quagga# nano /etc/snmp/snmpd.conf
```
Βάζουμε σε σχόλια τα παρακάτω πεδία:
- agentAddress  udp:127.0.0.1:161
- extend    test1
- extend-sh test2

Στο τέλος του αρχείου, βρίσκουμε τη γραμμή ```master agentx``` και προσθέτουμε τα παρακάτω:
- agentXSocket /var/agentx/master
- agentXPerms 777 777

Ουσιαστικά προσθέσαμε επιπλέον παραμέτρους ώστε η υπηρεσία SNMP να λειτουργεί στο ```/var/agentx/master``` socket.

Το επόμενο αρχείο ρυθμίσεων που πρέπει να επεξεργαστούμε είναι το ```/etc/default/snmpd```.
```console
root@msnlab:/quagga# nano /etc/default/snmpd
```
Βάζουμε σε σχόλια το παρακάτω:
- export MIBS=

Αλλάζουμε το ```SNMPDOPTS``` ως εξής:
- SNMPDOPTS='-Lsd -Lf /tmp/snmpd.log -u root -g root -I -agentx -p /var/run/snmpd.pid -c /etc/snmp/snmpd.conf'

Ουσιαστικά ενημερώνουμε την υπηρεσία να κρατά logs στο αρχείο ```/tmp/snmpd.log```, να τρέχει με τα δικαιώματα του χρήστη ```root``` και να χρησιμοποιεί ως αρχείο ρυθμίσεων, το ```/etc/snmp/snmpd.conf```.

Μένει να ρυθμίσουμε την υπηρεσία SNMP να ξεκινά κάθε φορά που ξεκινά και το σύστημα.
Στο αρχείο ```/etc/bash.bashrc``` προσθέτουμε το παρακάτω:
```console
root@msnlab:/quagga# nano /etc/bash.bashrc
```

- \#Enable SNMP service on start-up
- /usr/sbin/service snmpd start

Κάνουμε επανεκκίνηση της υπηρεσίας SNMP
```console
root@msnlab:/quagga# service snmpd restart
```
Με το ```snmpwalk``` έαν όλα λειτουργούν σωστά θα πρέπει να λάβουμε αρκετές πληροφορίες σχετικά με το σύστημα μέσω της υπηρεσίας SNMP
```console
root@msnlab:/quagga# snmpwalk -v 1 -c public 127.0.0.1
```

Εφόσον ρυθμίσαμε το Quagga σωστά με υποστήριξη SNMP, το μόνο που απομένει είναι να αποθηκέυσουμε τις αλλαγές που κάναμε σε αυτό το container στο κύριο docker image που χρησιμοποιεί το IMUNES με σκοπό ο κάθε κόμβος στο IMUNES να διαθέτει την υπηρεσία SNMP.

Τερματίζουμε τη σύνδεση μας με το docker container.
```console
root@msnlab:/quagga# exit
```
Υποβάλλουμε τις αλλαγές του container στο imunes/template docker image.
```console
user@msnlab:~$ sudo docker commit CONTAINER-ID imunes/template
```


# Εγκατάσταση OpenNMS

```console
user@msnlab:~$ 
```

## Virtual Machine Credentials

```
Username        Password
user            msnlab1234
```