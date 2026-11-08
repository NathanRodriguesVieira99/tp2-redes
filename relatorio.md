# TP2

**Aluno:** Nathan Rodrigues Vieira
**Ambiente:** Ubuntu 24.04 em container Docker (systemd ativo), host Windows 11 + Docker Desktop

---

## Preparação

```powershell
docker run -d --name lab --privileged -v /sys/fs/cgroup:/sys/fs/cgroup:rw jrei/systemd-ubuntu:24.04
docker exec -it lab bash
```

```bash
apt-get update && apt-get install -y telnet curl openssh-server net-tools lsof netcat-openbsd tcpdump nmap nano
```

---

## Exercício 1 — Sessão TCP manual e resposta HTTP mínima

**Enunciado:** Abra uma conexão TCP manual com telnet para um domínio público na porta 80 e registre evidência de conexão estabelecida. Envie uma requisição HTTP mínima (linha GET, header Host e linha em branco) e registre a status line e pelo menos 3 headers da resposta. Registre um trecho curto do corpo (ou a indicação de redirecionamento) e finalize com 2–3 linhas explicando como essa evidência ajuda a diagnosticar problemas de porta/serviço.

**Comando:**

```bash
telnet example.com 80
```

Ao conectar, digite:

```text
GET / HTTP/1.1
Host: example.com
```

**Resposta:**

```text
Trying 104.20.23.154...
Connected to example.com.
Escape character is '^]'.
HTTP/1.1 200 OK
Date: Sun, 08 Nov 2026 20:12:49 GMT
Content-Type: text/html; charset=utf-8
Transfer-Encoding: chunked
Connection: close
Server: cloudflare
Last-Modified: Thu, 05 Nov 2026 21:17:41 GMT
Allow: GET, HEAD
Accept-Ranges: bytes
Age: 4
cf-cache-status: HIT
CF-RAY: a5773c167ac2feaa-GIG
alt-svc: h3=":443"; ma=86400

241
<!doctype html><html lang=en><head><meta charset=utf-8><link rel=icon href=data:,>...

0

Connection closed by foreign host.
```

O `Connected to example.com.` prova a conexão TCP na porta 80; a status line `HTTP/1.1 200 OK` e os headers provam que o servidor processou a requisição; o trecho `<title>Example Domain</title>` do corpo prova que a resposta chegou completa. Se a porta estivesse bloqueada, essa etapa falharia antes de qualquer resposta HTTP.

---

## Exercício 2 — Coletar e comparar evidências do fluxo HTTP

**Enunciado:** Execute `curl --trace-ascii` para um endpoint público e grave o trace em arquivo (registre nome e tamanho). Extraia do trace: request line, 3 headers enviados, status code e 3 headers recebidos. Repita alterando um parâmetro de comportamento (ex.: seguir redirect) e registre o que mudou, em 2–4 linhas.

**Comando:**

```bash
mkdir evidencias
cd evidencias

curl --trace-ascii trace-example.txt -o pagina.html http://example.com/
curl --trace-ascii trace-sem-redirect.txt -o redir1.html http://httpbin.org/redirect/1
curl --trace-ascii trace-com-redirect.txt -L -o redir2.html http://httpbin.org/redirect/1

ls -lh
head -40 trace-example.txt
grep "HTTP/1.1" trace-sem-redirect.txt
grep "HTTP/1.1" trace-com-redirect.txt
```

**Resposta:**

```text
-rw-r--r-- 1 root root  577 Nov  8 20:13 pagina.html
-rw-r--r-- 1 root root 2.4K Nov  8 20:13 trace-com-redirect.txt
-rw-r--r-- 1 root root 1.9K Nov  8 20:13 trace-example.txt
-rw-r--r-- 1 root root 1.4K Nov  8 20:13 trace-sem-redirect.txt

== Info: Host example.com:80 was resolved.
== Info: IPv6: (none)
== Info: IPv4: 172.66.147.243, 104.20.23.154
== Info:   Trying 172.66.147.243:80...
== Info: Connected to example.com (172.66.147.243) port 80
=> Send header, 74 bytes (0x4a)
0000: GET / HTTP/1.1
0010: Host: example.com
0023: User-Agent: curl/8.5.0
003b: Accept: */*
0048: 
<= Recv header, 17 bytes (0x11)
0000: HTTP/1.1 200 OK
<= Recv header, 37 bytes (0x25)
0000: Date: Sun, 08 Nov 2026 20:14:08 GMT
<= Recv header, 40 bytes (0x28)
0000: Content-Type: text/html; charset=utf-8
<= Recv header, 28 bytes (0x1c)
0000: Transfer-Encoding: chunked
<= Recv header, 24 bytes (0x18)
0000: Connection: keep-alive
<= Recv header, 20 bytes (0x14)
0000: Server: cloudflare
[...]

--- sem redirect ---
0000: GET /redirect/1 HTTP/1.1
0000: HTTP/1.1 302 Found
--- com redirect ---
0000: GET /redirect/1 HTTP/1.1
0000: HTTP/1.1 302 Found
0000: GET /get HTTP/1.1
0000: HTTP/1.1 200 OK
```

Sem `-L` o curl para no `302 Found`. Com `-L` ele lê o header `Location`, refaz a requisição para `/get` e só então recebe `200 OK` — ou seja, o trace passa a ter dois pares request/response.

---

## Exercício 3 — SSH em operação: configurar serviço e comprovar acesso remoto

**Enunciado:** Verifique o status do sshd com `systemctl` e registre a evidência (ativo/inativo). Inspecione o `sshd_config` e registre os valores efetivos de `Port`, `PasswordAuthentication` e `PermitRootLogin`. Conecte via SSH ao próprio host (loopback) e comprove a execução de 1 comando remoto.

**Comando:**

```bash
systemctl status ssh
sshd -T | grep -E "^(port|passwordauthentication|permitrootlogin)"
ssh aluno@127.0.0.1 hostname
ssh aluno@127.0.0.1 whoami
```

**Resposta:**

```text
● ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/usr/lib/systemd/system/ssh.service; enabled; preset: enabled)
     Active: active (running) since Sun, 2026-11-08 20:14:45 UTC; 2s ago
       Docs: man:sshd(8)
             man:sshd_config(5)
    Process: 649 ExecStartPre=/usr/sbin/sshd -t (code=exited, status=0/SUCCESS)
   Main PID: 651 (sshd)
      Tasks: 1 (limit: 9466)

port 22
permitrootlogin yes
passwordauthentication yes

d6825dab8e97
aluno
```

O serviço está `active (running)`, a porta efetiva é 22, a autenticação por senha está habilitada e o login de root está permitido. O `ssh aluno@127.0.0.1 hostname` retornou o hostname do próprio container (`d6825dab8e97`), provando login remoto e execução de comando.

---

## Exercício 4 — Netstat como inventário: conexões, LISTEN e dual-stack

**Enunciado:** Liste as portas TCP em LISTEN e registre a evidência. Liste as conexões ativas e registre trechos curtos. Gere uma visão separada de IPv4 e IPv6 (`-4` e `-6`) e conclua objetivamente: o SSH está exposto em IPv4, IPv6 ou ambos?

**Comando:**

```bash
netstat -tlnp
netstat -tnp
netstat -4tlnp
netstat -6tlnp
```

> Para ter conexão ESTABLISHED, deixe `ssh aluno@127.0.0.1` aberto em outro terminal.

**Resposta:**

```text
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      366/sshd: /usr/sbin 
tcp6       0      0 :::22                   :::*                    LISTEN      366/sshd: /usr/sbin 

Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 127.0.0.1:37218         127.0.0.1:22            ESTABLISHED 517/ssh             
tcp        0      0 127.0.0.1:22            127.0.0.1:37218         ESTABLISHED 519/sshd: aluno [pr 
[... demais conexões em TIME_WAIT omitidas ...]

== IPv4 ==
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      366/sshd: /usr/sbin 

== IPv6 ==
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp6       0      0 :::22                   :::*                    LISTEN      366/sshd: /usr/sbin 
```

O SSH aparece em `0.0.0.0:22` (IPv4) e em `:::22` (IPv6): está exposto nos **dois** stacks. Por isso uma aplicação que prefere AAAA consegue conectar mesmo que o IPv4 esteja bloqueado, e vice-versa.

---

## Exercício 5 — Diagnóstico com lsof

**Enunciado:** Use `lsof` para listar sockets em LISTEN. Escolha 2 portas (uma deve ser a do SSH) e, para cada uma, registre PID, processo, usuário e o trecho da saída. Finalize com 2–3 linhas explicando por que esse mapeamento é obrigatório antes de "mexer" em serviços.

**Comando:**

```bash
lsof -i -P -n | grep LISTEN
lsof -i :22
lsof -i :55000
```

> A porta 55000 só aparece com o `nc` do Exercício 6 rodando.

**Resposta:**

```text
sshd     80 root    3u  IPv4  21288      0t0  TCP *:22 (LISTEN)
sshd     80 root    4u  IPv6  21290      0t0  TCP *:22 (LISTEN)
nc      318 root    3u  IPv4  50200      0t0  TCP *:55000 (LISTEN)

COMMAND PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
sshd     80 root    3u  IPv4  21288      0t0  TCP *:ssh (LISTEN)
sshd     80 root    4u  IPv6  21290      0t0  TCP *:ssh (LISTEN)

COMMAND PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
nc      318 root    3u  IPv4  50200      0t0  TCP *:55000 (LISTEN)
```

A porta 22 pertence ao `sshd` rodando como `root` (PID 80) e a 55000 ao `nc` (PID 318). Sem esse vínculo não dá para saber se a porta é do serviço esperado ou de outro processo — qualquer `restart` ou `kill` viraria tentativa e erro, com risco de derrubar o serviço errado.

---

## Exercício 6 — Criar um serviço TCP mínimo e validar conectividade local

**Enunciado:** Inicie um listener TCP local com `nc` em uma porta alta (50000–60000) e registre o comando. Conecte como cliente em outro terminal e envie uma mensagem curta; registre o que foi recebido no servidor. Prove com `netstat` ou `lsof` que a porta estava em LISTEN durante o experimento.

**Comando:**

```bash
# Terminal A
nc -lvp 55000

# Terminal B (digite a mensagem e Enter)
nc 127.0.0.1 55000

# Terminal C (com o listener aberto)
netstat -tlnp | grep 55000
lsof -i :55000
```

**Resposta:**

```text
tcp        0      0 0.0.0.0:55000           0.0.0.0:*               LISTEN      324/nc              
COMMAND PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
nc      324 root    3u  IPv4  47402      0t0  TCP *:55000 (LISTEN)
Listening on 0.0.0.0 55000
Connection received on localhost 59964
mensagem de teste do cliente
```

O listener subiu com `nc -lvp 55000` e ficou em LISTEN (PID 324) enquanto o cliente conectava. A linha `Connection received on localhost 59964` e a mensagem recebida provam que os bytes chegaram ao servidor pela porta 55000.

---

## Exercício 7 — Captura controlada para provar tráfego do experimento com nc

**Enunciado:** Identifique sua interface ativa e registre o nome. Execute `tcpdump` com filtro para a porta usada no `nc`, salvando em `.pcap`, e registre a evidência do arquivo gerado. Leia com `tcpdump -r` e registre até 10 linhas que provem a troca de tráfego no socket.

**Comando:**

```bash
# Terminal A
ip -br a
tcpdump -i lo port 55000 -w captura-nc.pcap

# Terminal B (com o nc -lvp 55000 aberto)
nc 127.0.0.1 55000

# Terminal A: Ctrl+C e depois
ls -lh captura-nc.pcap
tcpdump -r captura-nc.pcap -nn | head
```

**Resposta:**

```text
lo               UNKNOWN        127.0.0.1/8 ::1/128 
eth0@if15        UP             172.17.0.2/16 

-rw-r--r-- 1 tcpdump tcpdump 715 Nov  8 20:12 captura-nc.pcap

reading from file captura-nc.pcap, link-type EN10MB (Ethernet), snapshot length 262144
20:12:47.383143 IP 127.0.0.1.34816 > 127.0.0.1.55000: Flags [S], seq 1839521224, win 65495, options [mss 65495,sackOK,TS val 1841593836 ecr 0,nop,wscale 10], length 0
20:12:47.383157 IP 127.0.0.1.55000 > 127.0.0.1.34816: Flags [S.], seq 4077209636, ack 1839521225, win 65483, options [mss 65495,sackOK,TS val 4236574779 ecr 1841593836,nop,wscale 10], length 0
20:12:47.383167 IP 127.0.0.1.34816 > 127.0.0.1.55000: Flags [.], ack 1, win 64, options [nop,nop,TS val 1841593836 ecr 4236574779], length 0
20:12:47.383224 IP 127.0.0.1.34816 > 127.0.0.1.55000: Flags [P.], seq 1:20, ack 1, win 64, options [nop,nop,TS val 1841593836 ecr 4236574779], length 19
20:12:47.383229 IP 127.0.0.1.55000 > 127.0.0.1.34816: Flags [.], ack 20, win 64, options [nop,nop,TS val 4236574779 ecr 1841593836], length 0
20:12:52.382484 IP 127.0.0.1.34816 > 127.0.0.1.55000: Flags [F.], seq 20, ack 1, win 64, options [nop,nop,TS val 1841598836 ecr 4236574779], length 0
20:12:52.382569 IP 127.0.0.1.55000 > 127.0.0.1.34816: Flags [F.], seq 1, ack 21, win 64, options [nop,nop,TS val 4236579779 ecr 1841598836], length 0
20:12:52.382598 IP 127.0.0.1.34816 > 127.0.0.1.55000: Flags [.], ack 2, win 64, options [nop,nop,TS val 1841598836 ecr 4236579779], length 0
```

A interface usada foi `lo` (loopback). O arquivo tem 715 bytes e as linhas mostram o handshake (`[S]` / `[S.]`), o envio de dados (`[P.]`, 19 bytes do cliente para a 55000) e o encerramento (`[F.]`) — prova de que houve tráfego real na porta do experimento.

---

## Exercício 8 — Varredura controlada no host e validação cruzada

**Enunciado:** Execute `nmap` no próprio host para listar portas TCP abertas e registre o trecho. Identifique a porta do seu listener `nc` e comprove que apareceu como aberta. Cruze a evidência: use `lsof` ou `netstat` para provar qual processo manteve essa porta aberta.

**Comando:**

```bash
# Terminal A
nc -lvp 55000

# Terminal B
nmap -p- 127.0.0.1
lsof -i :55000
netstat -tlnp | grep 55000
```

**Resposta:**

```text
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-11-08 20:14 UTC
Nmap scan report for localhost (127.0.0.1)
Host is up (0.0000050s latency).
Not shown: 65533 closed tcp ports (reset)
PORT      STATE SERVICE
22/tcp    open  ssh
55000/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 0.73 seconds

COMMAND PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
nc      630 root    3u  IPv4  47701      0t0  TCP *:55000 (LISTEN)

tcp        0      0 0.0.0.0:55000           0.0.0.0:*               LISTEN      630/nc              
```

O nmap, vendo "de fora", listou a 55000 como `open`. O `lsof` e o `netstat`, vendo "de dentro", mostram que quem mantém essa porta é o `nc` (PID 630). As duas visões batem: a porta está realmente aberta e o processo dono é o esperado.

---

## Exercício 9 — Vulnerabilidade típica: serviço em claro e risco operacional

**Enunciado:** Use `nmap` ou `netstat` para verificar se portas típicas de serviços legados estão abertas (telnet 23 / ftp 21) e registre a evidência. Se não houver, registre a prova de "não exposto" e liste 2 motivos pelos quais isso é desejável. Registre uma mitigação mínima (2 itens) para servidor corporativo.

**Comando:**

```bash
nmap -p 21,23 127.0.0.1
netstat -tlnp | grep -E ":21|:23"
```

**Resposta:**

```text
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-11-08 20:12 UTC
Nmap scan report for localhost (127.0.0.1)
Host is up (0.000051s latency).

PORT   STATE  SERVICE
21/tcp closed ftp
23/tcp closed telnet

Nmap done: 1 IP address (1 host up) scanned in 0.09 seconds
```

O `netstat` não retornou nenhuma linha para 21/23 — confirmação local de que não há serviço escutando.

**Por que "não exposto" é desejável:**

1. Telnet e FTP trafegam credenciais em texto puro; qualquer captura de rede expõe usuário e senha.
2. São protocolos legados, sem criptografia nem verificação de integridade, e aumentam a superfície de ataque do host.

**Mitigação mínima (2 ações):**

1. Bloquear as portas 21 e 23 na borda (firewall / security group), liberando só o necessário.
2. Desinstalar ou desabilitar `telnetd` e o servidor FTP, usando SSH e SFTP/FTPS no lugar.

---

## Exercício 10 — SSH: mitigação mínima com prova e rollback

**Enunciado:** Faça backup do `sshd_config` e registre a evidência do arquivo. Aplique uma mitigação mínima (ex.: restringir root login) e registre a linha alterada. Reinicie o serviço, prove status ativo e prove que ainda consegue conectar via SSH e executar 1 comando remoto. Registre um rollback de 2 passos.

**Comando:**

```bash
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
ls -l /etc/ssh/sshd_config.bak

sshd -T | grep -E "permitrootlogin|passwordauthentication"

nano /etc/ssh/sshd_config     # troque "PermitRootLogin yes" por "PermitRootLogin no"

grep -n PermitRootLogin /etc/ssh/sshd_config
sshd -t

systemctl restart ssh
systemctl status ssh
ssh aluno@127.0.0.1 hostname
ssh root@127.0.0.1 hostname
```

**Rollback:**

```bash
cp /etc/ssh/sshd_config.bak /etc/ssh/sshd_config
systemctl restart ssh
```

**Resposta:**

```text
-rw-r--r-- 1 root root 3509 Nov  8 20:13 /etc/ssh/sshd_config.bak

ANTES:
permitrootlogin yes
passwordauthentication yes

LINHA ALTERADA:
42:PermitRootLogin no
sshd -t: OK

● ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/usr/lib/systemd/system/ssh.service; enabled; preset: enabled)
     Active: active (running) since Sun, 2026-11-08 20:13:58 UTC; 2s ago
       Docs: man:sshd(8)
             man:sshd_config(5)
    Process: 560 ExecStartPre=/usr/sbin/sshd -t (code=exited, status=0/SUCCESS)

ALUNO:
d6825dab8e97

ROOT:
root@127.0.0.1: Permission denied (publickey,password).
```

O backup foi criado, o estado anterior era `PermitRootLogin yes` e a linha 42 passou para `PermitRootLogin no`. O serviço reiniciou ativo, o usuário `aluno` continuou conectando e executando comando, e o login de root passou a ser recusado — mitigação aplicada sem perder acesso.

---

## Exercício 11 — Socket operacional: bind/listen/accept/send/recv

**Enunciado:** Refaça (ou reutilize) o experimento com `nc` (listener + cliente). Registre evidência de LISTEN (`netstat`) e do processo dono (`lsof`). Registre evidência do estado ESTABLISHED durante a conexão. Faça um mapeamento objetivo em 4 linhas: bind, listen, accept, send/recv.

**Comando:**

```bash
# Terminal A
nc -lvp 55000

# Terminal B (deixe aberto)
nc 127.0.0.1 55000

# Terminal C
netstat -tlnp | grep 55000
netstat -tnp | grep 55000
lsof -i :55000
```

**Resposta:**

```text
tcp        0      0 0.0.0.0:55000           0.0.0.0:*               LISTEN      376/nc              

tcp        0      0 127.0.0.1:34816         127.0.0.1:55000         TIME_WAIT   -                   
tcp        0      0 127.0.0.1:55000         127.0.0.1:44988         ESTABLISHED 376/nc              
tcp        0      0 127.0.0.1:44988         127.0.0.1:55000         ESTABLISHED 379/nc              
tcp        0      0 127.0.0.1:59964         127.0.0.1:55000         TIME_WAIT   -                   

COMMAND PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
nc      376 root    3u  IPv4  50303      0t0  TCP *:55000 (LISTEN)
nc      376 root    4u  IPv4  50307      0t0  TCP localhost:55000->localhost:44988 (ESTABLISHED)
nc      379 root    3u  IPv4  45495      0t0  TCP localhost:44988->localhost:55000 (ESTABLISHED)
```

- **bind:** o `lsof` mostra o `nc` (PID 376) como dono do socket na porta 55000.
- **listen:** o `netstat -tlnp` mostra `0.0.0.0:55000` em estado `LISTEN`.
- **accept:** o `netstat -tnp` mostra o par `127.0.0.1:55000 <-> 127.0.0.1:44988` em `ESTABLISHED`, ligando os PIDs 376 e 379.
- **send/recv:** a mensagem digitada no cliente apareceu no terminal do servidor e nos pacotes `[P.]` capturados no Exercício 7.

---

## Exercício 12 — Diagnóstico integrado: uma evidência por ferramenta

**Enunciado:** Inicie um listener `nc` em porta alta e conecte um cliente enviando mensagem. Colete uma evidência curta de cada ferramenta: `netstat` (LISTEN/ESTABLISHED), `lsof` (PID/processo dono), `nmap` (porta aberta) e `tcpdump` (linhas que provem tráfego). Finalize com uma conclusão de 3–5 linhas sobre o que as quatro evidências provam em conjunto.

**Comando:**

```bash
# Terminal A
nc -lvp 55000

# Terminal B (mantenha aberto)
nc 127.0.0.1 55000

# Terminal C
netstat -tnp | grep 55000
lsof -i :55000
nmap -p 55000 127.0.0.1
tcpdump -i lo port 55000 -c 6
```

> Digite no terminal B durante o `tcpdump` para gerar tráfego.

**Resposta:**

```text
== netstat ==
tcp        0      0 127.0.0.1:34816         127.0.0.1:55000         TIME_WAIT   -                   
tcp        0      0 127.0.0.1:55000         127.0.0.1:44988         TIME_WAIT   -                   
tcp        0      0 127.0.0.1:44988         127.0.0.1:55000         TIME_WAIT   -                   
tcp        0      0 127.0.0.1:44992         127.0.0.1:55000         ESTABLISHED 389/nc              
tcp        0      0 127.0.0.1:55000         127.0.0.1:44992         ESTABLISHED 386/nc              
tcp        0      0 127.0.0.1:59964         127.0.0.1:55000         TIME_WAIT   -                   

== lsof ==
COMMAND PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
nc      386 root    3u  IPv4  45515      0t0  TCP *:55000 (LISTEN)
nc      386 root    4u  IPv4  45519      0t0  TCP localhost:55000->localhost:44992 (ESTABLISHED)
nc      389 root    3u  IPv4  42753      0t0  TCP localhost:44992->localhost:55000 (ESTABLISHED)

== nmap ==
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-11-08 20:13 UTC
Nmap scan report for localhost (127.0.0.1)
Host is up (0.000061s latency).

PORT      STATE SERVICE
55000/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 0.07 seconds

== tcpdump ==
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on lo, link-type EN10MB (Ethernet), snapshot length 262144 bytes
20:13:07.129476 IP localhost.44998 > localhost.55000: Flags [S], seq 123059748, win 65495, options [mss 65495,sackOK,TS val 1630559080 ecr 0,nop,wscale 10], length 0
20:13:07.129491 IP localhost.55000 > localhost.44998: Flags [S.], seq 1978173996, ack 123059749, win 65483, options [mss 65495,sackOK,TS val 3238814383 ecr 1630559080,nop,wscale 10], length 0
20:13:07.129505 IP localhost.44998 > localhost.55000: Flags [.], ack 1, win 64, options [nop,nop,TS val 1630559080 ecr 3238814383], length 0
20:13:07.129560 IP localhost.44998 > localhost.55000: Flags [P.], seq 1:17, ack 1, win 64, options [nop,nop,TS val 1630559080 ecr 3238814383], length 16
20:13:07.129566 IP localhost.55000 > localhost.44998: Flags [.], ack 17, win 64, options [nop,nop,TS val 3238814383 ecr 1630559080], length 0
20:13:12.127502 IP localhost.44998 > localhost.55000: Flags [F.], seq 17, ack 1, win 64, options [nop,nop,TS val 1630564079 ecr 3238814383], length 0
6 packets captured
12 packets received by filter
0 packets dropped by kernel
```

O `netstat` mostra a 55000 em `LISTEN` e o par `127.0.0.1:55000 <-> 127.0.0.1:44992` em `ESTABLISHED`, provando que existe um socket servidor e uma conexão ativa. O `lsof` amarra esse socket ao processo `nc` (PID 386 no listen, 389 no cliente), identificando quem é o dono. O `nmap` confirma, de fora, que a porta está `open`. O `tcpdump` mostra o handshake (`[S]` / `[S.]` / `[.]`), o envio de 16 bytes (`[P.]`) e o encerramento (`[F.]`). Juntas, as quatro evidências provam a mesma coisa por ângulos diferentes: o serviço está escutando, tem dono conhecido, está alcançável pela rede e realmente trocou bytes.
