---
title: Renovando chaves GPG
date: 2024-03-28
---

As chaves GPG (Gnu Privacy Guard Keys) são utilizadas para verificação de autenticidade na comunicação de dados por meio
da criação de assinaturas digitais.

Um caso de uso comum onde as chaves GPG são aplicadas está no contexto de assinatura de commits, sendo tão comum que até 
Github possibilita a [configuração de chaves GPG][link-github-gpg] nas configurações do usuário.

Quando uma chave GPG expira é necessário realizar um processo de revalidação visando extender a validade para uma nova 
data. 

* O primeiro passo para a renovação está em relacionar a chave a ser renovada

```bash
$ gpg --list-keys
```

```text
/home/nuvem/.gnupg/pubring.kbx
------------------------------
pub   rsa2048 2023-11-07 [SCEA] [expira: 2024-05-01]
A248D439120EB1BCAAB5C4D1764EF304E823F236
uid           [final] rafael.becker <rafael.becker@magazord.com.br>
sub   rsa2048 2023-11-07 [SEA] [expira: 2024-05-01]
```

O identificador da chave em questão está localizado abaixo da linha que inicia com `pub`, sendo no exemplo anterior a
constante iniciada com `A248D`.

* Com o identificador da chave em mãos é possível realizar a extensão de sua data de expiração para, por exemplo, um ano a
partir de hoje.

```bash
# mantém chave para reuso posterior
$ export KEY1=A248D439120EB1BCAAB5C4D1764EF304E823F236
$ gpg --quick-set-expire $KEY1 1y
```

De maneira alternativa é ainda possível especificar a data em formato ISO `YYYY-MM-DD` ou manter a sua representação de
maneira relativa considerando dias `d`, semanas `w` ou meses `m`.

A instrução acima realiza a extensão da data de expiração apenas da chave principal. As chaves secundárias devem ser uma
a uma revistas de modo a também terem a sua data de expiração estendida.

* Semelhante ao caso anterior, primeiro é necessário obter o identificador da chave secundária

```bash
$ pgp --list-keys --with-subkey-fingerprints
```

```text
/home/nuvem/.gnupg/pubring.kbx
------------------------------
pub   rsa2048 2023-10-08 [SCEA] [expira: 2024-05-02]
      7B54A102ABDADDD89DAA01D9307B01241C6326FB
uid           [final] realfabecker <realfabecker@outlook.com>
sub   rsa2048 2023-10-08 [SEA] [expira: 2024-05-02]
      90DF38381B9B334A606E027964922E8364611660
```

O identificador da chave secundária em questão fica localizado abaixo da linha que inicia com `sub`, sendo no exemplo a
linha iniciada com `90DF`.

* A extensão do período de expiração pode ser realizado de maneira semelhante à chave principal

```bash
# mantém chave(s) para reuso posterior
$ export KEY1=A248D439120EB1BCAAB5C4D1764EF304E823F236
$ export SUB1=90DF38381B9B334A606E027964922E8364611660

$ gpg --quick-set-expire $KEY1 1y $SUB1
```



[link-github-gpg]: https://docs.github.com/en/authentication/managing-commit-signature-verification/adding-a-gpg-key-to-your-github-account