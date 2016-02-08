
.PHONY: install

install:
	cat *.env | sudo fw_setenv -s -
	(cd ../../macro; make install; make clean)
	sudo cp uEnv.txt /boot/conf.d/default/uEnv.txt.default
