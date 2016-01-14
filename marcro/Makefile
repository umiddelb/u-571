
.PHONY: clean all install

all: $(patsubst %.uboot,%.sh,$(wildcard *.uboot))

install: all
	cat *.sh | sudo sh

clean:
	rm -f *.sh

%.sh: %.uboot
	(echo -n "fw_setenv $* '" ; cat $< | grep -v '^#' | tr -d '\n' | tr -s ' ' ; echo "'" ) > $@	
