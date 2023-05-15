obj-m:= sample_module.o
all:
	make -C /lib/modules/$(shell uname -r)/build M=$(shell pwd)/sample_module modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(shell pwd)/sample_module modules
    
