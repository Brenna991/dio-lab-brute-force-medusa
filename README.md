# Simulando um Ataque de Força Bruta com Medusa e Kali Linux

Esse repositório é a minha documentação do desafio do curso "Criando um Ataque Brute Force de senhas com Medusa e Kali Linux", da DIO. A ideia aqui não é só colar comando e print — é registrar o que eu entendi sobre como ataques de força bruta funcionam na prática, por que eles ainda são um problema real em 2026, e o que dá pra fazer pra se defender deles.

Ainda estou no começo dos meus estudos em segurança, então trato esse README também como um caderno de anotações organizado — se algo estiver simplificado demais, é porque foi assim que eu consegui entender primeiro.

---

## 🎯 O que eu me propus a estudar

A aula trouxe três cenários clássicos de ataque de força bruta:

- Um serviço de **FTP**, onde as credenciais trafegam sem criptografia;
- Um compartilhamento **SMB**, onde dá pra testar o conceito de *password spraying*;
- Um **formulário de login web** (no caso, o DVWA), onde o ataque precisa ser adaptado para entender os campos HTML.

Em todos os três, usei o **Medusa** como ferramenta principal, por ser o que a aula apresentou, e o **Nmap** para o reconhecimento inicial dos serviços.

---

## 🏗️ Ambiente usado como referência

O cenário proposto pela aula monta duas VMs no VirtualBox, numa rede host-only isolada (sub-rede `192.168.56.0/24`), pra garantir que nada disso encoste na internet:

| Papel | Sistema | IP de referência | O que faz |
| :--- | :--- | :--- | :--- |
| Atacante | Kali Linux | `192.168.56.101` | Roda o Nmap e o Medusa |
| Alvo | Metasploitable 2 | `192.168.56.102` | Hospeda os serviços vulneráveis (FTP, Samba, DVWA) |

Documentei os comandos com base nesse ambiente de referência e na sintaxe oficial do Medusa, estudando cada módulo antes de escrever o porquê de cada flag — não é um print de execução real, é o meu estudo de como cada ataque seria montado.

---

## 🛠️ Entendendo o Medusa

Antes de sair digitando comando, precisei entender pra que servia cada parâmetro. Resumi assim:

| Parâmetro | Pra que serve | Exemplo |
| :--- | :--- | :--- |
| `-h` | IP ou host do alvo | `-h 192.168.56.102` |
| `-u` / `-U` | Um usuário único / uma lista de usuários | `-U wordlists/usuarios.txt` |
| `-p` / `-P` | Uma senha única / uma lista de senhas | `-P wordlists/senhas.txt` |
| `-M` | Qual módulo (protocolo) vai ser atacado | `-M ftp`, `-M smbnt`, `-M web-form` |
| `-m` | Parâmetros extras que cada módulo pede | `-m DIR:/dvwa/login.php` |
| `-t` | Quantas tentativas simultâneas (threads) | `-t 4` |
| `-f` | Para assim que encontra o primeiro login válido | `-f` |

---

## 🔍 Etapa 1 — Reconhecimento com Nmap

Antes de atacar qualquer coisa, o primeiro passo é saber o que está rodando no alvo:

```bash
nmap -sV -p 21,80,139,445 192.168.56.102
```

No cenário da aula, isso mapeia:
- **Porta 21 (FTP):** `vsftpd 2.3.4`
- **Porta 80 (HTTP):** Apache hospedando o DVWA
- **Portas 139/445 (SMB):** Samba

Essa etapa deixou claro pra mim por que reconhecimento vem sempre antes do ataque: sem saber qual serviço e qual versão está ativa, não dá pra escolher o módulo certo do Medusa.

---

## ⚔️ Etapa 2 — Os três cenários

### Cenário 1: Força bruta em FTP

```bash
medusa -h 192.168.56.102 -U wordlists/usuarios.txt -P wordlists/senhas.txt -M ftp -t 4 -f
```

Esse foi o mais fácil de entender: o Medusa testa cada combinação de usuário e senha das wordlists contra o FTP, usando 4 tentativas em paralelo, e para assim que acha a primeira combinação válida. O FTP tradicional não criptografa nada, então esse tipo de ataque funciona bem contra ele.

### Cenário 2: Password spraying em SMB

```bash
medusa -h 192.168.56.102 -U wordlists/usuarios.txt -p msfadmin -M smbnt
```

Esse conceito eu não conhecia antes da aula: ao invés de testar várias senhas contra um usuário só (o que pode bloquear a conta rápido), o *password spraying* testa **uma senha só** contra **vários usuários**. É uma técnica mais "discreta", porque evita disparar o bloqueio de tentativas de uma conta específica.

### Cenário 3: Automação de tentativas no formulário web (DVWA)

```bash
medusa -h 192.168.56.102 -u admin -P wordlists/senhas.txt -M web-form \
  -m DIR:/dvwa/login.php \
  -m FORM:"username=^USER^&password=^PASS^&Login=Login" \
  -m DENY-SIGNAL:"Login failed"
```

Confesso que esse foi o que mais me deu trabalho pra entender. Diferente do FTP e do SMB, aqui não basta apontar o IP — é preciso mapear manualmente os campos do formulário HTML:
- `DIR` é o caminho da página de login;
- `FORM` reproduz o corpo da requisição POST, trocando `^USER^` e `^PASS^` pelos valores das listas;
- `DENY-SIGNAL` é o texto que a página devolve quando o login falha — é assim que o Medusa sabe diferenciar uma tentativa certa de uma errada.

Entender essa lógica me ajudou a enxergar por que formulários web mal configurados (sem CAPTCHA, sem limite de tentativas) são um alvo tão comum.

---

## 🛡️ Etapa 3 — Como se defender disso

Estudando os três ataques, consegui listar as defesas que fazem mais sentido pra cada um:

1. **Bloqueio de conta (account lockout)** — travar a conta depois de poucas tentativas erradas (3 a 5, por exemplo).
2. **Autenticação multifator (MFA)** — mesmo que a senha vaze, só ela não é suficiente pra entrar.
3. **Rate limiting / Fail2Ban** — bloquear automaticamente IPs que estão gerando muitas tentativas de login.
4. **Trocar protocolos inseguros** — tirar o FTP tradicional de uso (ir pra SFTP/FTPS) e desabilitar versões antigas de SMB.
5. **CAPTCHA e token anti-CSRF** em formulários web, pra dificultar automação.
6. **Política de senha forte** — senha longa, complexa, e que não esteja em listas de vazamentos conhecidos.

---

## 📝 O que eu levo desse desafio

O que mais ficou pra mim foi entender que força bruta não é sobre "adivinhar senha na sorte" — é sobre explorar a ausência de controles básicos (bloqueio de tentativas, MFA, rate limiting). Os três cenários mostraram isso de formas diferentes: o FTP por falta de criptografia, o SMB pela possibilidade de enumerar usuários e testar em massa, e o formulário web por falta de proteção contra automação.

Pretendo, nos próximos estudos, tentar rodar esses cenários de fato num ambiente de laboratório (Kali + Metasploitable2 ou um lab online), pra comparar a teoria com a prática.

---

## ⚠️ Aviso legal

Todo o conteúdo e os comandos documentados neste repositório têm finalidade estritamente educacional, voltados à prática em ambiente controlado e autorizado (laboratório próprio ou plataformas de treinamento), sem qualquer intenção de uso contra sistemas reais sem permissão expressa do proprietário. Reproduzir esses comandos contra sistemas de terceiros, sem autorização, é crime.