obj-m:= sample_module.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(shell pwd)/ module

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(shell pwd)/ clean
