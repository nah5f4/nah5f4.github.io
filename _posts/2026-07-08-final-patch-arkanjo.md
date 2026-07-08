---
layout: post
title: "Contribuição para o ArKanjo: Desbravando a Arquitetura de uma CLI Otimizada"
date: 2026-07-07 22:30:00 -0300
categories: [software-libre, contribuicoes]
tags: [cpp, open-source, arkanjo, cli, jekyll]
authors: ["Naili Lucia Marques", "Giovanna Luisa Hirata dos Anjos"]
---

Para a segunda fase da disciplina de Software Livre, escolhemos contribuir para o **ArKanjo**, uma ferramenta de linha de comando (CLI) super interessante que possui um pipeline orquestrado para processar bases de código e analisar similaridades e duplicações entre trechos de código.

---

Decidimos trabalhar na issue **feat(cli): add cache location command (#25)**, cujo objetivo era adicionar um comando para que o usuário pudesse descobrir onde o ArKanjo armazena seus arquivos de cache (por padrão, no diretório `~/.cache/arkanjo`). 

Essa funcionalidade é extremamente útil para fins de manutenção manual, inspeção de conteúdos cacheados e resolução de problemas técnicos (troubleshooting). 

O comportamento esperado para a CLI era o seguinte:
```bash
$ arkanjo preprocessor path
# Output esperado:
/home/user/.cache/arkanjo
```

---

Em nossa primeira abordagem, implementamos a interceptação do comando diretamente no orquestrador global do projeto (`src/main.cpp` e `src/orchestrator_commands.hpp`). Alteramos o mapeamento do subcomando `preprocessor`, que originalmente retornava `nullptr`, para instanciar diretamente a nossa nova classe `PreprocessorPath`.

O **Guilherme Ivo** nos explicou que a CLI do ArKanjo foi projetada seguindo um padrão arquitetural modular bem definido e que deveríamos ajustar as nossas mudanças para seguir a arquitetura:

1. O orquestrador global mantém o comando `preprocessor` registrado como `nullptr` propositalmente. Isso sinaliza para o ArKanjo que ele deve delegar a execução de forma automática para um binário standalone dedicado chamado `arkanjo-preprocessor`.
2. Tratar os subcomandos com condicionais diretas no ponto de entrada global dificulta a evolução do código e quebra o princípio de responsabilidade única. Se futuramente quisermos adicionar flags como `--open`, o arquivo principal ficaria inflado e complexo.

---

Seguindo as orientações de arquitetura do projeto, mantivemos o orquestrador global intocado (preservando o fluxo original de delegação via `nullptr`) e isolamos toda a lógica dentro do subsistema do pré-processador.

#### 1. Criação do (`src/commands/pre/path/preprocessor_path.hpp`)
Definimos a classe herdando da base de comandos do projeto, documentando adequadamente o seu propósito através de comentários Doxygen:

```cpp
/**
 * @file preprocessor_path.hpp
 * @brief Interface for displaying the cache directory path
 * 
 * Defines the PreprocessorPath command that outputs the absolute path
 * of the ArKanjo cache directory. This is useful for manual maintenance,
 * inspecting cache contents, and troubleshooting.
 */

 #pragma once

 #include <arkanjo/commands/command_base.hpp>
 #include <iostream>

 /**
  * @brief Displays the location of the ArKanjo cache directory
  * 
  * Handles the execution of the 'path' command, which retrieves and
  * cleanly prints the base cache path configuration to standard output.
  */
class PreprocessorPath : public CommandBase<PreprocessorPath> {
public:
    PreprocessorPath();

    COMMAND_DESCRIPTION("Display the location of the ArKanjo cache directory.")

    bool validate(const ParsedOptions& options) override;

    bool run(const ParsedOptions& options) override;
};
```

#### 2. Implementação da Lógica (`src/commands/pre/path/preprocessor_path.cpp`)
Buscamos o caminho absoluto configurado de maneira limpa através da API interna do próprio ArKanjo (`Config::config().base_path`):

```cpp
#include "preprocessor_path.hpp"
#include <arkanjo/base/config.hpp>

PreprocessorPath::PreprocessorPath() { }

bool PreprocessorPath::validate([[maybe_unused]] const ParsedOptions& options) {
    return true;
}

bool PreprocessorPath::run([[maybe_unused]] const ParsedOptions& options) {
    std::cout << Config::config().base_path.string() << "\n";
    return true;
}
```

#### 3. Registro no Menu de Comandos Internos
Por fim, registramos a nova funcionalidade na tabela de comandos em `src/commands/pre/preprocessor_main.cpp`. 

**Boa prática de Git aplicada:** O Guilherme nos lembrou de incluir uma *trailing comma* (vírgula à direita) logo após a nossa nova entrada. Isso garante que, quando um desenvolvedor adicionar um comando futuro, a linha que criamos não precise ser modificada apenas para ganhar uma vírgula, mantendo o histórico do `git blame` perfeitamente limpo e associado à nossa contribuição original.

---

Após realizarmos os ajustes arquiteturais com um commit explicando as motivações da issue, o nosso Pull Request foi aprovado e integrado com sucesso à branch principal via *squash*! 

