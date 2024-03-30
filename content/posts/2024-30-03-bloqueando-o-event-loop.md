---
author: [ "Rafael Becker" ]
title: Bloqueando o event loop
date: "2024-03-30"
tags: [ "nodejs" ]
---

Uma das características que mais me fascinam sobre o Node.js está em sua estrutura assíncrona não bloqueante que é
implementada através de seu **Event Loop**.

Sempre tive curiosidade em experimentar como a manipulação indevida dessa estrutura poderia impactar de maneira negativa
no desempenho de uma aplicação web.

Neste artigo estudo um cenário em que um servidor web disponibiliza dois endpoints, ambos com objetivo de gerar um
arquivo grande (1GB) porém um operando por meio de ações **síncronas** enquanto outro por ações **assíncronas**.

## Estrutura do servidor web

O servidor web foi construído utilizando o módulo `http` padrão do nodejs

```javascript
import http from "node:http"

/**
 * @param {http.ServerResponse} res
 * @param {number} status
 * @param {string} message
 */
const httpSendResponse = (res, status, message) => {
    res.writeHead(status)
    res.end(JSON.stringify({"msg": message}));
}

http.createServer((req, res) => {
    switch (req.url) {
        case "/status":
            return httpSendResponse(res, 200, "Healthy")
        case "/fs-stream":
            return fsWriteToStream(req, res)
        case "/fs-append":
            return fsAppendToFile(req, res)
        default:
            return httpSendResponse(res, 404, "Not Found")
    }
}).listen(3000, () => console.log("Up and Running!"))
```

Nesse foram disponibilizados alguns endpoints para auxiliar cenário:

* **/status**: retorna o valor fixo `{"msg":"Healthy"}` representando o estado da aplicação
* **/fs-append**: cria um arquivo no working directory utilizando operações síncronas.
* **/fs-stream**: cria um arquivo no working directory utilizando operações assíncronas.

## Operação síncrona

No método síncrono foi optado por gerar o arquivo primeiro por meio da alocação de buffer com tamanho de 1GB e
pelo preenchimento desse com o valor fixo `A`.

Após isso é **iterado** sobre o seu conteúdo para então **concatenar** cada um de seus elementos em um ponteiro de
arquivo.

```javascript
/**
 * @param {http.IncomingMessage} req
 * @param {http.ServerResponse} res
 */
const fsAppendToFile = (req, res) => {
        const fd = fs.openSync("buff.txt", 'w')
        const buff = Buffer.alloc(1 * Math.pow(10, 9)).fill("A")
        buff.forEach((v) => fs.appendFileSync(fd, v + ""))
        fs.close(fd)
        httpSendResponse(res, 200, "done")
    }
```

Ao realizar a chamada para o endpoint **/fs-append** é dado início ao processo de geração do arquivo de maneira
síncrona pelo serviço web.

```bash
$ curl -v http://localhost:3000/fs-append

*   Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 3000 (#0)
> GET /fs-append HTTP/1.1
> Host: localhost:3000
> User-Agent: curl/7.58.0
> Accept: */*
> 

```

Visto o custo da operação o cliente http acaba segurar o terminal, aguardando o retorno da chamada até que o arquivo
tenha sido gerado de maneira bem sucedida ou não.

Como a escrita do arquivo foi realizada de maneira síncrona, essa ficará restringindo a execução do event loop, sendo
que esse então somente poderá dar vez a outras ações quando a escrita do arquivo tiver terminado, visto sua
característica de **single thread**.

Esse comportamento pode ser confirmado a partir da chamada do endpoint **/status** enquanto a escrita do arquivo ainda
não tiver sido finalizada:

```bash
$ curl -v http://localhost:3000/status
*   Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 3000 (#0)
> GET /status HTTP/1.1
> Host: localhost:3000
> User-Agent: curl/7.58.0
> Accept: */*
> 
```

O cliente http usado para chamar o endpoint ficará travado esperando pelo retorno que não será obtido enquanto o arquivo
não tiver sido gerado pelo serviço web.

## Operação assíncrona

A simulação da escrita de um arquivo de maneira assíncrona foi implementada por meio de um for loop entre 0 e 1 bilhão
utilizando um **fs.WriteStream** concatenando o valor `A` em um arquivo a cada iteração.

```javascript
/**
 * @param {http.IncomingMessage} req
 * @param {http.ServerResponse} res
 */
const fsWriteToStream = (req, res) => {
        /**
         * @param {fs.WriteStream} writable
         * @returns {Promise<void>}
         */
        const write = async (writable) => {
            for (let i = 0; i < (1 * Math.pow(10, 9)); i++) {
                if (!writable.write("A")) {
                    await new Promise((resolve, reject) => writable.once("drain", resolve));
                }
            }
            writable.end()
        }
        write(fs.createWriteStream("stream.txt"))
            .then(() => httpSendResponse(res, 200, "done"))
            .catch(e => httpSendResponse(res, 500, "error"))
    }
```

O processo de escrita foi encapsulado em uma função **async** o que permitiu interromper o loop nos momentos em que não
fosse seguro continuar com a operação, retomando a iteração apenas a partir da emissão do evento **drain**.

Ao realizar a chamada ao endpoint **/fs-stream** é dado início ao processo de geração do arquivo de maneira assíncrona

```bash
curl -v http://localhost:3000/fs-stream
*   Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 3000 (#0)
> GET /fs-stream HTTP/1.1
> Host: localhost:3000
> User-Agent: curl/7.58.0
> Accept: */*
> 

```

Como a operação de escrita é realizada de maneira assíncrona a aplicação não aguarda por essa ser concluída, com isso
o controle da operação é devolvido ao event loop que pode continuar a processar outras tarefas.

Esse comportamento pode também ser confirmado pela chamada do endpoint **/status** enquanto a escrita do arquivo ainda
não tiver sido finalizada pelo processo assíncrono:

```bash
curl -v http://localhost:3000/status
*   Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 3000 (#0)
> GET /status HTTP/1.1
> Host: localhost:3000
> User-Agent: curl/7.58.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< Date: Sun, 31 Mar 2024 12:30:09 GMT
< Connection: keep-alive
< Keep-Alive: timeout=5
< Transfer-Encoding: chunked
< 
* Connection #0 to host localhost left intact
{"msg":"Healthy"}
```