### Criando ambiente isolado com unshare

O comando unshare é usado para criar namespaces, isolando recursos como PID, rede, sistema de arquivos, etc. Neste exemplo, criaremos um novo namespace de PID, rede e sistema de montagem, e iniciaremos um bash dentro desse ambiente isolado.

```shell
sudo unshare -p -m -n -f --mount-proc bash
```

- p: Cria um novo namespace de PID, onde os processos terão IDs independentes do sistema principal.

- m: Cria um novo namespace de montagem, isolando o sistema de arquivos.

- n: Cria um novo namespace de rede, isolando as interfaces de rede.

- f: Força a criação do novo processo no namespace.

- -mount-proc: Monta o sistema de arquivos /proc dentro do namespace, permitindo visualizar apenas os processos do ambiente isolado.

### Identificando o Processo bash Criado pelo unshare
Depois de iniciar o bash com unshare, ele será o PID 1 dentro do namespace, mas precisamos do PID global para mover o processo para o cgroup. Para identificar esse PID:

Em outro terminal, execute:

```shell
ps -ef | grep unshare
```

Você verá uma lista de processos relacionados ao comando unshare. Procure pelo PID do unshare (geralmente, o último da cadeia).

Use o comando pstree para visualizar a árvore de processos e identificar o bash iniciado pelo unshare:

```shell
 pstree -p <PID_DO_UNSHARE>
```

Substitua <PID_DO_UNSHARE> pelo PID que você encontrou. O PID do bash será listado como um filho do unshare.

### Limitando recursos com cgroup

Criando e Configurando um cgroup para Limitar Recursos
Agora que temos o PID do bash criado pelo unshare, vamos criar um cgroup para controlar o uso de CPU e memória desse processo.

1. Crie o cgroup:
```shell
 sudo mkdir /sys/fs/cgroup/mycontainer
```

2. Configure os limites de CPU e Memória:

- Limitar o uso de CPU a 50% de um núcleo:
```shell
  echo "50000 100000" | sudo tee /sys/fs/cgroup/mycontainer/cpu.max
```


- Limitar o uso de memória a 100 MB:
```shell
  echo "100M" | sudo tee /sys/fs/cgroup/mycontainer/memory.max
```

3. Mover o Processo bash para o cgroup:

Use o PID do bash que você encontrou anteriormente e mova-o para o cgroup:
```shell
  echo <PID_DO_BASH> | sudo tee /sys/fs/cgroup/mycontainer/cgroup.procs
```

Isso garantirá que o bash e todos os seus processos filhos estejam limitados pelos recursos configurados no cgroup.

#### Executando um Processo no Ambiente Isolado

Dentro do bash que está rodando no namespace e agora está no cgroup, você pode executar processos que serão limitados pelos recursos do cgroup.

- Execute o comando stress para gerar carga e testar os limites:

```shell
stress --cpu 2 --vm 1 --vm-bytes 80M --timeout 30
```

- Este comando simula carga na CPU e consome memória, e deve estar limitado pelo que configuramos no cgroup.

#### Analisando o Uso de Recursos com cpu.stat

Podemos verificar se o cgroup está aplicando as limitações corretamente analisando o arquivo cpu.stat do cgroup.

1. Verifique o status de uso de CPU:
```shell
 cat /sys/fs/cgroup/mycontainer/cpu.stat
```


- usage_usec: Tempo total de CPU usado pelo cgroup (em microssegundos).
- nr_throttled: Quantas vezes o cgroup foi limitado (throttled).
- throttled_usec: Tempo total durante o qual o cgroup foi limitado.

2. Entendendo os Valores do cpu.stat:

- usage_usec: Esse valor representa o tempo total de CPU utilizado por todos os processos no cgroup, em microssegundos. Para calcular quanto tempo de CPU foi consumido em segundos, divida esse valor por 1.000.000.
- nr_throttled: Indica quantas vezes o cgroup foi impedido de usar a CPU além do limite definido. Se esse valor for alto, significa que os processos estão frequentemente sendo limitados pelo cgroup.
- throttled_usec: Mostra o tempo total (em microssegundos) em que os processos foram limitados e não puderam usar a CPU. Para entender a porcentagem de tempo em que a CPU foi limitada, você pode usar a seguinte fórmula:

```shell
Porcentagem de Throttle = (throttled_usec / (throttled_usec + usage_usec)) * 100
```
- Isso mostra a proporção do tempo total que os processos foram impedidos de usar a CPU.

3. Exemplo de Cálculo:

- Se usage_usec for 5000000 (5 segundos) e throttled_usec for 2000000 (2 segundos), a porcentagem de tempo em que o cgroup foi limitado seria:

  Porcentagem de Throttle = (2000000 / (2000000 + 5000000)) * 100 = 28,57%
Isso significa que, durante 28,57% do tempo, os processos do cgroup foram impedidos de usar a CPU devido às limitações definidas.

4. Comando para Calcular Automaticamente:

O awk é uma ferramenta poderosa de processamento de texto que permite buscar padrões e realizar cálculos em arquivos de texto. No exemplo abaixo, utilizamos o awk para calcular automaticamente a porcentagem de throttle a partir dos valores do cpu.stat:
```shell
  awk '{if ($1 == "usage_usec") usage=$2; if ($1 == "throttled_usec") throttle=$2} END {print "Porcentagem de Throttle: " (throttle / (throttle + usage)) * 100 "%"}' /sys/fs/cgroup/mycontainer/cpu.stat
```

- Explicação do comando:

    - $1, $2: Em awk, $1 representa o primeiro campo de cada linha do arquivo (ou seja, a primeira coluna), enquanto $2 representa o segundo campo (ou segunda coluna). No contexto do arquivo cpu.stat, o primeiro campo ($1) é o nome do atributo (como usage_usec ou throttled_usec), e o de comandos a ser executado para cada linha do arquivo.

    - if ($1 == "usage_usec") usage=$2: Se o primeiro campo da linha ($1) for usage_usec, armazena o valor correspondente (segundo campo, $2) na variável usage.
    
    - if ($1 == "throttled_usec") throttle=$2: Se o primeiro campo for throttled_usec, armazena o valor correspondente na variável throttle.

    - END { ... }: Após processar todas as linhas, executa o bloco final, que calcula e imprime a porcentagem de throttle.
    
    - print "Porcentagem de Throttle: " (throttle / (throttle + usage)) * 100 "%": Calcula a porcentagem de throttle e a imprime no formato desejado.**: Define um bloco de comandos a ser executado para cada linha do arquivo.
- Este comando lê os valores de usage_usec e throttled_usec do arquivo cpu.stat e calcula a porcentagem de tempo em que o cgroup foi limitado.

5. Use o comando watch para monitorar o arquivo cpu.stat em tempo real:
```shell
watch -n 1 cat /sys/fs/cgroup/mycontainer/cpu.stat
```

- Isso atualizará a cada segundo, permitindo observar como os valores mudam conforme o processo consome recursos.


### Dicas e Cuidados

- Criar um Novo cgroup para Cada Teste: Para garantir que não haja valores acumulados no cpu.stat de execuções anteriores, crie um novo cgroup para cada teste.
- Remover o cgroup: Após o teste, você pode remover o cgroup para limpar o sistema:
```shell
sudo rmdir /sys/fs/cgroup/mycontainer
```

- Certifique-se de que não há processos ativos no cgroup antes de removê-lo.