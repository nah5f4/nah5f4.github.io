---
title: "Tutoriais 5 e 6 - Configurando Git Send-Email com OAuth2 (E-mail USP)"
date: 2026-04-12
categories: [Kernel, Git]
tags: [linux, patch, git-send-email, usp, flusp]
---

# Preparação para o envio de patches: Autenticação e Git

Nos tutoriais 5 e 6, o objetivo foi preparar meu ambiente para o envio de patches para o kernel Linux utilizando o Git.

## O Problema: E-mails da USP e o Gmail

Se eu utilizasse um e-mail pessoal, a configuração seria simples: bastaria criar uma "Senha de Aplicativo". Porém, as contas institucionais da USP, apesar de usarem a infraestrutura do Google, possuem a Verificação em Duas Etapas gerenciada pela universidade, o que impede a criação dessa senha. Além disso, o suporte a "Aplicativos Menos Seguros" foi descontinuado em 2025.

### A Solução: OAuth 2.0
A alternativa foi usar o protocolo OAuth 2.0. Ele delega a autenticação ao Google (via navegador) e gera um token de acesso, sem que o git send-email precise armazenar minha senha real.

---

## 1. Instalação e Configuração Global

Antes de lidar com a autenticação específica, precisamos garantir que o Git e o pacote de e-mail estejam instalados e configurados.

A minha máquina já tinha os pacotes adicionais instalados, como também o meu nome e e-mail configurados, então não precisei repetir este passo:

# Instalação do pacote de envio (Ubuntu/Debian)
$ sudo apt-get install git-email

# Configuração de identidade
$ git config --global user.name "Naili Lucia Marques"
$ git config --global user.email "naahmarque345@usp.br"

### Configuração do servidor SMTP de envio de e-mail (Gmail Pessoal):

Para o Gmail comum, a configuração seria esta:

$ git config --global sendemail.smtpencryption tls
$ git config --global sendemail.smtpserver "smtp.gmail.com"
$ git config --global sendemail.smtpuser "naahmarque345@gmail.com"
$ git config --global sendemail.smtpserverport "587"

Eu cheguei a criar uma senha de aplicativo no Google para esse cenário, o que foi bem simples. No entanto, não cheguei a testar o envio por aqui porque percebi que este não era o tutorial que precisava ser feito para a matéria.

---

## 2. Configurando o Git Credential Gmail (E-mail USP)

Para automatizar o OAuth com a conta institucional, utilizei o auxiliar git-credential-gmail. Ele lida com a geração e renovação dos tokens de acesso de forma transparente para o Git.

Primeiro, precisei obter um ID de cliente. No terminal, executei o comando abaixo e selecionei a opção correspondente aos tokens que eu iria utilizar:

$ git credential-gmail --set-client

Quando solicitado pela Redirect URI, preenchi com: http://localhost

Em seguida, gerei o token de acesso que serve como minha "senha":

$ git credential-gmail --authenticate --external-auth

O processo seguiu estes passos:
1. O terminal exibiu uma URL. Copiei e colei no meu navegador.
2. Fiz login com minha conta USP e autorizei o acesso ao e-mail.
3. Fui redirecionada para uma página que dizia "Não foi possível conectar". Isso é ESPERADO.
4. Copiei a URL completa dessa página de erro (que começa com http://localhost/?code=...) e colei de volta no terminal.

Para verificar se o token foi salvo corretamente no meu chaveiro do sistema, digitei:
$ git credential-gmail

---

## 3. Configuração Local do Repositório

Com o token configurado, precisei dizer ao Git dentro do meu repositório de trabalho do Kernel para usar esse auxiliar e o protocolo de autenticação correto (OAUTHBEARER).

Acessei as configurações locais do repositório:
$ git config --local --edit

Adicionei as seguintes seções para que ficassem desta forma:

[credential "smtp://smtp.gmail.com:465"]
    helper = gmail

[sendemail]
    smtpEncryption = ssl
    smtpServer = smtp.gmail.com
    smtpUser = naahmarque345@usp.br
    smtpServerPort = 465
    smtpAuth = OAUTHBEARER
    from = Naili Lucia Marques <naahmarque345@usp.br>

---

## 4. Testando o envio com kw send-patch

Para garantir que a comunicação estava funcionando sem disparar e-mails reais para os mantenedores, utilizei o kworkflow (kw) com a flag de simulação.

### Criando um commit de teste
$ touch naili-Lucia.txt
$ git add naili-Lucia.txt

### Simulando o envio
Utilizei o kw para simular o envio do último commit para meu próprio e-mail e o dos monitores:

$ kw send-patch --send -1 --private --simulate --to='naahmarque345@usp.br', 'dsl26oauthproxy@gmail.com'

Nesta etapa, verifiquei se o editor abria com o conteúdo do e-mail e se os campos From: e To: estavam corretos. Ao fechar o editor, o kw realizou o processo de autenticação sem enviar o e-mail de fato.

Durante o processo, me deparei com o seguinte erro:

/tmp/uTj8sk2rYF/0001-Naili-Lucia-Marques.patch
No SASL mechanism found
 at /usr/share/perl5/Authen/SASL.pm line 76.
 at /usr/share/perl/5.38/Net/SMTP.pm line 211

Consegui encontrar a solução no Stack Overflow. O problema era que o módulo Perl Authen::SASL do meu sistema não suportava o OAuth (XOAUTH2), então tive que baixar uma versão nova. Executei os seguintes comandos:

# Instalei o módulo novo com o cpan
$ cpan -i Authen::SASL

# Descobri onde ele estava instalado
$ perl -MAuthen::SASL -e 'print $INC{"Authen/SASL.pm"}'

# Configurei a variável de ambiente no meu shell
$ export PERL5LIB=$HOME/perl5/lib/perl5

Quando rodei novamente o comando de envio real (sem o --simulate):
$ kw send-patch --send -1 --private --to='naahmarque345@usp.br','dsl26oauthproxy@gmail.com'

Finalmente tive sucesso e a mensagem foi enviada corretamente!

---

A configuração do OAuth 2.0 é o caminho mais simples para quem usa e-mail da USP. Embora exija passos extras no navegador e, por vezes, correções manuais em módulos do sistema como o Perl, ela resolve permanentemente os problemas de bloqueio do Google e garante que meus patches cheguem à lista de discussão do Kernel de forma íntegra.
