obj-m	:= my_usb_driver.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
	insmod ${PWD}/my_usb_driver.ko
	gcc my_usb_app.c -o my_usb_app
	./my_usb_app
	
clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
	rm -f my_usb_app my_usb_app?
	rmmod my_usb_driver
	
