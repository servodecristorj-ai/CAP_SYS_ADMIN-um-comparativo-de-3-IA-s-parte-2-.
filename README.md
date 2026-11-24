# Repositório - Parte 2: Fisiologia do Peido x Segurança de Container
## Modelo Esfíncter: Escape de Sandbox

### Resposta curta: qual parte é mais vulnerável?

Hoje, não é o kernel em si. É a fronteira voluntária.

No peido, é o esfíncter externo. No container, é a camada de política que o operador pode afrouxar: `--privileged`, `CAP_SYS_ADMIN`, `hostPath`, montagem do socket do Docker, `seccomp` desabilitado, `AppArmor` em modo complain.

É a parte mais vulnerável porque é a parte que o humano relaxa por conveniência, igual relaxa o esfíncter externo pra liberar o gás. O ataque real quase nunca quebra o kernel do zero, ele passa pela porta que alguém deixou aberta.

### A ponte: fisiologia x container

| Fisiologia do peido | Container até o host | Função |
| --- | --- | --- |
| Intestino grosso | Namespace do container, PID, NET, MNT, USER | Espaço confinado, o sandbox |
| Reto | Runtime, runc, containerd | Canal final antes da saída |
| Esfíncter interno | Namespaces, cgroups, seccomp default do kernel | Involuntário, sempre ativo, só falha com exploit de kernel |
| Esfíncter externo | Seccomp custom, AppArmor, SELinux, drop de capabilities, read-only rootfs | Voluntário, configurável, é onde mora o escape |
| Pressão interna | Exploit, escalação de privilégio, exaustão, misconfiguração | Força que tenta vencer a fronteira |
| Glúteo firme | Hardening, no privileged, user namespace, no-new-privileges | Aumenta resistência, mas não é infinito |
| Cheiro | Exfiltração de dados, vazamento silencioso | Você só percebe depois que já vazou |
| Barulho | Logs, alertas, detecção | Escape barulhento é detectado, escape silencioso não |

### Por que o escape acontece nos dois

Nos dois, a regra é a mesma que você já descreveu no levantamento terra:

**Pressão vence resistência.**

No peido: pressão abdominal lá em cima vence glúteo firme, gás sai de uma vez, vibra tudo, vira barulhento e fedido.

No container: um processo com `CAP_SYS_ADMIN` ou `--privileged` é o equivalente a levantar 200kg com pressão abdominal máxima. Ele já tem permissão pra mexer na fronteira. Aí ele monta o host, escapa do namespace, vira processo do host. Não precisa quebrar o kernel, só precisa que a pressão, no caso o privilégio, seja maior que a resistência.

E tem o caso clássico que é igual ao SBD, Silent But Deadly: escape silencioso. O atacante escapa sem barulho, sem log alto, mas deixa cheiro, que no caso é dado vazando, conexão estranha, arquivo novo no host. Igual peido silencioso que compensa no cheiro.

### Os 3 escapes mais prováveis hoje, em ordem

- **1. Misconfiguração, o esfíncter externo relaxado:** container privilegiado, `hostPath: /`, Docker socket montado, RBAC permissivo no Kubernetes. É o mais comum e o mais vulnerável.
- **2. Runtime, o reto:** vulnerabilidade no runc ou containerd, file descriptor vazado, que deixa o processo acessar o host pelo canal de criação do container.
- **3. Kernel, o esfíncter interno:** bug em syscall, io_uring, eBPF, que permite furar namespace e cgroup. É o mais grave, mas o mais raro e o mais difícil de explorar.

Ou seja, o ponto mais vulnerável não é o mais profundo, é o mais humano. É onde a gente decide relaxar pra facilitar a vida.

### Como deixar o glúteo firme no container

Traduzindo seu modelo pra hardening:

- **Nunca `--privileged`:** é o equivalente a abrir o esfíncter externo de propósito
- **Drop `ALL` e só adiciona o necessário:** tira `SYS_ADMIN`, `NET_ADMIN`, `SYS_PTRACE`
- **Sem `hostPath` e sem socket do Docker:** não dá acesso direto ao host
- **Rootfs read-only + `no-new-privileges` + user namespace:** aumenta a resistência sem depender só do kernel
- **Seccomp + AppArmor / SELinux em enforce:** mantém o esfíncter externo contraído por padrão

Continuação de: [parte1](https://github.com/servodecristorj-ai/CAP_SYS_ADMIN-um-comparativo-de-3-IA-s)
