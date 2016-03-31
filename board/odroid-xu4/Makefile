
.PHONY: all clean install

all: bundle.uEnv

clean:
	rm -f bundle.uEnv

bundle.uEnv: board_init.env
	cat $< | tr -s '\t' | tr '\t' '=' > $@
	(cd ../../macro; make macro.uEnv)
	cat ../../macro/macro.uEnv >> $@
	(cd ../../macro; make clean)

install:
	cat *.env | sudo fw_setenv -s -
	(cd ../../macro; make install; make clean)
