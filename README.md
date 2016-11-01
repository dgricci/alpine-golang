% Environnement GoLang
% Didier Richard
% rév. 0.0.1 du 27/19/2017

---

# Building #

```bash
$ docker build -t dgricci/alpine-golang:0.0.1 -t dgricci/alpine-golang:latest .
```

## Behind a proxy (e.g. 10.0.4.2:3128) ##

```bash
$ docker build \
    --build-arg http_proxy=http://10.0.4.2:3128/ \
    --build-arg https_proxy=http://10.0.4.2:3128/ \
    -t dgricci/alpine-golang:0.0.1 -t dgricci/alpine-golang:latest .
```

## Build command with arguments default values ##

```bash
$ docker build \
    --build-arg GOLANG_VERSION=1.7.3 \
    --build-arg GOLANG_DOWNLOAD_URL=https://golang.org/dl/go$GOLANG_VERSION.linux-amd64.tar.gz \
    --build-arg GOLANG_DOWNLOAD_SHA256=508028aac0654e993564b6e2014bf2d4a9751e3b286661b0b0040046cf18028e \
    -t dgricci/alpine-golang:0.0.1 -t dgricci/alpine-golang:latest .
```

# Use #

See `dgricci/alpine` README for handling permissions with dockers volumes.

```bash
$ cd src/go
$ tree .
.
├── bin
├── pkg
└── src
    └── pamplemousse
        └── html2csv.go

4 directories, 1 files
$ docker run -it --rm -v `pwd`:/go -e USER_ID=`id -u` dgricci/alpine-golang /bin/bash
bash-4.3$ go get github.com/PuerkitoBio/goquery
bash-4.3$ cd src/pamplemousse
bash-4.3$ go build html2csv.go
bash-4.3$ ls -1
html2csv
html2csv.go
bash-4.3$ exit
exit
$ src/pamplemousse/html2csv 

Usage: src/pamplemousse/html2csv URL
For example :
src/pamplemousse/html2csv http://localhost/~ricci/Pamplemousse-TSI-2015-2016.html
```

Let's suppose that the env variable GOPATH points at the current directory :

```bash
$ pwd
/home/ricci/src/go
$ echo $GOPATH
/home/ricci/src/go
$ cd src/remontees
$ docker run --rm -v ${GOPATH}:/go -w/go${PWD##${GOPATH}} -e USER_ID=`id -u` -e USER_NAME=`whoami` dgricci/alpine-golang go build kml2mongo.go
$ ls -lrt kml2mongo*
-rw-rw-r-- 1 ricci ricci    2165 août  18 13:45 kml2mongo.go
-rwxr-xr-x 1 ricci ricci 6124696 août  19 12:06 kml2mongo
```

# A shell to hide container's usage #

```bash
#!/bin/bash
#
# Exécute le container docker dgricci/alpine-golang
#
# Constantes :
VERSION="0.9.0"
# Variables globales :
readonly -A commands=(
[go]=""
[godoc]=""
[gofmt]=""
)
#
theShell="$(basename $0)"
theShell="${theShell%.sh}"
#
unset show
unset noMoreOptions
#
# Exécute ou affiche une commande
# $1 : code de sortie en erreur
# $2 : commande à exécuter
run () {
    local code=$1
    local cmd=$2
    if [ -n "${show}" ] ; then
        echo "cmd: ${cmd}"
    else
        eval ${cmd}
    fi
    # go|godoc|gofmt --help returns 2 ...
    [ ${code} -ge 0 -a $? -ne 0 ] && {
        echo "Oops #################"
        exit ${code#-} #absolute value of code
    }
    [ ${code} -ge 0 ] && {
        return 0
    }
}
#
# Affichage d'erreur
# $1 : code de sortie
# $@ : message
echoerr () {
    local code=$1
    shift
    echo "$@" 1>&2
    usage ${code}
}
#
# Usage du shell :
# $1 : code de sortie
usage () {
    cat >&2 <<EOF
usage: `basename $0` [--help -h] | [--show|-s] commandAndArguments

    --help, -h          : prints this help and exits
    --show, -s          : do not execute $theShell, just show the command to be executed

    commandAndArguments : arguments and/or options to be handed over to ${theShell}.
                          The directory where this script is lauched is a
                          sub-directory of GOPATH.

    The GOPATH environment variable must be set to the directory containing
    the alpine-golang sources, binaries and packages (aka alpine-golang projects !)
EOF
    exit $1
}
#
# main
#
[ -z "${GOPATH}" ] && {
    echoerr 2 "Missing environment variable GOPATH"
}
# remove the GOPATH prefix ...
w="${PWD##${GOPATH}}"
[ "${PWD}" = "${w}" ] && {
    echoerr 3 "The current directory is not a sub-directory of ${GOPATH}"
}
cmdToExec="docker run -e USER_ID=${UID} -e USER_NAME=${USER} --name=\"go$$\" --rm=true -v${GOPATH}:/go -w/go${w} dgricci/alpine-golang $theShell"
while [ $# -gt 0 ]; do
    # protect back argument containing IFS characters ...
    arg="$1"
    [ $(echo -n ";$arg;" | tr "$IFS" "_") != ";$arg;" ] && {
        arg="\"$arg\""
    }
    if [ -n "${noMoreOptions}" ] ; then
        cmdToExec="${cmdToExec} $arg"
    else
        case $arg in
        --help|-h)
            run -1 "${cmdToExec} --help"
            usage 0
            ;;
        --show|-s)
            show=true
            noMoreOptions=true
            ;;
        --)
            noMoreOptions=true
            ;;
        *)
            [ -z "${noMoreOptions}" ] && {
                noMoreOptions=true
            }
            cmdToExec="${cmdToExec} $arg"
            ;;
        esac
    fi
    shift
done

run 100 "${cmdToExec}"

exit 0
```

__Et voilà !__


_fin du document[^pandoc_gen]_

[^pandoc_gen]: document généré via $ `pandoc -V fontsize=10pt -V geometry:"top=2cm, bottom=2cm, left=1cm, right=1cm" -s -N --toc -o golang.pdf README.md`{.bash}

