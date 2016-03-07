
.PHONY: install clean

install:
	(cat board_init.env | tr -s '\t' | tr '\t' '=' > board_init.txt)
	(cd ../../macro; make uEnv.ext4.txt; cat uEnv.ext4.txt >> ../board/pine64_plus/board_init.txt; make clean)
	sudo cp uEnv.txt /boot/conf.d/default/uEnv.txt

clean:
	rm -f board_init.txt 
