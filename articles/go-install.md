<!---
markmeta_author: wongoo
markmeta_date: 2019-01-16
markmeta_title: Go Installation
markmeta_categories: 编程语言
markmeta_tags: golang,installation
-->

# go installation

Mac:
```bash
curl -C - -O https://dl.google.com/go/go1.14.3.darwin-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.14.3.darwin-amd64.tar.gz
```

Linux:
```bash
curl -C - -O https://dl.google.com/go/go1.14.3.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.14.3.linux-amd64.tar.gz
```

config env:
```bash
> sudo vi /etc/profile
export BASEDIR=/Users/gelnyang
export GOPATH=$BASEDIR/go
export GOBIN=$GOPATH/bin
export PATH=$PATH:/usr/local/go/bin:$GOBIN
export GO111MODULE=on

export GOPROXY=https://goproxy.io
```

upgrade: 
```bash
sudo rm -rf /usr/local/go
# then install the latest
```

Install from source:
```bash
curl -C - -O https://dl.google.com/go/go1.14.3.src.tar.gz
tar -C /usr/local -xzf go1.14.3.src.tar.gz
cd /usr/local/go/src
time sudo ./make.bash
```


## go tools

```bash
# golang tools
cd $GOPATH/src/github.com/golang/tools && git pull -v && go install ./...
# go get -u golang.org/x/tools/...


# golangci-lint
cd $GOPATH/src/github.com/golangci/golangci-lint && git pull -v && go install ./...
# go get -u github.com/golangci/golangci-lint/...
```

