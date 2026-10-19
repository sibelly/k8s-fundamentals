# Resumo Comparativo Arquitetural: Docker, Podman e containerd

![Comparativo Arquitetural de Containers](../.github/assets/comparativo-arquitetural-containers.png?raw=true)

No contexto das primitivas do kernel Linux (como **namespaces**, **cgroups**, **Netlink** e **Seccomp**), **a principal diferença entre eles não está na forma como o container roda em si — já que todos usam as mesmas primitivas do kernel —, mas sim em como o ciclo de vida e a arquitetura de gerenciamento desses processos são estruturados**.

Aqui está a divisão de como cada um interage com o ecossistema e com o kernel.

## 🐋 Docker (Abordagem baseada em Daemon)

O Docker tradicional funciona sob um modelo cliente-servidor e não gerencia as primitivas do kernel diretamente para rodar o container.

- **Como funciona:** O comando `docker` (CLI) fala com o `dockerd` (um daemon centralizado que roda como `root`).
- **Ponto cego no Kernel:** Historicamente, por depender de um daemon central rodando como root, o Docker tinha dificuldades em mapear nativamente os **User Namespaces** do Linux de forma simples para usuários comuns (arquitetura *rootful* por padrão). Além disso, se o daemon do Docker cair ou for reiniciado, o gerenciamento dos processos filhos (os containers) pode ser afetado se não estiver configurado o *live-restore*.

## 🦭 Podman (Abordagem "Daemonless" e nativa do Kernel)

O Podman foi desenhado para se integrar diretamente ao modelo de processos tradicional do Linux, eliminando o intermediário (o daemon).

- **Como funciona:** Quando você digita `podman run`, o próprio processo do Podman faz as chamadas de sistema e cria o container. Ele usa uma ferramenta chamada `conmon` (*container monitor*) para monitorar o processo do container.
- **Exploração do Kernel:** O Podman se destaca no uso de **User Namespaces**. Ele permite que usuários comuns (sem privilégios de `root`) criem containers seguros (**Rootless Containers**), mapeando o UID do usuário comum para o UID `0` dentro do container usando as primitivas de mapeamento de IDs do kernel. Ele também se integra nativamente ao **systemd**, tratando containers como serviços Linux padrão.


## 📦 containerd (Gerenciador do ciclo de vida dos containers)

O `containerd` é um serviço de gerenciamento de containers utilizado pelo Docker e por outras plataformas. Ele fica entre a camada de gerenciamento de alto nível e o runtime responsável por criar e executar os containers.

Ele é um daemon focado estritamente em gerenciar o ciclo de vida do container: baixar imagens, gerenciar o armazenamento e supervisionar a execução.

- **Como funciona:** No Docker, o comando `docker run` é enviado pelo cliente CLI ao `dockerd`, que processa a solicitação e delega o gerenciamento do ciclo de vida ao `containerd` por meio de uma API gRPC. O `containerd` gerencia operações como download de imagens, armazenamento e criação, execução e encerramento de containers.

- **Camada de execução:** O `containerd` utiliza o `containerd-shim` para manter o acompanhamento dos processos dos containers e aciona um runtime OCI, normalmente o `runc`, para configurar e iniciar cada container. O `runc` realiza as operações de baixo nível necessárias para criar o ambiente isolado, incluindo chamadas de sistema (*syscalls*) e configuração de namespaces, cgroups e outros recursos do kernel Linux.

- **Relação com o Kernel:** O `containerd` não cria os namespaces e cgroups diretamente. Ele traduz os comandos em uma especificação padrão (OCI) e chama o **`runc`** (ou outro runtime de baixo nível). É o `runc` que faz as *syscalls* (chamadas de sistema) do Linux, como `clone()` (para namespaces), e configura os arquivos em `/sys/fs/cgroup/` (para cgroups). O Docker, inclusive, usa o `containerd` por baixo dos panos.

- **Relação com o Docker:** O `dockerd` oferece a interface de gerenciamento, a integração com redes, volumes, build de imagens e outras funcionalidades para desenvolvedores. O `containerd` gerencia o ciclo de vida dos containers, enquanto o `runc` executa as operações de baixo nível necessárias para iniciá-los.

### 🔄 Fluxo de execução do Docker

`docker` (CLI) → `dockerd` → `containerd` → `containerd-shim` → `runc` → **Kernel Linux / container**

**Importante:** esse fluxo representa as responsabilidades lógicas, não necessariamente uma sequência literal de processos. O `runc` normalmente configura e inicia o container e termina sua execução; o `containerd-shim` permanece para acompanhar o processo do container e facilitar seu gerenciamento.

---

## 📊 Resumo Comparativo Arquitetural

| **Critério** | **Docker** | **Podman** | **containerd** |
|---|---|---|---|
| **Dependência de Daemon** | **Sim** (`dockerd`) | **Não** (Daemonless) | **Sim** (`containerd`) |
| **Execução de Syscalls** | Delega ao `containerd` / `runc` | Delega diretamente ao `runc` / `crun` | Delega diretamente ao `runc` |
| **Foco em Rootless** | Suporta, mas requer ajustes complexos | **Nativo** via *User Namespaces* do kernel | Suporta através de configurações de runtime |
| **Propósito Principal** | Plataforma completa para desenvolvedores | Substituto direto do Docker com foco em segurança | Runtime industrial integrado ao Kubernetes (CRI) |

---

## 🎯 Em resumo

No nível do kernel, um container rodando no Docker, Podman ou containerd utiliza as mesmas primitivas fundamentais: é um processo isolado por **namespaces** e limitado por **cgroups**.

A diferença real é **quem segura as rédeas desse processo na Terra dos Usuários (*userspace*)**.