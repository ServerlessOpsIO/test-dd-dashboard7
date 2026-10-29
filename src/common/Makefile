build-CommonLayer:
	python3.14 ./setup.py build
	pip3.14 wheel -w tmp -e .
	mkdir $(ARTIFACTS_DIR)/python
	pip3.14 install tmp/common-* --target $(ARTIFACTS_DIR)/python/

