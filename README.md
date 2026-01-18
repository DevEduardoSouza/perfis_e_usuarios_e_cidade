# Mapeamento de Perfis e Usuários - e-Cidade

## 📋 Visão Geral

O e-Cidade gerencia **usuários, permissões e departamentos** através de várias tabelas no schema `configuracoes`. Este documento mapeia a estrutura completa do sistema de autenticação e autorização.

---

## 🗄️ Tabelas Principais

### 1. `configuracoes.db_usuarios` - Usuários do Sistema

**Estrutura:**
| Campo | Tipo | Descrição |
|-------|------|-----------|
| `id_usuario` | INTEGER (PK) | ID único do usuário |
| `nome` | VARCHAR(60) | Nome completo |
| `login` | VARCHAR(20) UNIQUE | Login (obrigatório) |
| `senha` | VARCHAR(40) | Senha criptografada (obrigatório) |
| `usuarioativo` | INTEGER | Status: `1`=Ativo, `0`=Inativo |
| `email` | VARCHAR(200) | Email do usuário |
| `usuext` | INTEGER | Usuário externo (`0`=Não, `1`=Sim) |
| `administrador` | INTEGER | É admin? (`0`=Não, `1`=Sim) |
| `datatoken` | DATE | Data de criação do token |
| `remember_token` | VARCHAR(100) | Token "lembrar-me" |

**Estatísticas do Banco Zerado:**
- **Total de usuários:** 20
- **Usuários ativos:** 19
- **Administradores:** 3

**Campos Importantes:**
- `administrador = 1`: Usuário tem acesso total ao sistema
- `usuarioativo = 1`: Usuário pode fazer login
- `usuarioativo = 0`: Usuário bloqueado/inativo

---

### 2. `configuracoes.db_permissao` - Permissões de Menu

**Estrutura:**
| Campo | Tipo | Descrição |
|-------|------|-----------|
| `id_usuario` | INTEGER (FK) | ID do usuário |
| `id_item` | INTEGER | ID do item de menu |
| `permissaoativa` | CHAR(1) | `'t'`=Ativa, `'f'`=Inativa |
| `anousu` | INTEGER | Ano do usuário (exercício) |
| `id_instit` | INTEGER | ID da instituição |
| `id_modulo` | INTEGER | ID do módulo |

**Funcionamento:**
- Cada registro = uma permissão de um **item de menu** para um usuário
- Controla quais menus/opções o usuário pode acessar
- `permissaoativa = 't'` significa que o usuário tem acesso ao item

**Relacionamento:**
- `id_usuario` → `db_usuarios.id_usuario`
- `id_item` → `db_itensmenu.coditem` (item de menu)

---

### 3. `configuracoes.db_depusu` - Usuários x Departamentos

**Estrutura:** (campo `coddepto` e `id_usuario`)

**Funcionamento:**
- Vincula usuários a departamentos
- Um usuário pode estar em múltiplos departamentos
- Usado para controle de acesso por departamento

---

### 4. `configuracoes.db_menu` - Menu Principal

**Funcionamento:**
- Armazena a estrutura de menus do sistema
- Cada menu pode ter submenus (itens)

---

### 5. `configuracoes.db_itensmenu` - Itens de Menu

**Funcionamento:**
- Cada item de menu representa uma funcionalidade/tela
- Referenciado em `db_permissao.id_item`

---

### 6. `configuracoes.db_menuacesso` - Controle de Acesso ao Menu

**Funcionamento:**
- Define regras de acesso aos menus
- Pode restringir por perfil, departamento, etc.

---

## 🔐 Sistema de Permissões

### Tipos de Controle de Acesso

#### 1. **Administrador (`administrador = 1`)**
- Acesso total ao sistema
- Ignora permissões específicas de menu
- Pode gerenciar usuários e configurações

#### 2. **Permissões por Menu (`db_permissao`)**
- Cada usuário não-admin precisa ter permissões específicas
- Controla acesso a funcionalidades individuais
- Baseado em `id_item` (item de menu)

#### 3. **Permissões por Departamento (`db_depusu`)**
- Usuário só acessa dados/operações do seu departamento
- Múltiplos departamentos permitidos

#### 4. **Permissões por Instituição (`db_permissao.id_instit`)**
- Controle multi-institucional
- Usuário pode ter permissões em diferentes instituições

---

## 📊 Fluxo de Autenticação e Autorização

```
1. Usuário faz login (login.php → abrir.php)
   ↓
2. Sistema valida credenciais (db_usuarios)
   ↓
3. Verifica se usuário está ativo (usuarioativo = 1)
   ↓
4. Se administrador = 1:
   → Acesso total (ignora db_permissao)
   ↓
5. Se não-admin:
   → Consulta db_permissao WHERE id_usuario = X AND permissaoativa = 't'
   → Carrega apenas menus/itens permitidos
   ↓
6. Carrega departamentos (db_depusu)
   → Filtra dados por departamento
```

---

## 🔍 Consultas Úteis

### Listar todos os usuários ativos
```sql
SELECT id_usuario, login, nome, email, administrador 
FROM configuracoes.db_usuarios 
WHERE usuarioativo = 1
ORDER BY nome;
```

### Ver permissões de um usuário
```sql
SELECT 
    u.login,
    u.nome,
    i.descricao AS item_menu,
    p.permissaoativa,
    p.anousu
FROM configuracoes.db_permissao p
JOIN configuracoes.db_usuarios u ON p.id_usuario = u.id_usuario
JOIN configuracoes.db_itensmenu i ON p.id_item = i.coditem
WHERE u.login = 'dbseller'
AND p.permissaoativa = 't';
```

### Usuários e seus departamentos
```sql
SELECT 
    u.login,
    u.nome,
    d.descricao AS departamento
FROM configuracoes.db_depusu du
JOIN configuracoes.db_usuarios u ON du.id_usuario = u.id_usuario
JOIN configuracoes.db_depart d ON du.coddepto = d.coddepto
ORDER BY u.nome;
```

### Usuários administradores
```sql
SELECT id_usuario, login, nome, email 
FROM configuracoes.db_usuarios 
WHERE administrador = 1 
AND usuarioativo = 1;
```

---

## 📁 Arquivos Relacionados no Código

### Models (Laravel)
- `app/Models/User.php` - Model Laravel para db_usuarios
- `app/Models/DBUsuarios.php` - Model legacy

### Classes Legacy
- `classes/db_db_usuarios_classe.php` - Classe de manipulação de usuários
- `classes/db_db_permissao_classe.php` - Classe de permissões

### Modelos Legacy
- `model/configuracao/UsuarioSistema.model.php` - Lógica de usuário do sistema
- `model/configuracao/DBDepartamento.model.php` - Departamentos

### Bibliotecas
- `libs/db_conn.php` - Conexão com banco
- `libs/db_utils.php` - Utilitários (includes getDao)

---

## 🎯 Perfis Comuns no e-Cidade

### ⚠️ **IMPORTANTE: Não há perfis pré-definidos**

O e-Cidade **NÃO possui uma tabela de perfis** com roles reutilizáveis (como "Administrador", "Fiscal", "Tesoureiro"). O sistema funciona com **permissões individuais** por usuário.

### Tipos de Usuários (baseado em `administrador`)

#### 1. **Administrador do Sistema**
- `administrador = 1`
- Acesso total (ignora `db_permissao`)
- Pode gerenciar usuários e configurações
- **Usuário padrão:** `dbseller` (senha: `dbseller`)

#### 2. **Usuário Comum (Não-Admin)**
- `administrador = 0` ou NULL
- Permissões via `db_permissao` (item por item de menu)
- Acesso limitado apenas aos menus/opções permitidos
- Sem permissão = sem acesso à funcionalidade

### Tipos de Acesso (baseado em características)

#### 3. **Usuário Externo**
- `usuext = 1`
- Acesso via DBPortal (portal externo)
- Geralmente mais restritivo
- Pode ser para fornecedores, contribuintes, etc.

#### 4. **Usuário por Departamento**
- Vinculado via `db_depusu`
- Acesso apenas aos dados do seu(s) departamento(s)
- Comum em prefeituras com múltiplas secretarias
- Múltiplos departamentos permitidos por usuário

### 📋 **Usuários Padrão Criados no Banco Zerado**

O banco inicial já cria usuários com **nomes sugerindo funções**, mas são apenas **nomes de usuários**, não perfis reutilizáveis:

**Administradores:**
- `dbseller` - PREFEITURA DBSELLER (admin padrão)

**Usuários Comuns (sem permissões pré-configuradas):**
- `compras` - Compras
- `licitação` - Licitação
- `almoxarifado` - Almoxarifado
- `patrimonio` - Patrimônio
- `tesouraria` - Tesouraria
- `empenho` - Empenho
- `orçamento` - Orçamento
- `contratos` - Contratos
- `obras` - Obras
- `frotas` - Frotas
- `fornecedor` - Fornecedor
- `contribuinte` - Contribuinte
- `funcionario` - Funcionário
- `imobiliaria` - Imobiliária
- `escritorio` - Escritório

**⚠️ IMPORTANTE:** Esses usuários padrão **não vêm com permissões pré-configuradas**. As permissões devem ser atribuídas manualmente via interface ou SQL após a instalação.

### 🔧 **Como Criar "Perfis" Customizados**

Como não há perfis pré-definidos, cada município cria seus próprios "perfis" de forma dinâmica:

1. **Criar usuário** em `db_usuarios`
2. **Atribuir permissões** em `db_permissao` (copiando de outro usuário similar, se necessário)
3. **Vincular departamentos** em `db_depusu` (opcional)

**Exemplo:** Se quiser criar um "perfil Fiscal":
- Criar usuário `fiscal_joao`
- Copiar permissões de `db_permissao` de outro fiscal (ou definir manualmente)
- Vincular ao departamento "Fiscal" via `db_depusu`

---

## ⚙️ Configurações Importantes

### Variáveis de Sessão (após login)
```php
$_SESSION['DB_login']         // Login do usuário
$_SESSION['DB_id_usuario']    // ID do usuário
$_SESSION['DB_administrador'] // É admin? (1 ou 0)
$_SESSION['DB_coddepto']      // Departamento padrão
```

### Validação de Acesso (código)
```php
// Verifica se é admin
if ($_SESSION['DB_administrador'] == 1) {
    // Acesso liberado
}

// Verifica permissão específica
$temPermissao = db_verifica_permissao($id_usuario, $id_item);
```

---

## 🔧 Manutenção

### Criar novo usuário
1. Inserir em `db_usuarios`
2. Atribuir permissões em `db_permissao`
3. Vincular a departamentos em `db_depusu` (opcional)

### Bloquear usuário
```sql
UPDATE configuracoes.db_usuarios 
SET usuarioativo = 0 
WHERE id_usuario = X;
```

### Remover todas as permissões
```sql
DELETE FROM configuracoes.db_permissao 
WHERE id_usuario = X;
```

---




**Última atualização:** 2026-01-18  
**Versão do e-Cidade:** Base zerada (dump inicial)
