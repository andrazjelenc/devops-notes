# Mikrotik

## Serial console
1. Get a correct serial cable. I am using [USB-A to RJ45 cable with FT232RL chip](https://krajnik.si/proizvod/ugreen-usb-a-na-rj45-kabel-mrezna-kartica-1-5m/).
2. On your PC install drivers for your cable. Read manuals that comes with the cable.
3. Connect console cable to your PC.
4. Identify serial port by opening "Computer management" and then "Device Manager". Your port should be specified under Ports (COM & LPT). Let's assume it is on COM3.
5. Open Putty and under Connection->Serial set:
    - Serial line: COM3
    - Speed: 115200
    - Data bits: 8
    - Stop bits: 1
    - Parity: None
    - Flow control: None
    
    Navigate to Session tab, select Serial and click on Open button.
6. Press ENTER.
7. Login using your credentials.