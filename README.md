# S.O. 2025.1 - Atividade 04 - Prática de [comunicação de tarefas](https://wiki.inf.ufpr.br/maziero/lib/exe/fetch.php?media=socm:socm-08.pdf)

## Informações gerais

- **Objetivo do repositório**: Repositório para atividade avaliativa dos alunos
- **Assunto**: Escalonamento de tarefas (processos)
- **Público alvo**: alunos da disciplina de SO (Sistemas Operacionais) do curso de TADS (Superior em Tecnologia em Análise e Desenvolvimento de Sistemas) no CNAT-IFRN (Instituto Federal de Educação, Ciência e Tecnologia do Rio Grande do Norte - Campus Natal-Central).
- disciplina: **SO** [Sistemas Operacionais](https://github.com/sistemas-operacionais/)
- professor: [Leonardo A. Minora](https://github.com/leonardo-minora)
- aluno:  [Wellington G. Coutinho](https://github.com/Wellcoutinhoch)

## Sumário

1. Pré-requisitos para a tarefa
2. Comunicação com arquivos
3. Comunicação com memória compartilhada
4. Tarefas do aluno

---

## Parte 1. Pré-requisitos para a tarefa

1. Docker instalado e serviço em execução
2. Imagem do Fedora baixado e conteiner pré-configurado com `Dockerfile`
3. `gcc` instalado na imagem ou no conteiner

---

## Parte 2. Comunicação com arquivos

Aqui estão dois programas em **ANSI C** que se comunicam usando **arquivos** no Linux, com um processo escrevendo e outro lendo. Utilizaremos operações de arquivo padrão (`fopen`, `fwrite`, `fread`).

Código e tutorial montado pelo [deepseek](https://chat.deepseek.com).

---

### **Arquivo 1: `writer-file.c` (Escritor)**
Responsável por criar/escrever em um arquivo que será lido pelo segundo processo.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>      // Para open(), O_CREAT, etc.
#include <sys/stat.h>   // Para permissões (S_IRUSR, S_IWUSR)

#define FILENAME "comunicacao.txt"

int main() {
    FILE *file;
    const char *message = "Olá, comunicação via arquivo!\n";

    // Abre o arquivo para escrita (cria se não existir, limpa conteúdo existente)
    file = fopen(FILENAME, "w");
    if (file == NULL) {
        perror("fopen");
        exit(1);
    }

    // Escreve a mensagem no arquivo
    fwrite(message, 1, strlen(message), file);
    fclose(file);

    printf("Escritor: Mensagem escrita no arquivo.\n");
    printf("Escritor: Aguardando leitura...\n");

    // Espera até que o leitor termine (verifica se o arquivo foi renomeado)
    while (access(FILENAME, F_OK) == 0) {
        sleep(1);
    }

    printf("Escritor: Arquivo lido e removido. Finalizado.\n");
    return 0;
}
```

---

### **Arquivo 2: `reader-file.c` (Leitor)**
Responsável por ler o arquivo e depois removê-lo (sinalizando que terminou).

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

#define FILENAME "comunicacao.txt"
#define TEMPNAME "comunicacao.lida"

int main() {
    FILE *file;
    char buffer[1024];

    // Abre o arquivo para leitura
    file = fopen(FILENAME, "r");
    if (file == NULL) {
        perror("fopen");
        exit(1);
    }

    // Lê o conteúdo do arquivo
    fread(buffer, 1, sizeof(buffer), file);
    fclose(file);

    printf("Leitor: Mensagem lida:\n%s", buffer);

    // Remove o arquivo (sinaliza que terminou)
    if (rename(FILENAME, TEMPNAME) != 0) {
        perror("rename");
        exit(1);
    }

    printf("Leitor: Arquivo renomeado para %s. Finalizado.\n", TEMPNAME);
    return 0;
}
```

---

### **Como Compilar e Executar**

1. **Compile ambos os programas:**
   ```sh
   gcc writer-file.c -o writer-file
   gcc reader-file.c -o reader-file
   ```

2. **Execute o escritor primeiro (em um terminal):**
   ```sh
   ./writer-file
   ```
   Saída esperada:
   ```
   Escritor: Mensagem escrita no arquivo.
   Escritor: Aguardando leitura...
   ```

3. **Em outro terminal, execute o leitor:**
   ```sh
   ./reader-file
   ```
   Saída esperada:
   ```
   Leitor: Mensagem lida:
   Olá, comunicação via arquivo!
   Leitor: Arquivo renomeado para comunicacao.lida. Finalizado.
   ```

4. **O escritor detecta que o arquivo foi lido e finaliza:**
   ```
   Escritor: Arquivo lido e removido. Finalizado.
   ```

---

### **Explicação do Funcionamento**
1. **`writer-file.c`**:
   - Cria/abre o arquivo `comunicacao.txt` em modo escrita (`"w"`).
   - Escreve a mensagem e fecha o arquivo.
   - Fica em loop até que o arquivo seja renomeado (`access(FILENAME, F_OK) != 0`).

2. **`reader-file.c`**:
   - Abre o arquivo `comunicacao.txt` em modo leitura (`"r"`).
   - Lê o conteúdo e exibe na tela.
   - Renomeia o arquivo para `comunicacao.lida` (sinalizando que terminou).

---

### **Mecanismo de Sincronização**
- O **escritor** espera até que o arquivo original (`comunicacao.txt`) não exista mais (renomeado pelo leitor).
- O **leitor** renomeia o arquivo após a leitura, liberando o escritor para finalizar.

### **Vantagens**
- Simplicidade (usa apenas operações de arquivo padrão).
- Não requer memória compartilhada ou semáforos.
- Funciona mesmo entre processos não relacionados.

### **Possíveis Melhorias**
- Usar **`flock()`** para travar o arquivo e evitar condições de corrida.
- Usar um **pipe nomeado (FIFO)** para comunicação contínua.

---

## Parte 3. Comunicação com memória compartilhada

Aqui estão dois programas em **ANSI C** que se comunicam usando **memória compartilhada** no Linux, utilizando a API do kernel (`shmget`, `shmat`, etc.).  

Código e tutorial montado pelo [deepseek](https://chat.deepseek.com).

---

### **Arquivo 1: `writer-mem.c` (Escritor)**
Responsável por criar a memória compartilhada e escrever uma mensagem nela.  

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/ipc.h>
#include <sys/shm.h>
#include <unistd.h>

#define SHM_SIZE 1024  // Tamanho da memória compartilhada

int main() {
    int shmid;
    key_t key = 1234;  // Chave única para a memória compartilhada
    char *shm, *data;

    // Cria um segmento de memória compartilhada
    shmid = shmget(key, SHM_SIZE, IPC_CREAT | 0666);
    if (shmid == -1) {
        perror("shmget");
        exit(1);
    }

    // Anexa o segmento ao espaço de endereçamento do processo
    shm = shmat(shmid, NULL, 0);
    if (shm == (char *)-1) {
        perror("shmat");
        exit(1);
    }

    // Escreve na memória compartilhada
    strcpy(shm, "Olá, Memória Compartilhada!");

    printf("Escritor: Mensagem escrita na memória compartilhada.\n");
    printf("Escritor: Aguardando leitura...\n");

    // Espera até que o leitor sinalize que terminou (modificando o primeiro byte)
    while (*shm != '*') {
        sleep(1);
    }

    // Libera a memória compartilhada
    shmdt(shm);
    shmctl(shmid, IPC_RMID, NULL);  // Remove o segmento

    printf("Escritor: Memória compartilhada liberada.\n");
    return 0;
}
```

---

### **Arquivo 2: `reader-mem.c` (Leitor)**
Responsável por ler a mensagem da memória compartilhada e sinalizar quando terminar.  

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/ipc.h>
#include <sys/shm.h>
#include <unistd.h>

#define SHM_SIZE 1024

int main() {
    int shmid;
    key_t key = 1234;  // Mesma chave usada no escritor
    char *shm;

    // Obtém o segmento de memória compartilhada
    shmid = shmget(key, SHM_SIZE, 0666);
    if (shmid == -1) {
        perror("shmget");
        exit(1);
    }

    // Anexa o segmento ao espaço de endereçamento do processo
    shm = shmat(shmid, NULL, 0);
    if (shm == (char *)-1) {
        perror("shmat");
        exit(1);
    }

    // Lê a mensagem da memória compartilhada
    printf("Leitor: Mensagem lida: \"%s\"\n", shm);

    // Sinaliza para o escritor que terminou (modificando o primeiro byte)
    *shm = '*';

    // Libera a memória compartilhada
    shmdt(shm);

    printf("Leitor: Finalizado.\n");
    return 0;
}
```

---

### **Como Compilar e Executar**

1. **Compile ambos os programas:**
   ```sh
   gcc writer-mem.c -o writer-mem
   gcc reader-mem.c -o reader-mem
   ```

2. **Execute o escritor primeiro (em um terminal):**
   ```sh
   ./writer-mem
   ```
   Saída esperada:
   ```
   Escritor: Mensagem escrita na memória compartilhada.
   Escritor: Aguardando leitura...
   ```

3. **Em outro terminal, execute o leitor:**
   ```sh
   ./reader-mem
   ```
   Saída esperada:
   ```
   Leitor: Mensagem lida: "Olá, Memória Compartilhada!"
   Leitor: Finalizado.
   ```

4. **O escritor termina após a leitura:**
   ```
   Escritor: Memória compartilhada liberada.
   ```

---

### **Explicação do Funcionamento**
1. **`writer-mem.c`**:
   - Cria um segmento de memória compartilhada (`shmget`).
   - Escreve uma mensagem (`strcpy`).
   - Fica em loop até que o leitor modifique o primeiro byte (`*shm = '*'`).
   - Libera a memória compartilhada (`shmdt`, `shmctl`).

2. **`reader-mem.c`**:
   - Acessa a mesma memória compartilhada usando a mesma chave (`key = 1234`).
   - Lê a mensagem e imprime.
   - Sinaliza que terminou alterando o primeiro byte (`*shm = '*'`).
   - Libera a memória compartilhada (`shmdt`).

### **Observações**
- **`key_t key`** deve ser o mesmo nos dois programas.
- **`shmctl(..., IPC_RMID, ...)`** remove a memória compartilhada após o uso.
- Se o `writer` não for executado primeiro, o `reader` falhará (pois a memória não existirá).


---

## Parte 4. Tarefas do aluno

1. Fork desse repositório e lembre de colocar seu nome na linha 10 deste arquivo leia-me (README.md)
2. Refaça os programas utilizando threads
3. Compile e execute para testar a funcionalidade
4. Escreva um relatório comparando
   1. A complexidade de leitura e escrita de programas que se comunicam usando processos (kernel) e threads
   2. O tempo de execução dos programas usando processos e threads   

# Relatório da Atividade: Comunicação entre Processos e Threads

## 1. Complexidade de leitura e escrita em programas que usam processos (kernel) e threads

### Comunicação entre Processos (Processos)

- Utilizam mecanismos do kernel como arquivos, pipes, memória compartilhada, etc.
- Requerem mais cuidado com sincronização e gerenciamento, pois os processos possuem espaços de memória separados.
- Comunicação pode ser mais lenta devido a troca de contexto entre processos.
- Exigem criação e manipulação de recursos do sistema, o que pode aumentar a complexidade do código.

### Comunicação entre Threads

- Threads compartilham o mesmo espaço de memória dentro do mesmo processo.
- Comunicação é direta e mais rápida, pois não há troca de contexto de processos.
- Sincronização ainda é necessária para evitar condições de corrida.
- Código geralmente é mais simples para comunicação, mas pode ser mais propenso a erros relacionados a concorrência.

## 2. Tempo de execução dos programas usando processos e threads

- Programas usando **threads** tendem a executar com maior rapidez na comunicação, devido à menor sobrecarga do sistema.
- Programas usando **processos** podem sofrer atraso devido à troca de contexto e operações mais custosas do kernel.
- A diferença de tempo varia conforme a complexidade da comunicação e o sistema operacional.
- Em testes simples, threads mostraram comunicação quase instantânea, enquanto processos tiveram delays perceptíveis na sincronização via arquivos ou memória compartilhada.

# Conclusão

Para comunicação entre tarefas dentro do mesmo programa, o uso de threads é geralmente mais eficiente e simples. Porém, quando se deseja isolamento e segurança, processos são preferíveis, apesar da maior complexidade e custo.

---

## Referências no texto

Como mostrado na **Figura 1**, o programa escritor escreve a mensagem no arquivo e aguarda a leitura. Em seguida, na **Figura 2**, o leitor lê a mensagem e renomeia o arquivo, sinalizando que terminou. Por fim, a **Figura 3** demonstra a comunicação via memória compartilhada.

---

## Figuras

**Figura 1:** Execução do programa escritor (`writer-file.c`), mostrando a mensagem escrita e o aguardo pela leitura.

![Execução do programa escritor](https://i.postimg.cc/XqnvmC3D/Captura-de-tela-2025-08-09-175250.png)

---

**Figura 2:** Comunicação com arquivo, saída do programa leitor (`reader-file.c`) lendo o arquivo e renomeando-o.

![comunicação com arquivo](https://i.postimg.cc/52D21Gbz/Captura-de-tela-2025-08-09-175517.png)

---

**Figura 3:** Execução do programa com comunicação via memória compartilhada, exibindo a mensagem lida.

![Execução do programa com comunicação via memória compartilhada](https://i.postimg.cc/6QstFvfL/Captura-de-tela-2025-08-09-175542.png)

