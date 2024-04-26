
# Como flashear el Octopus Pro y EBB 2209 (RP2040) usando Katapult
_Las explicaciones están en cursiva. Puedes ignorarlas si sólo quieres flashear. Los créditos son a @bkpkt_patrick en el Discord Siboor para mostrar este método de archivo de configuración y @Krhomv por crear este documento originalmente en ingles._

### 0. Conéctate a tu Raspberry Pi/BTT Pi a través de SSH utilizando su método preferido (recomiendo MobaXTerm).
Todos los comandos se ejecutarán en la Pi.

### 1. Transfiera los archivos de configuración a su carpeta klipper
_Esos archivos de configuración contienen la información de configuración para el Octopus Pro y EBB 2209 (RP2040) específicamente. Esto te ahorrará tener que meterse con `make menuconfig` y reducirá las posibilidades de dañarlo._
```bash
cd ~/klipper
wget -O config.octopus https://raw.githubusercontent.com/Krhomv/Voron2.4Config/main/config.octopus
wget -O config.sb2209 https://raw.githubusercontent.com/Krhomv/Voron2.4Config/main/config.sb2209
```

### 2. Asegúrese de tener Katapult instalado
_Katapult es una herramienta que te permite flashear tus MCUs usando canbus y sin tener que acceder físicamente a las placas._
```bash
cd
ls katapult
```
Si obtiene un error, entonces necesita instalar Katapult:
```bash
git clone https://github.com/Arksine/katapult
```

### 3. Anote sus ID de CANbus
Abre el bloc de notas o su editor de texto preferido (recomiendo Notepadd++)
Abre su printer.cfg y busque algo como `canbus_uuid: d9093b323a18`. Deberían aparecer dos.
Anota el que está debajo del `[mcu]` este es tu Octopus CAN id.
Anota el que está debajo del `[mcu EBBCan]` (o similar), este es su tablero Toolhead, el EBB 2209 (RP2040).
El documento del bloc de notas debería tener este aspecto:
```
Octopus Pro CAN Id: d9093b323a18
EBB 2209 CAN Id: 1ebcd26c2bc9
```

## Flasheo de la Octopus Pro

### 1. Coloca la placa en modo DFU
_Esto usará Katapult, a través del CANbus, para decirle al octopus que se ponga en modo "listo para flashear". Dado que el Octopus está conectado a través de USB pero normalmente actúa como nuestro puente CAN, esto hará que se revierta como un dispositivo USB normal (sin puente), y lo flashearemos a través de USB._
```bash
~/katapult/scripts/flashtool.py -u [Octopus Pro CAN Id] -r
```
Sustituye `[Octopus Pro CAN Id]` con el ID que has anotado. Debería tener el siguiente aspecto:
```bash
~/katapult/scripts/flashtool.py -u d9093b323a18 -r
```

### 2. Recuperar el USB Id del Octopus en modo DFU
```bash
lsusb
```
Esto listará todos los dispositivos usb conectados. Localice el que dice `Device in DFU Mode`, y anote su Id, debería verse así: `0483:df11`. Añade esto a tu documento del bloc de notas:
```bash
Octopus Pro DFU USB Id: 0483:df11
```

### 3.  Construir y flashear la placa
```bash
cd klipper
make clean
make KCONFIG_CONFIG=config.octopus flash FLASH_DEVICE=[Octopus Pro DFU USB Id]
```
Sustituye `[Octopus Pro DFU USB Id]` con el ID que has anotado. Debería tener el siguiente aspecto:
```bash
make KCONFIG_CONFIG=config.octopus flash FLASH_DEVICE=0483:df11
```
En este punto obtendrá el siguiente texto y error:
```bash
Download done.
File downloaded successfully
dfu-util: Error during download get_status

Failed to flash to 0483:df11: Error running dfu-util
```
Ignóralo, no pasa nada. Mientras tengas el `File downloaded successfully` línea, entonces todo funcionó.
_Por lo que tengo entendido, la placa realiza la actualización del firmware inmediatamente y luego no responde a `dfu_util` cuando intenta obtener su estado, presumiblemente para comprobar si el flasheo se ha realizado correctamente. Pero en este caso, ninguna noticia es buena y el flasheo fue exitoso._

### 4. Apague y encienda la máquina
```bash
sudo shutdown now
```
A continuación, apaga y enciende el la impresora. (Debes retirar el jumper utilizado para colcoar la placa en modo DFU)

## Flasheo del EBB 2209 (RP2040)
_Esto es más sencillo porque todo se puede hacer a través de Katapult_

### 1. Construir el firmware
```bash
cd klipper
make clean
make KCONFIG_CONFIG=config.sb2209
```

### 2. Flashear la placa
```bash
sudo ~/katapult/scripts/flash_can.py -u [EBB 2209 CAN Id] -f ~/klipper/out/klipper.bin
```
Sustituye `[EBB 2209 CAN Id]` con el ID que has anotado. Esto debería tener este aspecto:
```bash
sudo ~/katapult/scripts/flash_can.py -u d9093b323a18 -f ~/klipper/out/klipper.bin
```

### 3. Vuelva a encender la máquina. Enhorabuena, ¡ya has flasheado tus MCUs!
