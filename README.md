# Trabalho de Segurança da Informação - Buffer Overflow

> Projeto desenvolvido e finalizado em Julho de 2026 para a matéria de Segurança da Informação.

Este repositório contém a resolução do **segundo** trabalho prático da disciplina de Segurança da Informação, do oitavo período do curso de Sistemas de Informação da Universidade Federal de Uberlândia (UFU). Desse modo, o exercício foi proposto pelo professor através de um código em linguagem C, seguido de 4 tarefas a serem realizadas com base nele.

O objetivo do trabalho é analisar uma vulnerabilidade de **buffer overflow na pilha**, explorar o código fornecido pelo professor e observar como diferentes mecanismos de proteção interferem no ataque.

## Tecnologias utilizadas

![C](https://img.shields.io/badge/C-Linguagem-00599C?style=flat&logo=c&logoColor=white)
![GCC](https://img.shields.io/badge/GCC-Compilador-A42E2B?style=flat&logo=gnu&logoColor=white)
![GDB](https://img.shields.io/badge/GDB-Debugger-5C6BC0?style=flat)
![objdump](https://img.shields.io/badge/objdump-GNU_Binutils-4EAA25?style=flat)
![checksec](https://img.shields.io/badge/checksec-Analise_de_protecoes-333333?style=flat)
![Ubuntu](https://img.shields.io/badge/Ubuntu-Linux-E95420?style=flat&logo=ubuntu&logoColor=white)

As ferramentas utilizadas foram:

- `gcc`, para compilar as diferentes versões do programa;
- `gdb`, para analisar a execução, os registradores e a pilha;
- `objdump`, para visualizar o assembly do executável;
- `nm`, para localizar símbolos e endereços de funções;
- `checksec`, para verificar as proteções habilitadas nos binários.

## Ambiente de execução

O trabalho foi desenvolvido e testado em um ambiente **Ubuntu Linux x86-64**.

Os comandos, flags e resultados apresentados dependem de características desse ambiente, como o formato de executável ELF, os cabeçalhos POSIX, as ferramentas GNU e a organização da pilha no Linux.

Por isso, a execução direta no Windows pode produzir resultados diferentes ou não aceitar algumas das flags utilizadas. Para reproduzir a análise, recomenda-se utilizar Linux, WSL ou uma máquina virtual Ubuntu.

## Como o exercício foi montado

O código fornecido possui um buffer local de 10 bytes e utiliza a função `gets()`, que não permite informar um limite para a quantidade de caracteres recebida.

A vulnerabilidade está concentrada neste trecho:

```c
void c() {
    char buff[10];

    buff[0] = 'a';
    buff[1] = 'a';
    buff[2] = 'a';
    buff[3] = 'a';
    buff[4] = 'a';

    gets(buff);
}
```

Quando uma entrada maior que o buffer é fornecida, os bytes excedentes continuam sendo gravados na pilha e podem sobrescrever informações importantes, inclusive o endereço de retorno da função.

O mesmo código-fonte foi compilado de três maneiras:

- sem proteção de pilha e sem PIE;
- com `-fstack-protector-all`;
- com `-fPIE -pie`.

Isso permitiu comparar diretamente o comportamento do programa em cada cenário.

## Os conceitos envolvidos

### Buffer overflow

Um buffer overflow acontece quando um programa grava mais dados em um buffer do que o espaço reservado para ele.

No código analisado, `buff` possui 10 bytes, mas `gets()` aceita uma entrada de tamanho indefinido.

### Pilha de execução

A pilha armazena variáveis locais, o valor salvo do registrador de base e o endereço para o qual a função deve retornar.

Na versão sem proteção, a disposição relevante encontrada foi:

```text
buff[10] -> RBP salvo[8] -> endereço de retorno
```

Portanto, foram necessários 18 bytes para alcançar o endereço de retorno.

### Little-endian

A arquitetura x86-64 armazena valores de múltiplos bytes começando pelo byte menos significativo.

Por isso, o endereço da função de destino precisou ser inserido no payload em ordem little-endian.

### Stack canary

O stack canary é um valor inserido entre as variáveis locais e os dados de controle da pilha.

Antes de retornar, o programa verifica se esse valor foi alterado. Caso tenha sido, a execução é encerrada.

### PIE e ASLR

PIE permite que o executável seja carregado em diferentes regiões da memória.

Em conjunto com o ASLR, isso faz com que o endereço absoluto das funções varie entre execuções.

## Tarefas

### As perguntas que precisamos responder

1. Qual entrada deve ser fornecida para que a função `alvo()` seja executada?
2. Por que o ataque deixa de funcionar com `-fstack-protector-all`?
3. Por que o mesmo payload deixa de funcionar com `-fPIE -pie`?
4. Quais funções da linguagem C são mais adequadas para trabalhar com buffers?

## Como organizamos a resolução

### Tarefa 1 - Exploração do buffer overflow

O programa foi compilado sem stack protector, sem PIE e com a pilha executável.

O `checksec` confirmou:

- ausência de stack canary;
- PIE desativado;
- NX desativado.

A análise do assembly da função `c()` mostrou que o buffer começa em `rbp-0xa`.

Como o endereço de retorno está depois dos 8 bytes do `RBP` salvo, o deslocamento encontrado foi:

```text
10 bytes do buffer + 8 bytes do RBP salvo = 18 bytes
```

A entrada utilizada foi formada por 18 bytes de preenchimento seguidos pelo endereço de `alvo()` em little-endian.

Ao sobrescrever o endereço de retorno com o endereço da função `alvo()`, a instrução `ret` desviou o fluxo do programa e produziu:

```text
Achou!!
```

A falha de segmentação posterior é esperada, pois `alvo()` não foi chamada por uma instrução `call` normal e não recebeu um endereço de retorno válido.

### Tarefa 2 - Stack protector

O código foi recompilado com a flag:

```text
-fstack-protector-all
```

O `checksec` passou a indicar a presença de um stack canary.

No assembly protegido, o programa copia um valor de `fs:0x28` para a pilha no início da função. Antes de retornar, esse valor é comparado com o original.

Como o overflow precisa atravessar essa região para alcançar o endereço de retorno, o canário também é sobrescrito. A comparação falha e o programa chama `__stack_chk_fail()`.

O resultado observado foi:

```text
*** stack smashing detected ***
```

Assim, o processo é encerrado antes que o endereço de retorno adulterado seja utilizado.

### Tarefa 3 - PIE

O programa foi recompilado com:

```text
-fPIE -pie
```

O `checksec` confirmou:

```text
PIE enabled
```

O payload criado na Tarefa 1 causou falha de segmentação e não executou `alvo()`, pois continha um endereço pertencente ao executável sem PIE.

Durante duas execuções, o endereço de `alvo()` foi:

```text
0x5ff307eb31bb
0x5c42b14d61bb
```

Os endereços possuem o mesmo deslocamento final, mas bases diferentes. Isso demonstra que a posição do executável foi alterada entre as execuções.

PIE não elimina o buffer overflow. A função `gets()` continua escrevendo além do limite do buffer. A proteção dificulta o ataque porque o endereço absoluto da função de destino deixa de ser previsível.

### Tarefa 4 - Funções adequadas para buffers

Para trabalhar com buffers de forma mais segura, podem ser utilizadas funções como `fgets()`, `getline()`, `snprintf()`, `memcpy()`, `memmove()` e `strnlen()`.

Também é possível utilizar `scanf()` com uma largura máxima definida.

Funções como `gets()`, `strcpy()`, `strcat()` e `sprintf()` devem ser evitadas quando o tamanho da entrada não é controlado.

`strncpy()` e `strncat()` também não são automaticamente seguras, pois ainda exigem o cálculo correto do espaço disponível e cuidado com o terminador nulo.

## Estrutura do projeto

```text
buffer-overflow-ufu/
├── src/
│   └── exploitme.c
├── entrada_teste.bin
├── exploit.bin
├── exploitme
├── exploitme_canary
├── exploitme_pie
├── .gitignore
├── LICENSE
└── README.md
```

Os três binários representam:

- `exploitme`: versão vulnerável sem as proteções analisadas;
- `exploitme_canary`: versão compilada com stack protector;
- `exploitme_pie`: versão compilada com PIE.

## Conclusão

O trabalho demonstrou como uma função insegura pode permitir a alteração do fluxo de execução de um programa.

Sem proteções, foi possível sobrescrever o endereço de retorno e executar diretamente a função `alvo()`.

Com o stack protector, a corrupção da pilha foi detectada antes do retorno da função. Com PIE, o endereço absoluto de `alvo()` passou a variar entre execuções, fazendo com que o payload baseado em um endereço fixo deixasse de funcionar.

Esses mecanismos aumentam a dificuldade de exploração, mas não substituem a escrita segura do código. A correção principal continua sendo impedir a escrita além dos limites do buffer e utilizar funções que permitam controlar a quantidade de dados recebida ou copiada.
