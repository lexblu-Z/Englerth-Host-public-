# Política e Arquitetura de Segurança — Englerth Host (SECURITY.md)

A segurança e a integridade do computador pessoal do usuário são prioridades máximas e inegociáveis no **Englerth Host**. Este documento formaliza as práticas de confinamento, auditoria e proteção implementadas no código.

---

## 1. Princípios Invioláveis de Segurança (Seção 4 da Especificação)

1. **Proteção de Arquivos Pessoais:** O Englerth Host **NUNCA** apaga, move ou modifica jogos existentes, launchers (CurseForge, Prism, Modrinth), mundos pessoais ou documentos fora de sua própria pasta de dados (`%LOCALAPPDATA%\EnglerthHost`).
2. **Exclusões com Confirmação e Lixeira Interna:** Operações destrutivas (excluir um servidor) exigem confirmação explícita digitando o nome do servidor. Arquivos excluídos no gerenciador de arquivos são enviados primeiro para uma lixeira interna (`trash/`) com retenção e carimbo de data/hora.
3. **Respeito ao Windows Defender e Firewall:** O aplicativo **NUNCA desativa o Windows Defender, antivírus ou o Firewall do Windows**. Não são criadas exceções globais ou silenciosas no Defender.
4. **Sem Alterações no Sistema Operacional:** O Englerth Host **NÃO altera planos de energia do Windows, configurações da BIOS ou chaves do Registro do Windows**.
5. **Sem UPnP ou Abertura Automática de Portas:** O aplicativo não envia comandos UPnP ao roteador do usuário e não tenta abrir portas sem o conhecimento prévio e a aprovação explícita do administrador.
6. **Firewall com UAC e Escopo Mínimo:** Nenhuma regra de firewall é aplicada silenciosamente. Regras de firewall são geradas para cópia ou execução explícita via PowerShell com UAC, restritas à porta do servidor, ao executável Java gerenciado e ao perfil de rede Privada.
7. **Downloads Exclusivos de Fontes Oficiais:** Todos os downloads de binários (Java, JARs de servidor, plugins, mods) provêm exclusivamente de servidores e CDNs oficiais verificadas (Mojang, Adoptium, PaperMC, PurpurMC, FabricMC, Modrinth, GeyserMC, Microsoft) e passam por conferência criptográfica de checksum SHA-1 / SHA-256.

---

## 2. Confinamento em Sandbox do Gerenciador de Arquivos

O gerenciamento de arquivos em `crates/hostd/src/files/mod.rs` implementa mecanismos rígidos contra ataques de *Path Traversal*:
- **Canonicalização Estrita:** Todo caminho solicitado pelo usuário é normalizado através de chamadas ao sistema de arquivos (`canonicalize()`). O caminho absoluto resultante deve obrigatoriamente iniciar com a raiz canônica do diretório do servidor (`canonical_path.starts_with(canonical_server_root)`).
- **Bloqueio de `..` e Links Simbólicos:** Tentativas de escape de diretório utilizando sequências relativas `../` ou junctions/symlinks que apontem para arquivos fora da pasta do servidor são interceptadas e rejeitadas com erro `400 Bad Request`.

---

## 3. Isolamento do RCON e Credenciais Criptográficas

- **Bind Estrito em Loopback:** A interface de administração remota do Minecraft (RCON) é vinculada unicamente à interface de loopback `127.0.0.1`. Ela nunca é exposta na rede local ou na internet.
- **Senhas Fortes Aleatórias:** Cada servidor criado recebe uma senha RCON pseudoaleatória de alta entropia gerada pelo daemon e armazenada localmente de forma isolada.
- **Armazenamento Seguro de Tokens:** Tokens de acesso e chaves de API externas (ex: CurseForge) são armazenados em arquivos com permissões de usuário ou na API DPAPI do Windows Credential Manager.

---

## 4. Prevenção de Crash Loops e Exaustão de Recursos

- **Proteção contra Crash Loops:** Se um servidor falhar de forma repetitiva (ex: por um mod defeituoso), o supervisor do Englerth Host permite no máximo 3 reinícios automáticos em uma janela deslizante de 10 minutos. Se o limite for excedido, o servidor transiciona para o estado `Failed`, prevenindo que a CPU e o disco sejam consumidos em um ciclo infinito.
- **Guarda RAII `SaveOnGuard`:** Durante a criação de backups a quente, o salvamento automático do mundo é pausado (`save-off`). Para garantir que o mundo nunca permaneça desprotegido em caso de erro, interrupção ou pânico, o Englerth Host utiliza uma guarda RAII cujo destrutor executa incondicionalmente `save-on`.

---

## 5. Trilha de Auditoria (`audit_log`)

Todas as ações com privilégios ou sensíveis para a segurança do computador são registradas na tabela `audit_log` do banco de dados relacional SQLite local:
- Geração de regras do Firewall do Windows (`FIREWALL_RULE_GENERATED`).
- Criação e exclusão de regras de Tarefa Agendada / Autostart (`AUTOSTART_CONFIGURED`).
- Ativação ou desativação do Modo Jogo (`GAMEMODE_CONFIGURED`).
- Exclusão ou restauração de backups e arquivos do servidor.
- Cada entrada armazena o timestamp UTC, a ação, o alvo, os detalhes do comando e o status de sucesso.

---

## 6. Como Reportar Vulnerabilidades

Caso você descubra uma vulnerabilidade de segurança potencial no Englerth Host, solicitamos que nos informe de forma responsável:

1. **Canal de Contato:** Envie um e-mail para `leonardoessouza@gmail.com` com o assunto `[SECURITY] Englerth Host - Relatório de Vulnerabilidade`.
2. **Conteúdo:** Descreva os passos para reprodução, versão do aplicativo, ambiente Windows e o impacto potencial.
3. **Divulgação Responsável:** Por favor, não publique detalhes da falha em *Issues* públicas do GitHub antes que a equipe tenha tempo hábil para investigar e disponibilizar uma correção.
4. **Prazo de Resposta:** Nos esforçamos para acusar o recebimento em até 48 horas úteis e manter o relator informado sobre o progresso da correção.

---

&copy; 2026 Leonardo Englerth. All rights reserved.
