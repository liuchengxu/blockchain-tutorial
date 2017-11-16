BINARY := blockchain

all: build test

build: deps
	@echo "====> Go build"
	@go build -o $(BINARY)

deps:
	@go get -u github.com/boltdb/bolt

test:
	./$(BINARY) printchain
	./$(BINARY) getbalance -address Ivan
	./$(BINARY) printchain

.PHONY: build deps test
