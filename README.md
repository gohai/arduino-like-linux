# What's missing to build an Arduino-like API in Linux userspace

While working on a hardware I/O library for Processing ([Source](https://github.com/processing/processing/tree/master/java/libraries/io/src/processing/io)) there were a few instances where I wished there was a clean, hardware-independent way of getting something done though kernel interfaces instead of manually fiddling in /dev/mem. While this might be acceptable when you're just trying to figure out things on your own, it doesn't seem like a good idea for a tool or library meant to run on a wide range of single-board computers.

Below is an incomplete list - feel free to add your own pain-points via [PR](https://github.com/gohai/arduino-like-linux/pulls) or [issue](https://github.com/gohai/arduino-like-linux/issues/new)!


## Software PWM

SBCs typically only have a limited number of hardware PWM channels, yet a much larger number of GPIO pins. While software PWM might not be sufficient for *serious* applications, it might be just fine for hobbyist use (dimming lights, controlling servos).

The best way to implement this would probably be high-resolution kernel timers, which toggle specific GPIO pins in kernel-space. Bill Gatliff [implemented](https://lkml.org/lkml/2010/2/2/36) this in 2010 for a proposed PWM subsystem, which didn't get merged. GPIO & Pin Control maintainer Linus Walleij mentioned that this generally looks like a fairly valid use case to him.

Ideal case for the Processing IO library: writing to `/sys/class/gpio/gpioN/software_pwm` would create a channel in `/sys/class/pwm`.


## Runtime pullup configuration

The Linux kernel knows the concepts of pull-up and pull-down resistors in `GENERIC_PINCONF`, but they can currently only be set by the user through device-tree overlays, and not at run-time. For interfacing with digital and analog hardware it would, however, be very convenient to be able to do so - like it is commonly done on the Arduino. This would also be a huge help to keep the number of required external components and connections at a minimum.

Linus Walleij suggested using debugfs for such an interface, which would not be ideal, since it doesn't seem to be always mounted, and require root provileges. Perhaps  a sysfs interface that's disabled by default?

Ideal case for the Processing IO library: `/sys/class/gpio/gpio/gpioN/bias`.


## Make PWM channel exports show up in udev

For some reason, new PWM channels that get exported by writing to `/sys/class/pwm/.../export` don't show up as events in udev - which means there isn't a way to apply a more permissive policy to them.

This seems to be due to a difference in how [PWM](https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/drivers/pwm/sysfs.c) does device creation compared to [GPIO](https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/drivers/gpio/gpiolib-sysfs.c), since the latter does create events like this one:

    KERNEL[78.588317] add      /devices/platform/soc/3f200000.gpio/gpio/gpio0 (gpio)
    UDEV  [78.638944] add      /devices/platform/soc/3f200000.gpio/gpio/gpio0 (gpio)


## Race-free export of GPIO pins

Exporting new GPIO pins by writing to `/sys/class/gpio/export` creates new directories that initially get created with very restrictive permissions, before udev asynchronously applies a more permissive policy.

Right now Processing's IO library simply sleeps and hopes that udev gets around doing its thing. Is there perhaps a better way? (e.g. creating the directories with the same owner/group/permissions of the `export` file?)


## Race-free export of PWM channels

The same problem also applies to `/sys/class/pwm/.../export`.


## A way to go from PWM channel to GPIO pin

There's currently no way of retrieving the underlying GPIO pin(s) of a PWM channel in `/sys/class/pwm`, which would probably be very relevant information in many cases.