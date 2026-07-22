<div align="center">

# 🛡️ Home Lab SOC — Wazuh SIEM na Nuvem

### Construindo um Security Operations Center do zero: Wazuh + GCP + simulação de ataques

![Wazuh](https://img.shields.io/badge/SIEM-Wazuh_4.14-1A73E8?style=for-the-badge&logo=wazuh&logoColor=white)
![GCP](https://img.shields.io/badge/Cloud-Google_Cloud-4285F4?style=for-the-badge&logo=googlecloud&logoColor=white)
![Ubuntu](https://img.shields.io/badge/OS-Ubuntu_22.04-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![Windows](https://img.shields.io/badge/OS-Windows-0078D6?style=for-the-badge&logo=windows&logoColor=white)
![Status](https://img.shields.io/badge/Status-Em_andamento-yellow?style=for-the-badge)

</div>

---

## 📌 Sobre o projeto

Este repositório documenta a construção de um **Home Lab de SOC (Security Operations Center)**, do provisionamento da infraestrutura até a simulação de eventos de segurança reais, com o objetivo de praticar:

🔍 Instalação e administração de um SIEM (**Wazuh**)
🖥️ Deployment de agentes em múltiplos sistemas operacionais
🎛️ Calibração de regras de coleta para reduzir ruído e focar em telemetria relevante
⚔️ Simulação de técnicas de ataque mapeadas ao **MITRE ATT&CK**

> 💡 Ambiente 100% funcional, construído com recursos gratuitos/créditos de nuvem — ideal para portfólio de segurança ofensiva/defensiva.

---

## 🗂️ Índice

| | Seção |
|---|---|
| 🏗️ | [Arquitetura](#️-arquitetura) |
| ☁️ | [1. Provisionamento da infraestrutura](#️-1-provisionamento-da-infraestrutura-gcp) |
| ⚙️ | [2. Instalação do Wazuh Manager](#️-2-instalação-do-wazuh-manager-all-in-one) |
| 🔥 | [3. Rede e Firewall](#-3-configuração-de-rede-e-firewall) |
| 🐧 | [4. Agente — Endpoint Linux](#-4-instalação-do-agente--endpoint-linux) |
| 🪟 | [5. Agente — Endpoint Windows](#-5-instalação-do-agente--endpoint-windows) |
| 🎚️ | [6. Calibração das regras](#️-6-calibração-das-regras-de-coleta-redução-de-ruído) |
| 🧪 | [7. Simulação de ataques](#-7-simulação-de-eventos-de-segurança) |
| 🐛 | [8. Troubleshooting](#-8-lições-aprendidas--troubleshooting) |
| 🚀 | [Próximos passos](#-próximos-passos) |

---

## 🏗️ Arquitetura

```
                        ┌───────────────────────────────┐
                        │       Wazuh Manager (GCP)     │
                        │   • Wazuh Indexer             │
                        │   • Wazuh Manager             │
                        │   • Wazuh Dashboard           │
                        │  Ubuntu 22.04 | e2-standard-2 │
                        └────────────────┬──────────────┘
                                         │
                    ┌────────────────────┼─────────────────────┐
                    │                                          │
         ┌──────────┴───────────┐                  ┌───────────┴────────────┐
         │    Endpoint Linux    │                  │    Endpoint Windows    │
         │  GCP · rede interna  │                  │  VirtualBox · local    │
         │  Ubuntu 22.04        │                  │  Windows 10/11         │
         └──────────────────────┘                  └────────────────────────┘
```

<div align="center">

| Componente | 🖥️ Máquina | 💽 SO | 📍 Localização |
|:---|:---:|:---:|:---:|
| **wazuh-manager** | e2-standard-2 (2 vCPU / 8GB) | Ubuntu 22.04 LTS | ☁️ GCP |
| **endpoint-linux** | e2-small (2 vCPU / 2GB) | Ubuntu 22.04 LTS | ☁️ GCP |
| **endpoint-windows** | 2 vCPU / 4GB | Windows 10/11 | 🏠 Local (VirtualBox) |

**Versão do Wazuh:** `4.14.6` &nbsp;|&nbsp; **Região GCP:** `southamerica-east1` (São Paulo)

</div>

---

## ☁️ 1. Provisionamento da infraestrutura (GCP)

### 🤔 Por que GCP?

O lab foi originalmente planejado para **Oracle Cloud Free Tier**, mas esbarrou em indisponibilidade recorrente de capacidade na shape gratuita `VM.Standard.A1.Flex`:

> ⚠️ `Out of capacity for shape VM.Standard.A1.Flex in availability domain AD-1`

Um problema conhecido e recorrente da plataforma em regiões de alta demanda. A solução foi migrar para o **Google Cloud Platform**, aproveitando os **$300 em créditos gratuitos** (90 dias) para novas contas.

### 🖥️ VM do Manager

| Campo | Valor |
|---|---|
| Nome | `wazuh-manager` |
| Região | `southamerica-east1` |
| Machine type | `e2-standard-2` (2 vCPU / 8GB RAM) |
| Boot disk | Ubuntu 22.04 LTS · 50GB |
| Firewall | `Allow HTTPS` + `Allow HTTP` |

> ℹ️ Iniciado com `e2-medium` (4GB), mas a instalação falhou por falta de recursos — [ver troubleshooting](#-8-lições-aprendidas--troubleshooting).

### 🐧 VM do endpoint Linux

| Campo | Valor |
|---|---|
| Nome | `endpoint-linux` |
| Região | `southamerica-east1` (mesma VPC do manager) |
| Machine type | `e2-small` (2 vCPU / 2GB RAM) |
| Boot disk | Ubuntu 22.04 LTS · 20GB |

> 💻 Acesso via **SSH no navegador**, direto pelo console do GCP — sem gerenciar chaves `.pem`.

<img width="906" height="727" alt="09-ssh-terminal" src="https://github.com/user-attachments/assets/3b49180b-6b41-425b-897d-ea37f893ff25" />

---

## ⚙️ 2. Instalação do Wazuh Manager (all-in-one)

```bash
# 🔄 Atualizar o sistema
sudo apt update && sudo apt upgrade -y

# ⬇️ Baixar o instalador (versão fixa, evita instabilidade do alias /4.x/)
wget https://packages.wazuh.com/4.14/wazuh-install.sh

# 🚀 Instalação all-in-one: Indexer + Manager + Dashboard
sudo bash wazuh-install.sh -a
```

<img width="767" height="187" alt="01-wget-download" src="https://github.com/user-attachments/assets/9db44dc9-0047-48e8-8bad-74caa2c1d9a3" />

Ao final, o script exibe as credenciais:

```
--- Summary ---
✅ You can access the web interface https://<wazuh-dashboard-ip>:443
👤 User: admin
🔑 Password: <senha gerada automaticamente>
```

<img width="765" height="716" alt="02-install-summary" src="https://github.com/user-attachments/assets/4fef547a-2fa6-469c-9534-0899bbf77610" />


> 🚨 **Atenção:** a senha só aparece uma vez — copie e guarde em local seguro imediatamente.

🌐 **Acesso ao dashboard:** `https://<IP_EXTERNO_DA_VM>` *(aceitar o aviso de certificado autoassinado)*

<img width="1536" height="692" alt="03-dashboard-login" src="https://github.com/user-attachments/assets/57f652f6-c700-44fe-b729-9c5537401ef0" />


### 🖥️ Visão geral do Dashboard

Após o login, a tela inicial (*Overview*) resume o estado do ambiente: agentes registrados, alertas por severidade nas últimas 24h, e atalhos para os principais módulos (FIM, Malware Detection, Threat Hunting, MITRE ATT&CK, Vulnerability Detection, entre outros):

<img width="1533" height="696" alt="04-dashboard-overview" src="https://github.com/user-attachments/assets/d96d5d82-be96-40b9-b26e-380b1cd5fa58" />


O Wazuh também expõe módulos prontos de compliance (PCI DSS, GDPR, HIPAA, NIST 800-53) e integrações de nuvem (AWS, GCP, Docker, Office 365, GitHub) — não utilizados neste lab, mas disponíveis para expansão futura:

<img width="1532" height="627" alt="05-dashboard-modules" src="https://github.com/user-attachments/assets/4ec2a899-c238-486c-8fcd-452a83463881" />

---

## 🔥 3. Configuração de rede e firewall

Regra criada no GCP para permitir a comunicação dos agentes com o manager:

<div align="center">

| Campo | Valor |
|---|---|
| **Nome** | `allow-wazuh-agents` |
| **Direção** | Ingress |
| **Alvo** | Todas as instâncias na rede |
| **Origem** | `0.0.0.0/0` |
| **Protocolo/portas** | `tcp: 1514, 1515` |

</div>

- 🔌 **1514/tcp** — comunicação de eventos do agente com o manager
- 📝 **1515/tcp** — enrollment (registro) do agente

<img width="558" height="552" alt="07-firewall-rule" src="https://github.com/user-attachments/assets/947d3038-0e6e-4331-a96c-c191e79efa15" />

---

## 🐧 4. Instalação do agente — Endpoint Linux

```bash
# ⬇️ Baixar o pacote do agente (mesma versão do manager)
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.6-1_amd64.deb

# 📡 Instalar apontando para o IP interno do manager
sudo WAZUH_MANAGER='<IP_INTERNO_MANAGER>' dpkg -i ./wazuh-agent_4.14.6-1_amd64.deb

# ▶️ Habilitar e iniciar o serviço
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent

# 🔍 Verificar conexão
sudo tail -f /var/ossec/logs/ossec.log
```

✅ Log esperado de sucesso:
```
INFO: Trying to connect to server ([IP_MANAGER]:1514/tcp).
INFO: Connected to the server ([IP_MANAGER]:1514/tcp)
```

📊 Confirmação visual: **Dashboard → Agents → status `Active`**

---

## 🪟 5. Instalação do agente — Endpoint Windows

> 🏠 Endpoint executado localmente em VirtualBox, conectando-se ao manager na nuvem via **IP externo**.

> 📝 *Nota: durante o planejamento, chegou-se a avaliar um endpoint Windows Server na própria nuvem — o GCP oferece diversas versões (2019 a 2025), mas VMs Windows exigem upgrade de billing na conta, então a opção final foi manter esse endpoint localmente.*

<img width="490" height="317" alt="06-windows-server-selection" src="https://github.com/user-attachments/assets/00d6b722-a0cb-4823-a7b8-8a7e42c7b8f9" />


```powershell
# ⬇️ Download do instalador MSI (mesma versão do manager)
# https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.6-1.msi

# 🚀 Instalação silenciosa, apontando para o IP externo do manager
msiexec.exe /i wazuh-agent-4.14.6-1.msi /q WAZUH_MANAGER='<IP_EXTERNO_MANAGER>'

# ▶️ Iniciar o serviço
NET START WazuhSvc

# 🔍 Verificar conexão
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" -Tail 20 -Wait
```

---

## 🎚️ 6. Calibração das regras de coleta (redução de ruído)

> 🎯 **Objetivo:** manter apenas telemetria relevante para detecção de segurança, cortando eventos de baixo valor analítico.

### 🕵️ 6.1 File Integrity Monitoring (FIM)

Monitoramento em **tempo real** habilitado, incluindo `/root` e `/home` (relevantes para testes de escalada de privilégio):

```bash
sudo sed -i 's|<directories>/etc,/usr/bin,/usr/sbin</directories>|<directories realtime="yes">/etc,/usr/bin,/usr/sbin</directories>|' /var/ossec/etc/ossec.conf

sudo sed -i 's|<directories>/bin,/sbin,/boot</directories>|<directories realtime="yes">/bin,/sbin,/boot</directories>\n    <directories realtime="yes">/root</directories>\n    <directories realtime="yes" report_changes="yes">/home</directories>|' /var/ossec/etc/ossec.conf
```

```xml
<directories realtime="yes">/etc,/usr/bin,/usr/sbin</directories>
<directories realtime="yes">/bin,/sbin,/boot</directories>
<directories realtime="yes">/root</directories>
<directories realtime="yes" report_changes="yes">/home</directories>
```

### 🚫 6.2 Security Configuration Assessment (SCA) — desativado

Gera alto volume de eventos de *compliance* (benchmark CIS), fora do foco de um lab orientado a detecção de ataques:

```bash
sudo sed -i '/<sca>/,/<\/sca>/ s|<enabled>yes</enabled>|<enabled>no</enabled>|' /var/ossec/etc/ossec.conf
```

### ✂️ 6.3 Rootcheck — reduzido aos checks essenciais

```bash
sudo sed -i \
  -e 's|<check_dev>yes</check_dev>|<check_dev>no</check_dev>|' \
  -e 's|<check_pids>yes</check_pids>|<check_pids>no</check_pids>|' \
  -e 's|<check_ports>yes</check_ports>|<check_ports>no</check_ports>|' \
  -e 's|<check_if>yes</check_if>|<check_if>no</check_if>|' \
  /var/ossec/etc/ossec.conf
```

<div align="center">

| Check | Status | Justificativa |
|:---:|:---:|---|
| `check_files` | ✅ ativo | Detecta arquivos conhecidos de rootkits |
| `check_trojans` | ✅ ativo | Detecta binários trojanizados |
| `check_sys` | ✅ ativo | Anomalias gerais do sistema |
| `check_dev` | ❌ desativado | Baixo valor de detecção no escopo do lab |
| `check_pids` | ❌ desativado | Alto custo de processamento, baixo retorno |
| `check_ports` | ❌ desativado | Detecta apenas portas ocultas por rootkit — **não detecta port scan externo** |
| `check_if` | ❌ desativado | Modo promíscuo de interface — fora do escopo atual |

</div>

### ♻️ Aplicando as mudanças

```bash
sudo systemctl restart wazuh-agent
sudo tail -f /var/ossec/logs/ossec.log
```

---

## 🧪 7. Simulação de eventos de segurança

### ✅ 7.1 Teste de File Integrity Monitoring

```bash
sudo touch /etc/teste-fim.txt
```

📸 **Resultado no dashboard** (*FIM: Recent events*):

<div align="center">

| 🕐 Time | 📁 Path | ⚡ Action | 📋 Rule description | 🔢 Level | 🆔 Rule ID |
|---|---|:---:|---|:---:|:---:|
| Jul 15, 2026 @ 10:23:29 | `/etc/teste-fim.txt` | `added` | File added to the system | 5 | 554 |

</div>

<img width="1492" height="178" alt="08-fim-event" src="https://github.com/user-attachments/assets/e3f26cb9-c751-4d57-af72-46e657c6f704" />


✅ Detecção confirmada em **tempo real**, segundos após a criação do arquivo.

### 🎯 7.2 Outros cenários simulados *(em andamento)*

- [x] 🔓 [Brute force SSH (tentativas de login falho) — detecção + bloqueio automático via Active Response](https://github.com/vick2715/test_01_ssh_bruteforce-homelab-wazuh-SIEM)
- [x] 🔺 [Escalada de privilégio via `sudo` (backdoor em `/etc/sudoers.d/`) — detecção via FIM + análise de logs de sudo](https://github.com/vick2715/test-02-privilege-escalation)
- [ ] ⏰ Persistência via cron job malicioso
- [ ] 🦠 Teste EICAR (arquivo de teste padrão de antivírus)
- [ ] 🌐 Port scan com Nmap
- [ ] 🤖 Emulação de adversário com Atomic Red Team (MITRE ATT&CK)

> 📝 Seção em atualização contínua conforme os testes são executados.

---

## 🐛 8. Lições aprendidas / troubleshooting

<details>
<summary><strong>☁️ Oracle Cloud — indisponibilidade de capacidade Always Free</strong></summary>
<br>

**Erro:** `Out of capacity for shape VM.Standard.A1.Flex in availability domain AD-1`

Problema recorrente e documentado da Oracle Cloud Free Tier em regiões de alta demanda (ex: São Paulo). Não foi possível contornar via troca de *availability domain* ou subscrição de região adicional (bloqueada para o tipo de conta utilizado).

**✅ Solução:** migração para GCP.
</details>

<details>
<summary><strong>⚙️ Erro no instalador: <code>User wazuh is not registered in Wazuh API</code></strong></summary>
<br>

Erro conhecido do instalador all-in-one, geralmente causado por uma condição de corrida (*race condition*) entre a inicialização do Wazuh Manager e o registro do usuário da API, agravada por hardware insuficiente.

**✅ Solução:**
1. Upgrade da VM de `e2-medium` (4GB) para `e2-standard-2` (8GB)
2. Limpeza completa antes de reinstalar:

```bash
sudo apt remove --purge wazuh-manager wazuh-indexer wazuh-dashboard filebeat -y
sudo rm -rf /var/ossec /etc/filebeat /usr/share/filebeat /etc/wazuh-indexer /var/lib/wazuh-indexer /etc/wazuh-dashboard /usr/share/wazuh-dashboard
sudo rm -rf wazuh-install-files.tar
sudo bash wazuh-install.sh -a
```
</details>

<details>
<summary><strong>🌐 URL genérica <code>/4.x/</code> retornando <code>403 AccessDenied</code></strong></summary>
<br>

A URL `https://packages.wazuh.com/4.x/wazuh-install.sh` apresentou instabilidade via `curl`.

**✅ Solução:** fixar a versão exata na URL (`.../4.14/wazuh-install.sh`) e usar `wget` como alternativa ao `curl`.
</details>

<details>
<summary><strong>🪟 VMs Windows não incluídas no modo GCP Free Trial</strong></summary>
<br>

O GCP exige upgrade da conta para faturamento completo para provisionar VMs Windows, mesmo com créditos gratuitos disponíveis.

**✅ Solução:** endpoint Windows mantido localmente em VirtualBox, conectando ao manager na nuvem via IP externo (portas 1514/1515 liberadas no firewall do GCP).
</details>

---

## 🚀 Próximos passos

- [ ] 🧪 Completar simulação de todos os cenários de ataque da seção 7.2
- [ ] 🌐 Instalar e integrar o **Suricata** (IDS de rede) ao Wazuh, para detecção de port scans
- [ ] 📐 Criar regras customizadas em `local_rules.xml` (ex: alerta de múltiplas tentativas de login falho)
- [ ] 🤖 Explorar o **Atomic Red Team** para emulação de adversário mapeada ao MITRE ATT&CK
- [ ] 📊 Documentar dashboards e queries customizadas do Wazuh Dashboard

---

<div align="center">

## 📚 Referências

[![Wazuh Docs](https://img.shields.io/badge/📖-Wazuh_Documentation-1A73E8?style=flat-square)](https://documentation.wazuh.com/)
[![MITRE ATT&CK](https://img.shields.io/badge/🎯-MITRE_ATT%26CK-red?style=flat-square)](https://attack.mitre.org/)
[![Atomic Red Team](https://img.shields.io/badge/⚛️-Atomic_Red_Team-orange?style=flat-square)](https://github.com/redcanaryco/atomic-red-team)

---

⭐ *Se este projeto foi útil, considere deixar uma estrela no repositório!*

</div>
