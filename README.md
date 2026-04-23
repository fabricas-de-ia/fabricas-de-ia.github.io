# fabricas-de-ia.github.io

## Runbook: GitHub Enterprise Server (GHES) no Azure + Entra ID

Este repositório agora inclui um guia rápido para finalizar a configuração do **GHES hospedado no Azure** no cenário informado (VM `githubserver`, imagem GitHub Enterprise).

### 1) Validar pré-requisitos de infraestrutura

- Confirmar DNS público para o GHES (ex.: `github.seudominio.com`) apontando para IP público da VM/LB.
- Abrir apenas portas necessárias no NSG para tráfego de entrada:
  - `22/tcp` (admin via SSH, restrito por IP)
  - `80/tcp` (apenas para redirect, opcional)
  - `443/tcp` (acesso web/API)
  - `8443/tcp` (Management Console para setup/administração, restrita por IP e/ou VPN)
- Garantir conectividade de saída para serviços externos necessários:
  - SMTP relay: `25/tcp` ou `587/tcp`, conforme o servidor/serviço de e-mail utilizado
- Anexar disco de dados dedicado para armazenamento de repositórios e ações (produção).
- Evitar VM Spot para produção (risco de eviction). Preferir instância regular.
- Ajustar tamanho da VM conforme usuários/ações (D2ls costuma ser apenas laboratório).

### 2) Primeira configuração do GHES

1. Acessar `https://<hostname-do-ghes>/setup/start`.
2. Aplicar licença GHES.
3. Definir hostname público definitivo.
4. Configurar certificado TLS válido (CA pública ou corporativa).
5. Configurar SMTP para convites/notificações.
6. Salvar e aguardar `config-apply`.

### 3) Integrar autenticação com Azure Entra ID (SAML)

No **Azure Entra ID**:
1. Criar Enterprise Application (SAML).
2. Definir:
   - Identifier (Entity ID): `https://<hostname-do-ghes>`
   - Reply URL (ACS): `https://<hostname-do-ghes>/saml/consume`
   - Sign-on URL: `https://<hostname-do-ghes>/login/saml`
3. Configurar claims:
   - `NameID` (email ou UPN estável)
   - `username`
   - `groups` (opcional, para autorização por grupo)
4. Baixar metadados/fingerprint e certificado de assinatura.
5. Atribuir usuários/grupos piloto.

No **GHES Management Console**:
1. Ir em **Authentication** → habilitar **SAML**.
2. Inserir:
   - Sign on URL (Login URL do Entra)
   - Issuer (Entra Identifier)
   - Certificado X.509 do Entra
3. Validar mapeamento de atributos.
4. Salvar e aplicar configuração.

### 4) Endurecimento (hardening) recomendado

- Exigir 2FA (se a política de identidade permitir).
- Restringir acesso administrativo por IP (NSG + firewall corporativo).
- Habilitar logs/auditoria e retenção centralizada.
- Configurar backup/restore com GitHub Backup Utilities.
- Definir política de atualização do GHES (janela mensal/trimestral).
- Não armazenar chaves privadas, tokens, senhas ou outros segredos em documentação pública.

### 5) Piloto e validação

- Testar login SAML com 3 perfis: admin, developer, read-only.
- Validar criação de repositório, clone/push, PR, Actions (se habilitado).
- Confirmar entrega de e-mails (convite/redefinição/notificação).
- Validar auditoria de eventos de autenticação.

### 6) Go-live

- Migrar grupos por ondas (time a time).
- Monitorar CPU, RAM, disco e latência Git/HTTP.
- Ter runbook de incidentes: indisponibilidade SSO, expiração de certificado, rollback.

---

## Observações específicas do ambiente informado

- A VM está com `priority: Spot` e `evictionPolicy: Deallocate`: adequado para testes, não para produção crítica.
- A imagem reportada é `GitHub-Enterprise` versão `3.20.1`.
- Recomenda-se rotação de chaves administrativas e revisão de exposição de dados sensíveis.
