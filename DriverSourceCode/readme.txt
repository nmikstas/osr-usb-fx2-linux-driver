**********************************Problem Statement**********************************

The purpose of this project is to create a USB driver for the ORS FX2 USB learning board and demonstrate the capabilities of the board with a user space application.  The OSR FX2 board was designed to teach programmers how to write USB drivers with the Windows Driver Kit (WDK).  The WDK comes with two drivers for use with the Windows operating system.  This project will create a Linux kernel driver for the board.

The board includes the following features:

    - A 7 segment display that can be set and read through vendor commands.

    - An 8 LED bargraph than can be set and read through vendor commands.

    - An 8 switch DIP switch pack that can be read through a vendor command.
      The 8 switch status is sent by use of an interrupt endpoint back to the
      host every time the user changes the position of a switch.

    - Bulk out and a bulk in endpoints.  The two bulk endpoints are configured
      in a loopback scheme.  Information written to the bulk out endpoint can
      be read back by the host through the bulk in endpoint.

Detailed information about the OSR FX2 board can be found at the following website:
http://www.osronline.com/hardware/OSRFX2_35.pdf

***************************************Design****************************************

The design of this driver is based on the usb-skeleton.c file found under the drivers/usb folder of the kernel source code.  The major functions of the driver and a summary of their tasks are as follows:

-init_module.  Called when the USB driver is inserted into the kernel.
    1. Registers the driver with the USB core (usb_register).

-cleanup_module.  Called when the USB driver is removed from the kernel.
    1. Deregisters the driver with the USB core (usb_deregister).

-probe.  Called when the USB device is connected to the host and after enumeration has occurred.
    1. Create and initialize the device context structure.
    2. Create sysfs files for the various device components (device_create_file).
       These files appear in the sys/class/usb/(usb device)/device/ directory.
    3. Increment through struct usb_interface->cur_altsetting-> desc.bNumEndpoints
       to find the various endpoint addresses and types.  Store this information
       in the device context structure. A pointer to struct usb_interface is
       passed by the USB core to the probe function.
    4. Initialize interrupts (usb_rcvintpipe).
    5. Create interrupt endpoint buffer.
    6. Create interrupt endpoint URB (usb_alloc_urb).
    7. Fill interrupt endpoint URB (usb_fill_int_urb).
    8. Submit interrupt URB to USB core (usb_submit_urb).
    9. create bulk endpoint buffers.
    10. Register device (usb_register_dev).

-disconnect.  Called when the device is unplugged from the host.
    1. Release interface resources (usb_put_dev, usb_free_urb, usb_set_intfdata).
    2. Return minor number to driver core (usb_deregister_dev).
    3. Release all URB resources (usb_kill_urb).
    4. Remove files from sysfs (device_remove_file).
    5. Decrement device reference count (kref_put).

-suspend
    1. Stop the interrupt URB (usb_kill_urb).

-resume
    1. Restart the interrupt URB (usb_submit_urb).

-open.  Called when /dev/osrfx2_0 is opened.
    1. Reset bulk out pipe (usb_clear_halt).
    2. Reset bulk in pipe (usb_clear_halt).
    3. Increment device reference count (kref_get).
    4. Save pointer to device instance for future reference.

-close.  Called when /dev/osrfx2_0 is closed.
    1. Clear bulk read and bulk write available status.
    2. Decrement device reference count (kref_put).

-read.  Called when /dev/osrfx2_0 is read from.
    1. Initialize pipe (usb_rcvbulkpipe).
    2. Read bulk in data (usb_bulk_msg).
    3. Copy data to user space (copy_to_user).

-write.  Called when data is written to /dev/osrfx2_0.
    1. Create URB (usb_alloc_urb).
    2. Create URB buffer (usb_alloc_coherent).
    3. Copy data to write buffer (copy_from_user).
    4. Initialize the URB (usb_sndbulkpipe, usb_fill_bulk_urb).
    5. Send the data to the device (usb_submit_urb).
    6. Release the URB (usb_free_urb).

-write_callback
    1. Check for device errors that may have occurred during the write.
    2. Release the write buffer (usb_free_coherent).

-interrupt_handler.  Called when interrupt received from device.
    1. Get interrupt data.
    2. Restart interrupt URB (usb_submit_urb).

-Vendor commands.  Vendor commands are sent using the usb_control_msg function.

-Create device attributes in the sysfs.  When the attribute files are read and
 written in the sysfs, corresponding get and set commands are called to control
 the 7 segment display, bargraph and switches ( DEVICE_ATTR).

The following figures show how the status bits displayed to the user match up to the actual hardware on the OSR FX2 board:

   7 Segment Display       Bargraph Bit                Switch Bit
    Bit Correlation        Correlation                 Correlation

          0                   [===] 7        [ ] [ ] [ ] [ ] [ ] [ ] [ ] [ ]
        -----                 [===] 6         7   6   5   4   3   2   1   0
       |     |                [===] 5
     5 |  6  | 1              [===] 4        NOTE: The switch bit correlation
        -----                 [===] 3        is opposite the switch labels on
       |     |                [===] 2        the actual board.
     4 |     | 2              [===] 1
        -----     * 7         [===] 0
          3

*************************************How to Use**************************************

In order to use the OSR FX2 board demonstraion, the driver module must be compiled and inserted into the kernel.  The OSR FX2 board must also be attached to the computer.  The 7 segment display and bargraph can be read and set through device attributes located in the sysfs.  The current state of the DIP switches can also be read through device attributes in the sysfs.  Information can be written to and read from the bulk endpoints through the devfs.

The following are examples of how to manually control the OSR FX2 board:

Read the status of the 7 segment display:
cat /sys/class/usb/osrfx2_0/device/7segment

Turn on all the LEDs in the 7 segment display:
echo 255 > /sys/class/usb/osrfx2_0/device/7segment

Read the status of the bargraph:
cat /sys/class/usb/osrfx2_0/device/bargraph

Turn on all of the LEDs in the bargraph:
echo 255 > /sys/class/usb/osrfx2_0/device/bargraph

Read the status of the switches:
cat /sys/class/usb/osrfx2_0/device/ switches

Write information to the bulk out endpoint:
echo "This is a test" > /dev/osrfx2_0

Read information from the bulk in endpoint:
cat /dev/osrfx2_0

************************************How to Build*************************************

The makefile included with this design will perform all the necessary tasks for building the driver module, inserting the module into the kernel, compiling the user application and running the user application.  The object files can be deleted and the module removed from the kernel by simply typing make clean.

NOTE: the make file must be run by root in order to work properly.

*******************************Output From Executable********************************

The following is a sample output from the application my_usb_app.c.  The program displays the initial condition of the switches, 7 segment display and bargraph.  It sends test packets to the device every 5 seconds.  The application then reads the information back from the device.  The read information should be identical to the written information.  The test packet number is incremented for every packet.  Every time the user changes the position of a DIP switch, the application reports it back to the user.  At the time a switch update is sent to the user, the current state of the 7 segment display and bargraph are also sent.

Switch status:    00000000
7 segment status: 10100000
Bargraph status:  01000010

Writing to bulk endpoint: Test packet 0
Read from bulk endpoint:  Test packet 0

Writing to bulk endpoint: Test packet 1
Read from bulk endpoint:  Test packet 1

Switch status:    00010000
7 segment status: 00000100
Bargraph status:  00100100

Switch status:    00010100
7 segment status: 10100000
Bargraph status:  01000010

Switch status:    00010111
7 segment status: 10100000
Bargraph status:  01000010

Writing to bulk endpoint: Test packet 2
Read from bulk endpoint:  Test packet 2

Switch status:    11010111
7 segment status: 10100000
Bargraph status:  01000010

Switch status:    11110111
7 segment status: 00010000
Bargraph status:  00100100

Switch status:    11111111
7 segment status: 00010000
Bargraph status:  00100100

Switch status:    11011111
7 segment status: 10001000
Bargraph status:  00011000

Writing to bulk endpoint: Test packet 3
Read from bulk endpoint:  Test packet 3

Switch status:    11010111
7 segment status: 10000010
Bargraph status:  01000010

Switch status:    11010011
7 segment status: 00010000
Bargraph status:  00100100

Switch status:    11010010
7 segment status: 00000100
Bargraph status:  00100100

Switch status:    01010010
7 segment status: 10000010
Bargraph status:  01000010

Switch status:    00010010
7 segment status: 00010000
Bargraph status:  00100100

Switch status:    00000010
7 segment status: 00000100
Bargraph status:  00100100

Writing to bulk endpoint: Test packet 4
Read from bulk endpoint:  Test packet 4

Writing to bulk endpoint: Test packet 5
Read from bulk endpoint:  Test packet 5

Switch status:    00000000
7 segment status: 00000001
Bargraph status:  00000011

Writing to bulk endpoint: Test packet 6
Read from bulk endpoint:  Test packet 6

Writing to bulk endpoint: Test packet 7
Read from bulk endpoint:  Test packet 7
