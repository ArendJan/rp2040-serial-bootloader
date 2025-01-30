# RP2040 Serial Bootloader

This is a bootloader for the RP2040 which enables code upload via UART.

There's a more complete description at: https://blog.usedbytes.com/2021/12/pico-serial-bootloader/

There are currently two tools I know of which can be used to upload code to it:

* [serial-flash](https://github.com/usedbytes/serial-flash) - my tool written in `go`
* [pico-py-serial-flash](https://github.com/ConfedSolutions/pico-py-serial-flash) - a similar tool written in Python, contributed by another user.



# Usage
Add this folder as a submodule to your project

Add
```
set(SKIP_BOOTLOADER_ENTRY_PIN 0) # skip bootloader gpio reading if wanted
add_subdirectory(rp2040-serial-bootloader)
bootloader_build_combined($PROJECT_NAME)
```
to your CMakeLists.txt

Run make like usual

Output files will be 
$PROJECT_NAME_combined.elf/.uf2/.bin 

The uf2 you can flash like normal with picotool.
The normal uf2 