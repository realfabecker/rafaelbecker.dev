---
author: [ "Rafael Becker" ]
title: Renovando chaves gpg
date: "2024-03-28"
tags: [ "gpg" ]
---

O uso de chaves GPG por plataformas de hospedagem git é um recurso que vem como um método adicional para a verificação
de integridade e autenticidade do autor de um commit para um repositório.

Uma chave GPG tem como característica possuir uma data de expiração pré-definida prevenindo assim a exploração de chaves
que possam ser expostas.

Porém, isso significa que nós, como usuários, precisamos ficar atentos e manter nossas chaves atualizadas para não
ocorrerem problemas na hora fazer os nossos commits.

* O primeiro passo no processo de renovação está em obter o fingerprint da chave a ser renovada

```bash
$ gpg --list-keys
```

```text
/home/nuvem/.gnupg/pubring.kbx
------------------------------
pub   rsa2048 2023-10-08 [SCEA] [expira: 2023-11-08]
      7B54A102ABDADDD89DAA01D9307B01241C6326FB
uid           [final] realfabecker <realfabecker@outlook.com>
sub   rsa2048 2023-10-08 [SEA] [expira: 2023-11-08]
```

O identificador da chave em questão está localizado abaixo da linha que inicia com `pub` com o valor constante com o
valor `7B54A102ABDADDD89DAA01D9307B01241C6326FB`.

* Com o identificador da chave em mãos é possível realizar a extensão de sua data de expiração para, por exemplo, cinco
  semanas a partir da data de hoje.

```bash
$ gpg --quick-set-expire 7B54A102ABDADDD89DAA01D9307B01241C6326FB 5w
```

De maneira alternativa é ainda possível especificar a data em formato ISO `YYYY-MM-DD` ou manter a sua representação de
maneira relativa considerando anos `y`, dias `d` ou meses `m`.

Chaves secundárias também podem ter a sua data de expiração estendida, porém é necessário especificar o seu fingerprint
ou um wildcard `*` para considerar todas as chaves secundárias associadas a chave principal.

```bash
$ gpg --quick-set-expire 7B54A102ABDADDD89DAA01D9307B01241C6326FB 5w '*'
```

O `--quick-set-expire` considera apenas sub-chaves não expiradas ao extender a expiração da chave principal. Para editar
sub-chaves já expiradas é necessário realizar a extensão de maneira manual.

* Primeiro deve-se iniciar a interface de edição a partir do fingerprint da chave principal

```bash
$ gpg --edit-key 7B54A102ABDADDD89DAA01D9307B01241C6326FB
```

```text
gpg (GnuPG) 2.2.4; Copyright (C) 2017 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Chave secreta disponível.

sec  rsa2048/307B01241C6326FB
     criado: 2023-10-08  expira: 2024-05-02  uso: SCEA
     confiança: final         validade: final
ssb  rsa2048/64922E8364611660
     criado: 2023-10-08  expira: 2024-05-02  uso: SEA 
[final] (1). realfabecker <realfabecker@outlook.com>

gpg> 
```

* Em seguido selecionar a sub-chave a ser editada, prefixada com `ssb`

```text
gpg> key 1

sec  rsa2048/307B01241C6326FB
     criado: 2023-10-08  expira: 2024-05-02  uso: SCEA
     confiança: final         validade: final
ssb* rsa2048/64922E8364611660
     criado: 2023-10-08  expira: 2024-05-02  uso: SEA 
[final] (1). realfabecker <realfabecker@outlook.com>

gpg>
```

* O processo de renovação é iniciado com `expire` e confirmado com `save`

```text
gpg> expire

Modificando a data de validade para uma subchave.
Por favor especifique por quanto tempo a chave deve ser válida.
         0 = chave não expira
      <n>  = chave expira em n dias
      <n>w = chave expira em n semanas
      <n>m = chave expira em n meses
      <n>y = chave expira em n anos
A chave é valida por? (0) 5w
A chave expira em sex 03 mai 2024 16:26:02 -03
Está correto (s/N)? s

sec  rsa2048/307B01241C6326FB
     criado: 2023-10-08  expira: 2024-05-02  uso: SCEA
     confiança: final         validade: final
ssb* rsa2048/64922E8364611660
     criado: 2023-10-08  expira: 2024-05-03  uso: SEA 
[final] (1). realfabecker <realfabecker@outlook.com>

gpg> save
```
