BINARY := blockchain

all: build test

build: deps
	@echo "====> Go build"
	@go build -o $(BINARY)

deps:
	@go get -u github.com/boltdb/bolt

test:
	./$(BINARY) printchain
	./$(BINARY) addblock -data "Send 1 BTC to Ivan"
	./$(BINARY) addblock -data "Pay 0.31337 BTC for a coffee"
	./$(BINARY) printchain

.PHONY: build deps test
