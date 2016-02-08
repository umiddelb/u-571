
.PHONY: clean all install

all: $(patsubst %.uboot,%.env,$(wildcard *.uboot))

install: all
	cat *.env | sudo fw_setenv -s -

install.ext4: all
	cat *.env | sed -e 's/ext2/ext4/g' | sudo fw_setenv -s -

uEnv.txt: $(patsubst %.uboot,%.uEnv,$(wildcard *.uboot))
	cat *.uEnv | sort > $@

clean:
	rm -f *.sh *.env *.uEnv uEnv.txt

%.sh: %.uboot
	(echo -n "fw_setenv $* '" ; cat $< | grep -v '^#' | tr '\n' ' ' | tr -s ' ' ; echo "'" ) > $@	

%.env: %.uboot
	(echo -n "$*\t\t" ; cat $< | grep -v '^#' | tr '\n' ' ' | tr -s ' ' ; echo ) > $@	

%.uEnv: %.uboot
	(echo -n "$*=" ; cat $< | grep -v '^#' | tr '\n' ' '| tr -s ' ' ; echo ) > $@	
