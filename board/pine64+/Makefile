
.PHONY: all clean install

all: bundle.uEnv

clean:
	rm -f bundle.uEnv

bundle.uEnv: board_init.env
	cat $< | tr -s '\t' | tr '\t' '=' > $@
	(cd ../../macro; make macro.ext4.uEnv)
	cat ../../macro/macro.ext4.uEnv >> $@
	(cd ../../macro; make clean)

install: bundle.uEnv
	cp bundle.uEnv /boot/bundle.uEnv 
