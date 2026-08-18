# Painel de Projetos — Via3 Tecnologia

Plataforma web para acompanhar projetos com **estimativa de horas por tarefa**, **apontamento de horas com data e autor** e **totais por etapa e por projeto** — o que falta no Planner.

- Acesso por **código de 6 dígitos enviado para o e-mail corporativo** (sem senha para gerenciar).
- Multiprojeto, multiusuário, dados compartilhados em tempo real entre a equipe.
- Papéis: **administrador** (cria/exclui projetos, gerencia a equipe) e **técnico** (edita tarefas e lança horas).
- Banco SQLite em arquivo, dentro de um volume — backup é copiar uma pasta.

---

## 1. Subir em 5 minutos

Pré-requisitos: um servidor Linux com **Docker** e **Docker Compose**.

```bash
# 1. copie a pasta do projeto para o servidor, ex.: /opt/via3-painel
cd /opt/via3-painel

# 2. crie o arquivo de configuração
cp .env.example .env
nano .env          # ajuste domínio, e-mails autorizados e SMTP

# 3. suba
docker compose up -d --build

# 4. acompanhe
docker compose logs -f painel
```

Acesse `https://SEU-DOMINIO` (ou `http://IP:3000` se optar pelo modo interno — veja abaixo).

**O primeiro e-mail que entrar vira administrador.** Os endereços listados em `ADMIN_EMAILS` também são sempre administradores.

### Sem domínio público (uso só na rede interna ou via VPN)

No `docker-compose.yml`: remova o serviço `proxy`, descomente o bloco `ports` do serviço `painel` e, no `.env`, use `COOKIE_SECURE=false`. O painel responde em `http://IP-DO-SERVIDOR:3000`.

> Publicar direto na internet sem HTTPS não é recomendado — o cookie de sessão trafegaria em texto claro. Se precisar de acesso externo sem abrir portas, um Cloudflare Tunnel resolve bem.

---

## 2. Envio do código de acesso (SMTP)

O padrão do `.env.example` usa o Microsoft 365 (`smtp.office365.com:587`, STARTTLS).

Para funcionar, a caixa usada precisa ter o **SMTP autenticado habilitado**:

1. No admin do Microsoft 365, crie uma caixa dedicada (ex.: `painel@via3tecnologia.com.br`).
2. Habilite *Authenticated SMTP* nessa caixa (Usuários → Usuários ativos → e-mail → Gerenciar aplicativos de email).
3. Se o tenant exigir MFA em todos os usuários, crie uma **exclusão de Acesso Condicional** para essa conta de serviço restrita ao IP do servidor — a Microsoft bloqueia autenticação básica por padrão.

Alternativas se a política interna não permitir SMTP básico: usar um relay SMTP interno, Amazon SES, Resend, Brevo ou similar — basta trocar `SMTP_HOST/PORT/USER/PASS`.

**Para testar antes de configurar o SMTP:** use `MAIL_TRANSPORT=log`. O código aparece em `docker compose logs -f painel`.

---

## 3. Segurança implementada

| Item | Como está |
|---|---|
| Quem entra | Só e-mails dos domínios em `DOMINIOS_PERMITIDOS` |
| Código | 6 dígitos aleatórios, guardado como hash SHA-256, validade de 10 min, uso único |
| Força bruta | Máx. 5 tentativas por código e 5 códigos por e-mail/hora |
| Sessão | Token aleatório de 256 bits, hash no banco, cookie `HttpOnly` + `SameSite=Lax` + `Secure` |
| CSRF | Toda escrita exige o cabeçalho `x-via3` (não enviável por formulário externo) |
| Bloqueio | Admin pode desativar um usuário na aba **Equipe** — a sessão dele cai no próximo clique |

Recomendação: revise a aba **Equipe** de tempos em tempos e bloqueie quem saiu da empresa. Se a conta é desligada no Entra, ela deixa de receber o código — mas o bloqueio explícito encerra a sessão ativa na hora.

---

## 4. Backup e restauração

Todo o banco fica em `./dados` (montado em `/app/data`).

```bash
# backup diário (coloque no cron)
tar czf /backup/painel-$(date +%F).tar.gz -C /opt/via3-painel dados

# restaurar
docker compose down
tar xzf /backup/painel-2026-08-20.tar.gz -C /opt/via3-painel
docker compose up -d
```

---

## 5. Atualizar

```bash
cd /opt/via3-painel
docker compose down
# substitua os arquivos da aplicação
docker compose up -d --build
```

O esquema do banco é criado/atualizado sozinho na subida (`CREATE TABLE IF NOT EXISTS`).

---

## 6. Como a equipe usa

1. Entra com o e-mail da empresa e o código recebido.
2. Abre o projeto e preenche a **Estimativa** de cada tarefa (aceita `8`, `1,5`, `30m`, `1h30`).
3. Clica no valor da coluna **Realizado** para lançar horas: data, quantidade e uma observação.
4. Os subtotais por etapa, o total do projeto e o gráfico se atualizam para todo mundo.
5. **Relatório de horas** mostra o total por técnico e os últimos lançamentos.
6. **Exportar CSV** gera a planilha para anexar no status report.

---

## 7. Estrutura

```
src/index.js   API REST (Express) e entrega do front
src/auth.js    código por e-mail, sessões e permissões
src/db.js      esquema SQLite
src/mail.js    envio do código (SMTP ou log)
src/seed.js    modelo do projeto CPA
public/        interface (HTML/CSS/JS, sem build)
```

Sem framework de front e sem etapa de build: o que está em `public/` é o que roda.
