---
title: "Meu Primeiro Patch: Refatoração com Guard Mutex no Subsistema IIO"
date: 2026-04-22
categories: [Kernel, IIO]
tags: [linux, kernel, patch, software-livre, mutex]
---

# Primeiro patch:

Como parte da disciplina de Software Livre, eu e minha dupla, Giovanna Hirata, enviamos nossa primeira contribuição oficial para o Kernel Linux. O objetivo foi modernizar o código utilizando primitivas de sincronização mais seguras.

## Modernizando o Mutex

Nós optamos por seguir uma sugestão de refatoração que tem sido bem aceita na comunidade: substituir o par manual mutex_lock() e mutex_unlock() pela macro guard(mutex)().

Como dito em aula, o uso manual de locks é propenso a erros. É muito fácil um desenvolvedor esquecer de liberar o mutex em um caminho de erro (return -ERROR), causando um "deadlock". 

A macro guard(mutex)(&lock) utiliza o atributo cleanup do GCC. Isso significa que o mutex é liberado automaticamente assim que a execução sai do escopo (do bloco { }) onde foi declarado. Isso limpa o código, remove a necessidade de múltiplos unlocks antes de cada return e evita vazamentos de trava.



---

## Como foi feito:

Após clonar o repositório e realizar algumas buscas (grep) por candidatos ideais, encontramos um arquivo perfeito para iniciantes no subsistema IIO:
drivers/iio/common/ms_sensors/ms_sensors_i2c.c.

Fizemos a substituição em trechos onde a lógica de saída da função era repetitiva, garantindo que o código ficasse mais legível e robusto.

---

## O Envio e o feedback 

Após passar pelos testes da pipeline de CI (Integração Contínua), enviamos o patch para os mantenedores.

 O feedback veio do Andy Shevchenko, recebemos um pedido de "slow down" (ir mais devagar). Ficamos confusas no início, pois era nossa primeira contribuição, mas logo entendemos que os mantenedores lidam com centenas de patches similares e acabam se confundindo. Ele aconselhou revisar outros patches aceitos sobre o mesmo tema antes de enviar o nosso.

## Próximos Passos

Respondemos ao Andy esclarecendo que este era o nosso primeiro patch e agradecendo as orientações. Agora, estamos aguardando o "sinal verde" para enviar uma versão 2 (V2) do patch, incluindo as melhorias na organização dos cabeçalhos.
