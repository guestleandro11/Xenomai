# Xenomai
Guida in Italiano per installare Xenomai su Raspberry Pi3 B+
Introduzione: 
È possibile compilare il kernel sia in cross-compile ovvero usando un PC host per la compila-
zione, oppure direttamente dal raspberry.
Di seguito vengono presentate due guide, la prima utilizza un approccio manuale dove è possi-
bile modificare il kernel a proprio piacimento, mentre nella seconda guida si utilizza uno
script e non è possibile effettuare alcuna modifica al kernel xenomai.
invece un approccio automatico e

Inserire tutti i file in una cartella chiamata "Prebuild"

Preparazione:
-Raspberry Pi3 model B+;
-Immagine di raspbian con kernel 4.9.x
 http://downloads.raspberrypi.org/raspbian/images/raspbian-2018-03-14/
-Installare il sistema operativo sul raspberry;
-Connettere il raspberry ad internet;
-eseguire apt-get update;
-NON eseguire apt-get upgrade.


GUIDA MANUALE
/*******************************************************************************************/
Da Host (o dal raspberry se lo si usa per compilare il kernel):

#sudo apt-get install subversion
#sudo apt-get install gcc-arm-linux-gnueabihf (non necessario)
#sudo apt-get install --no-install-recommends ncurses-dev bc

Scaricare xenomai-3
#wget wget https://xenomai.org/downloads/xenomai/stable/xenomai-3.0.7.tar.bz2
#tar -xjvf xenomai-3.0.7.tar.bz2
#ln -s xenomai-3.0.7 xenomai

Aprire con un editor di testo come geany il file:
#geany xenomai/scripts/prepare-kernel.sh
e modificare "ln -sf" in "cp"

Scaricare rpi-linux-4.9.x:
#git clone -b rpi-4.9.y --depth 1 git://github.com/raspberrypi/linux.git linux-rpi-4.9.y-xeno3
#ln -s linux-rpi-4.9.y-xeno3 linux

Crea una cartella (da /home/pi):
#mkdir xeno3-patches

Scaricare dal browser dal sito:
https://github.com/thanhtam-h/rpi23-4.9.80-xeno3
la cartella "scripts" che contiene al suo interno due file (README e ipipe)
scompatta l'archivio e copia i file contenuti nella cartella "script" nella cartella creata
in precedeza "xeno3-patches".

Inizio del patching e preparazione del kernel:
#cd linux
#../xenomai/scripts/prepare-kernel.sh --linux=./  --arch=arm  --ipipe=../xeno3-patches/ipipe-core-4.9.51-arm-4-for-4.9.80.patch
#make -j4 O=build ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- bcm2709_defconfig
#make O=build ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j4 menuconfig

Si aprirà il menù di configurazione del kernel, selezionare le seguenti opzioni obbligatorie:

General setup ---> │ │ (-v7-xeno3) Local version - append to kernel release │ │ Stack Protector buffer overflow detection (None) --->

Kernel Features  --->
│ │                                Preemption Model (Preemptible Kernel (Low-Latency Desktop))  --->                              
│ │                                Timer frequency (1000 Hz)  --->   
│ │                            [ ] Allow for memory compaction
│ │                            [ ] Contiguous Memory Allocator

CPU Power Management  --->
│ │                                CPU Frequency scaling  --->
                                    [ ] CPU Frequency scaling
								
Kernel hacking  --->
│ │                            [ ] KGDB: kernel debugger  ---	

Aggiungere nel relativo campo Xenomai eventuali moduli da inserire.

Build image, moduli e device overlay (sono necessarie diverse ore):
#make O=build ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j4 bzImage modules dtbs

Installazione dei moduli:
#make O=build ARCH=arm INSTALL_MOD_PATH=dist -j4 modules_install

Build package (sono necessarie diverse ore):
#make O=build ARCH=arm KBUILD_DEBARCH=armhf  CROSS_COMPILE=arm-linux-gnueabihf- -j4 deb-pkg 

Comprimere i file dts:
#cd build/arch/arm/boot
#tar -cjvf linux-dts-4.9.y-xeno3+.tar.bz2 dts
#cd ../../../..
#cp build/arch/arm/boot/linux-dts-4.9.y-xeno3+.tar.bz2 ./ 

Adesso terminati questi processi, nella cartella linux abbiamo diversi file, tra cui:
-linux-image*, 
-linux-headers*,
-linux-dts*.

Creare nella cartella principale (/home/pi) una cartella chiamata ad esempio "prebuild":
#mkdir prebuild
e copiare all'interno di questa cartella i file creati precedentemente che si trovano dentro
la cartella linux:
-linux-image*, 
-linux-headers*,
-linux-dts*

Posizionarsi nella cartella xenomai:
#cd /home/pi/xenomai
ed eseguire:
#./scripts/bootstrap (non necessario)
#./configure --host=arm-linux-gnueabihf --enable-smp --with-core=cobalt
#make
#sudo make install

Terminata l'installazione eseguire:
#tar -cjvf rpi23-xeno3-deploy.tar.bz2 /usr/xenomai
e copiare anche questo file tar.bz2 nella cartella "prebuild" e scompattare l'archivio:
# sudo tar -xjvf rpi23-xeno3-deploy.tar.bz2 -C /

Copiare nella cartella prebuild il file "xenomai.conf" presente nella cartella di questa guida.

Adesso aprire il terminale nella cartella "prebuild" che contiene i vari file creati e 
eseguire i seguenti comandi ignorando eventuali errori:
#sudo dpkg -i linux-image*
#sudo dpkg -i linux-headers*
#sudo tar -xjvf 4.9.y-dts.tar.bz2
#cd dts
#sudo cp -rf * /boot/
#sudo mv /boot/vmlinuz-4.9.80-v7-xeno3+ /boot/kernel7.img
#cd ..
#sudo tar -xjvf rpi23-xeno3-deploy.tar.bz2 -C /
#sudo cp xenomai.conf /etc/ld.so.conf.d/
#sudo ldconfig
#sudo reboot

Una volta che il sistema si è riavviato eseguire da terminale:
#cd /usr/src/linux-headers-4.9.80-v7-xeno3+/
#sudo make -i modules_prepare
questo richiederà qualche minuto, ignora eventuali errori.

Cpu affinity: 
aprire con un editor di testo il file cmdline.txt:
#sudo nano /boot/cmdline.txt

Aggiungi alla fine del testo sulla stessa riga:
isolcpus=0,1,2,3 xenomai.supported_cpus=0xF
 
Test:
per controllare se tutto funzioni correttamente esegui:
#sudo /usr/xenomai/bin/latency

/*******************************************************************************************/

GUIDA AUTOMATICA

Assisema a questa guida, possiamo trovare una cartella chiamata "prebuild", copiala nella 
directory principale del raspberry (/home/pi)

Aprire il terminale ed eseguire:
#sudo apt-get install subversion

Spostarsi nella cartella prebuild appena copiata ed eseguire:
#chmod +x deploy.sh
#./deploy.sh

Terminato il processo la raspberry si riavvierà.
Quando la raspberry è riavviata eseguire da terminale:
#cd /usr/src/linux-headers-4.9.80-v7-xeno3+/
#sudo make -i modules_prepare

Cpu affinity: 
aprire con un editor di testo il file cmdline.txt:
#sudo nano /boot/cmdline.txt

Aggiungi alla fine del testo sulla stessa riga:
isolcpus=0,1,2,3 xenomai.supported_cpus=0xF
 
Test:
per controllare se tutto funzioni correttamente esegui:
#sudo /usr/xenomai/bin/latency

Per donazioni:
https://paypal.me/LeandroBucci?locale.x=it_IT








