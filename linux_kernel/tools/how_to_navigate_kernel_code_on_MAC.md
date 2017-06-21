# How to navigate linux kernel code on a MAC

## Env

 * Macbookpro with Retina (2015 mid 15" )
 * OSX 10.11.3 Darwin Kernel Version 15.3.0
 * Eclipse Mars.2
 * Kernel 4.11.2

## 1. Generate `autoconf.h`

On a linux system, do the following steps
> https://serverfault.com/questions/568395/what-is-creating-the-generated-autoconf-h

## 2. Copy the folder `include/generated` to Mac

At the end of 1st step, you will get a `include/generated` folder on your linux system's kernel src folder.

This folder needs to be copied to your Mac's linux kernel src folder.

## 3. Config Eclipse

Please follow these steps:
> https://wiki.eclipse.org/HowTo_use_the_CDT_to_navigate_Linux_kernel_source