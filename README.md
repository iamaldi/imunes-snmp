## Περιγραφή

Το [```OpenNMS```](https://www.opennms.com/) είναι μια πλατφόρμα παρακολούθησης και διαχείρησης δικτύων. Θα χρησιμοποιήσουμε τη λειτουργία ανακάλυψης (Discovery) του OpenNMS για να ανακαλύψουμε κόμβους στο δίκτυό μας και κάνοντας χρήση του πρωτοκόλλου SNMP να λάβουμε πληροφορίες σχετικά με το κάθε κόμβο που συναντάμε. Το δίκτυο μας θα είναι ένα προσομοιωμένο δίκτυο μέσα από το IMUNES.

Το [```IMUNES```](http://imunes.net/) είναι ένας προσομοιωτής δικτύου ο οποίος χρησιμοποιεί το [```Quagga```](https://www.quagga.net/). Το Quagga είναι ένα πακέτο λογισμικού δρομολόγησης. Για κάθε εικονικό κόμβο δικτύου, το IMUNES τρέχει ένα docker ```container``` το οποίο κάνει χρήση ενός έτοιμου docker ```image```, το [```imunes/template```](https://hub.docker.com/r/imunes/template) image.

Το έγγραφο αυτό παρουσιάζει τα απαραίτητα βήματα για την:
- εγκατάσταση του IMUNES
- επεξεργασία του ```imunes/template``` docker image
- ρύθμιση και εγκατάσταση του Quagga με υποστήριξη πρωτοκόλλου SNMP
- ρύθμιση υπηρεσίας πρωτοκόλλου SNMP
- αποθήκευση αλλαγών στο ```imunes/template``` docker image
- εγκατάσταση και ρύθμιση του OpenNMS

## Περιβάλλον Εγκατάστασης
Το λειτουργικό σύστημα που χρησιμοποιήθηκε για αυτή την εργασία είναι το *Xubuntu 20.04.1 LTS (Focal Fossa) 64-bit* σε εικονικό μηχάνημα *VirtualBox*. Στο σύστημα αυτό ο χρήστης ```user``` έχει δικαιώματα root και το όνομα του τερματικού είναι ```msnlab```. Η πρόσβαση στο docker container θα εμφανίζεται ώς ```root@msnlab``` καθώς το container χρησιμοποιεί το όνομα του συστήματος ως δικό του μιας και μοιραζόνται την ίδια σύνδεση δικτύου.

## Εγκατάσταση IMUNES
Αρχικά, ενημερώνουμε και εγκαθιστούμε τυχόν αναβαθμίσεις στο σύστημα.

```console
user@msnlab:~$ sudo apt update && sudo apt dist-upgrade -y
```

Εγκαθιστούμε τα απαραίτητα πακέτα λογισμικού για το IMUNES.
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

Πλέον, μπορούμε να τρέξουμε το IMUNES ως εξής:
```console
user@msnlab:~$ sudo imunes
```
Στο σημείο αυτό έχουμε ολοκληρώσει την εγκατάσταση του IMUNES και το επόμενο βήμα είναι να ρυθμίσουμε το Quagga έτσι ώστε υποστηρίζει το πρωτόκολλο SNMP.

## Επεξεργασία του ```imunes/template``` docker image
Επιβεβαιώνουμε πως διαθέτουμε τοπικά το ```imunes/template``` docker image
```console
user@msnlab:~$ sudo docker image ls
REPOSITORY          TAG         IMAGE ID            CREATED SIZE
imunes/template     latest      28ab347bef71        934MB
```

Εκτελούμε το image αυτό σε ένα container ως εξής:
```console
user@msnlab:~$ sudo docker run --detach --tty --net='host' imunes/template
```
Το container αυτό θα έχει πρόσβαση στο ίδιο δίκτυο με τον host διότι χρειαζόμαστε πρόσβαση στο διαδίκτυο και αυτό το επιτυχγάνουμε με τη παράμετρο ```--net='host'```.

Επιβεβαιώνουμε πως το ```container``` δημιουργήθηκε επιτυχώς.
```console
user@msnlab:~$ sudo docker container ls
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
d3cdf12c8d50        c009531c75fd        "/bin/bash"         10 seconds ago      Up 10 seconds                             clever_babbage
```

Από το αποτέλεσμα της παραπάνω εντολής θα πρέπει να λάβουμε πληροφορίες σχετικά με το ```container``` που δημιουργήθηκε, ιδιάιτερα, χρειαζόμαστε το αναγνωριστικό ```CONTAINER-ID``` σε αυτή τη περίπτωση το ```d3cdf12c8d50```.

Πρόσβαση στο shell του ```container```
```console
user@msnlab:~$ sudo docker exec -u root -t -i d3cdf12c8d50 /bin/bash
root@msnlab:~#
```
Εκτελώντας τη παραπάνω εντολή έχει ως αποτέλεσμα να αποκτήσουμε ```root``` πρόσβαση στο shell του container.

## Ρύθμιση και εγκατάσταση του Quagga με υποστήριξη πρωτοκόλλου SNMP

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

Τροποποιούμε το πεδίο ```sysLocation``` από την προεπιλεγμένη τιμή σε:
- sysLocation       University of Macedonia

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
Εαν όλα λειτουργούν σωστά, με το ```snmpwalk``` θα πρέπει να λάβουμε αρκετές πληροφορίες σχετικά με το σύστημα μέσω της υπηρεσίας SNMP με την εξής εντολή.
Όπου:
- το ```-v 1``` είναι η έκδοση του SNMP που χρησιμοποιούμε για αυτή τη δοκιμή
- το ```-c public``` είναι το community string και μπορεί να βρεθεί στο αρχείο /etc/snmp/snmpd.conf
- και τέλος το ```127.0.0.1``` είναι η IPv4 διέυθυνση του συστήματος του οποίου προσπαθούμε να επικοινωνήσουμε με την SNMP υπηρεσία του. Μπορεί να είναι οποιοσδήποτε άλλος κόμβος που προσφέρει την υπηρεσία SNMP.

```console
root@msnlab:/quagga# snmpwalk -v 1 -c public 127.0.0.1
iso.3.6.1.2.1.1.1.0 = STRING: "Linux msnlab 5.4.0-42-generic #46-Ubuntu SMP Fri Jul 10 00:24:02 UTC 2020 x86_64"
iso.3.6.1.2.1.1.2.0 = OID: iso.3.6.1.4.1.8072.3.2.10
iso.3.6.1.2.1.1.3.0 = Timeticks: (8074890) 22:25:48.90
iso.3.6.1.2.1.1.4.0 = STRING: "Me <me@example.org>"
iso.3.6.1.2.1.1.5.0 = STRING: "msnlab"
iso.3.6.1.2.1.1.6.0 = STRING: "University of Macedonia"
iso.3.6.1.2.1.1.7.0 = INTEGER: 72
iso.3.6.1.2.1.1.8.0 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.2.1 = OID: iso.3.6.1.6.3.11.3.1.1
iso.3.6.1.2.1.1.9.1.2.2 = OID: iso.3.6.1.6.3.15.2.1.1
iso.3.6.1.2.1.1.9.1.2.3 = OID: iso.3.6.1.6.3.10.3.1.1
iso.3.6.1.2.1.1.9.1.2.4 = OID: iso.3.6.1.6.3.1
iso.3.6.1.2.1.1.9.1.2.5 = OID: iso.3.6.1.6.3.16.2.2.1
iso.3.6.1.2.1.1.9.1.2.6 = OID: iso.3.6.1.2.1.49
iso.3.6.1.2.1.1.9.1.2.7 = OID: iso.3.6.1.2.1.4
iso.3.6.1.2.1.1.9.1.2.8 = OID: iso.3.6.1.2.1.50
iso.3.6.1.2.1.1.9.1.2.9 = OID: iso.3.6.1.6.3.13.3.1.3
iso.3.6.1.2.1.1.9.1.2.10 = OID: iso.3.6.1.2.1.92
iso.3.6.1.2.1.1.9.1.3.1 = STRING: "The MIB for Message Processing and Dispatching."
iso.3.6.1.2.1.1.9.1.3.2 = STRING: "The management information definitions for the SNMP User-based Security Model."
iso.3.6.1.2.1.1.9.1.3.3 = STRING: "The SNMP Management Architecture MIB."
iso.3.6.1.2.1.1.9.1.3.4 = STRING: "The MIB module for SNMPv2 entities"
iso.3.6.1.2.1.1.9.1.3.5 = STRING: "View-based Access Control Model for SNMP."
iso.3.6.1.2.1.1.9.1.3.6 = STRING: "The MIB module for managing TCP implementations"
iso.3.6.1.2.1.1.9.1.3.7 = STRING: "The MIB module for managing IP and ICMP implementations"
iso.3.6.1.2.1.1.9.1.3.8 = STRING: "The MIB module for managing UDP implementations"
iso.3.6.1.2.1.1.9.1.3.9 = STRING: "The MIB modules for managing SNMP Notification, plus filtering."
iso.3.6.1.2.1.1.9.1.3.10 = STRING: "The MIB module for logging SNMP Notifications."
iso.3.6.1.2.1.1.9.1.4.1 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.2 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.3 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.4 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.5 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.6 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.7 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.8 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.9 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.10 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.25.1.1.0 = Timeticks: (42810291) 4 days, 22:55:02.91
iso.3.6.1.2.1.25.1.2.0 = Hex-STRING: 07 E4 08 1A 10 02 1C 00 2B 00 00 
iso.3.6.1.2.1.25.1.3.0 = INTEGER: 393216
iso.3.6.1.2.1.25.1.4.0 = STRING: "BOOT_IMAGE=/boot/vmlinuz-5.4.0-42-generic root=UUID=08550881-62ca-4260-98c1-dad2b7f21ed9 ro quiet splash
"
iso.3.6.1.2.1.25.1.5.0 = Gauge32: 0
iso.3.6.1.2.1.25.1.6.0 = Gauge32: 4
iso.3.6.1.2.1.25.1.7.0 = INTEGER: 0
End of MIB
```

## Αποθήκευση αλλαγών στο ```imunes/template``` docker image
Εφόσον ρυθμίσαμε το Quagga σωστά με υποστήριξη SNMP, το μόνο που απομένει είναι να αποθηκέυσουμε τις αλλαγές που κάναμε σε αυτό το container στο κύριο docker image που χρησιμοποιεί το IMUNES με σκοπό ο κάθε κόμβος στο IMUNES να διαθέτει την υπηρεσία SNMP.

Τερματίζουμε τη σύνδεση μας με το docker container.
```console
root@msnlab:/quagga# exit
```
Υποβάλλουμε τις αλλαγές του container στο imunes/template docker image.
```console
user@msnlab:~$ sudo docker commit d3cdf12c8d50 imunes/template
```
Πλέον ο κάθε εικονικός κόμβος στο IMUNES θα εμπεριέχει τις αλλαγές που κάναμε. Δηλαδή, ο κάθε κόμβος θα διαθέτει την υπηρεσία SNMP μέσω της οποίας θα μπορούμε να λαμβάνουμε πληροφορίες συστήματος κάνοντας χρήση του OpenNMS.

## Εγκατάσταση και ρύθμιση του OpenNMS
```console
user@msnlab:~$ sudo apt install openjdk-11 ...
```

## Διαπιστευτήρια Εικονικής Μηχανής

```
Username        Password
user            msnlab1234
```