# Owasp Coraza Haproxy
[![Code Linting](https://github.com/corazawaf/coraza-spoa/actions/workflows/lint.yaml/badge.svg)](https://github.com/corazawaf/coraza-spoa/actions/workflows/lint.yaml)
[![CodeQL Scanning](https://github.com/corazawaf/coraza-spoa/actions/workflows/codeql.yaml/badge.svg)](https://github.com/corazawaf/coraza-spoa/actions/workflows/codeql.yaml)

## Overview
This is a third-party daemon that connects to SPOE. It sends the request and response sent by HAProxy to [OWASP Coraza](https://github.com/corazawaf/coraza) and returns the verdict.

## Compilation
### Build
The command `make` will compile the source code and produce the executable file `coraza-spoa`.

### Clean
When you need to re-compile the source code, you can use the command `make clean` to clean the executable file.

## Start the service
After you have compiled it, you can start the service by running the command `./coraza-spoa`.
```shell
$> ./coraza-spoa -h
Usage of ./coraza-spoa:
  -config-file string
        The configuration file of the coraza-spoa. (default "./config.yml")
```

## Configure the service
The example configuration file is `config.yml.default`, you can copy it and modify the related configuration information.

## Configure a SPOE to use the service
Here is the configuration template to use for your SPOE with OWASP Coraza module, you can find it in the "doc/config/coraza.cfg":
```editorconfig
[coraza]
spoe-agent coraza-agent
    messages coraza-req coraza-res
    option var-prefix coraza
    timeout hello 100ms
    timeout idle 2m
    timeout processing 15ms
    use-backend coraza-spoa
    log global

spoe-message coraza-req
    args unique-id src method path query req.ver req.hdrs req.body
    event on-frontend-http-request

spoe-message coraza-res
    args unique-id status res.ver res.hdrs res.body
    event on-http-response
```

The engine is in the scope "coraza". So to enable it, you must set the following line in a frontend/listener section:
``` editorconfig
frontend coraza.io
    ...
    unique-id-format %[uuid()]
    unique-id-header X-Unique-ID
    filter spoe engine coraza config coraza.cfg
    ...
```

Because, in SPOE configuration file, we declare to use the backend "coraza-spoa"(of course, you can change it to any other backend) to communicate with the service, so we need to define it in the HAProxy file. For example:
```editorconfig
backend coraza-spoa
    mod tcp
    balance roundrobin
    timeout connect 5000ms
    timeout client 5000ms
    timeout server 5000ms
    server s1 127.0.0.1:9000
```

The OWASP Coraza action is returned in a variable named "txn.coraza.fail". It contains the verdict of the request. If the variable is set to 1, the request will be denied.
```editorconfig
http-request deny if { var(txn.coraza.fail) -m int eq 1 }
http-response deny if { var(txn.coraza.fail) -m int eq 1 }
```
With this rule, all the requests not clean are rejected. You can find the example configuration file in the "doc/config/haproxy.cfg".

