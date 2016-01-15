
.PHONY: clean all install

all: $(patsubst %.uboot,%.uEnv,$(wildcard *.uboot))

install: all
	cat *.uEnv | sudo fw_setenv -s -

clean:
	rm -f *.sh *.uEnv

%.sh: %.uboot
	(echo -n "fw_setenv $* '" ; cat $< | grep -v '^#' | tr -d '\n' | tr -s ' ' ; echo "'" ) > $@	

%.uEnv: %.uboot
	(echo -n "$*\t\t" ; cat $< | grep -v '^#' | tr -d '\n' | tr -s ' ' ; echo ) > $@	
