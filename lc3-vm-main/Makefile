all: vm.out

CC = gcc
CFLAGS = -Wall --std=c11

vm.out: vm.c vm_dbg.c
	$(CC) -o vm.out vm.c vm_dbg.c
clean:
	rm -f *.o
