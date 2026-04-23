# Autenticação por Token Simples em Java
### Mini Aula — Sistemas Distribuídos

---

## 1. Contexto

Em sistemas distribuídos, é comum usar **tokens** para identificar e autenticar usuários sem precisar consultar o banco de dados a cada requisição. Neste exemplo, usamos um formato minimalista e didático — sem hash, sem criptografia — para focar na **lógica de parse e validação**.

> ⚠️ **Importante:** Em produção, tokens devem conter hash (JWT, por exemplo). Este exemplo é exclusivamente didático.

---

## 2. Formato do Token

O sistema possui dois tipos de token com formatos distintos:

### Administrador

O administrador é **único** no sistema. Seu token é fixo, sem separador, sem nome:

```
adm
```

### Usuário comum

Usuários seguem o padrão `ROLE_NOME`:

```
usr_[NOME_ÚNICO]
```

### Regras do nome de usuário:

- Apenas letras e números — **sem caracteres especiais, sem espaços, sem `_`**
- Mínimo de **5 caracteres**
- Máximo de **20 caracteres**

### Exemplos:

| Token | Resultado |
|-------|-----------|
| `adm` | ✅ Administrador |
| `usr_joaosilva` | ✅ Usuário comum |
| `adm_qualquercoisa` | ❌ Admin não segue o padrão `ROLE_NOME` |
| `usr_jo` | ❌ Nome muito curto (menos de 5 caracteres) |
| `usr_nomemuitolongoultrapassando` | ❌ Nome muito longo (mais de 20 caracteres) |
| `mod_carlos` | ❌ Role desconhecida |

---

## 3. A Lógica de Parse — Passo a Passo

### Passo 1 — Verificar se o token não está vazio

```java
if (token == null || token.isEmpty()) {
    System.out.println("INVÁLIDO: token vazio");
    return;
}
```

---

### Passo 2 — Verificar se é o token de administrador

O token `adm` é checado **antes do split**, pois ele não segue o padrão `ROLE_NOME` — ele é o token inteiro.

```java
if (token.equals("adm")) {
    System.out.println("Role: ADMINISTRADOR");
    return;
}
```

> 💡 O `return` garante que o código abaixo não execute. Se chegou até o próximo passo, só pode ser usuário comum ou inválido.

---

### Passo 3 — Dividir o token com `split("_")`

Como o nome não possui `_`, o resultado deve ter **exatamente 2 partes**. Qualquer outro número indica formato errado.

```java
String[] partes = token.split("_");
```

| Token | Resultado do split | Partes |
|---|---|---|
| `usr_joaosilva` | `["usr", "joaosilva"]` | 2 ✅ |
| `semunderline` | `["semunderline"]` | 1 ❌ |
| `usr_nome_invalido` | `["usr", "nome", "invalido"]` | 3 ❌ |

---

### Passo 4 — Exigir exatamente 2 partes

```java
if (partes.length != 2) {
    System.out.println("INVÁLIDO: formato incorreto (esperado usr_NOME)");
    return;
}
```

---

### Passo 5 — Extrair role e nome

```java
String role        = partes[0]; // "usr"
String nomeUsuario = partes[1]; // "joaosilva"
```

---

### Passo 6 — Validar o nome com regex

A regex `[a-zA-Z0-9]{5,20}` valida tudo de uma vez:
- `[a-zA-Z0-9]` — apenas letras e números
- `{5,20}` — entre 5 e 20 caracteres

```java
if (!nomeUsuario.matches("[a-zA-Z0-9]{5,20}")) {
    System.out.println("INVÁLIDO: nome deve ter entre 5 e 20 caracteres alfanuméricos");
    return;
}
```

---

### Passo 7 — Verificar a role

```java
if (role.equals("usr")) {
    System.out.println("Role: USUÁRIO | Nome: " + nomeUsuario);
} else {
    System.out.println("INVÁLIDO: role desconhecida -> " + role);
}
```

> 💡 Use sempre `.equals()` para comparar Strings em Java, nunca `==`.

---

## 4. Código Completo

```java
public class TokenParser {

    private static final String TOKEN_ADMIN = "adm";
    private static final String ROLE_USER   = "usr";

    public static void parseToken(String token) {
        // Passo 1: proteção contra entrada vazia
        if (token == null || token.isEmpty()) {
            System.out.println("  INVÁLIDO: token vazio");
            return;
        }

        // Passo 2: admin é um token fixo — verificar antes do split
        if (token.equals(TOKEN_ADMIN)) {
            System.out.println("  Role: ADMINISTRADOR");
            return;
        }

        // Passo 3: dividir pelo separador _
        String[] partes = token.split("_");

        // Passo 4: exigir exatamente 2 partes
        if (partes.length != 2) {
            System.out.println("  INVÁLIDO: formato incorreto (esperado usr_NOME)");
            return;
        }

        // Passo 5: extrair as partes
        String role        = partes[0];
        String nomeUsuario = partes[1];

        // Passo 6: validar nome — alfanumérico, entre 5 e 20 caracteres
        if (!nomeUsuario.matches("[a-zA-Z0-9]{5,20}")) {
            System.out.println("  INVÁLIDO: nome deve ter entre 5 e 20 caracteres alfanuméricos");
            return;
        }

        // Passo 7: verificar role
        if (role.equals(ROLE_USER)) {
            System.out.println("  Role: USUÁRIO (usr)");
            System.out.println("  Nome: " + nomeUsuario);
        } else {
            System.out.println("  INVÁLIDO: role desconhecida \"" + role + "\"");
        }
    }
}
```

---

## 5. Exemplos de Validação

**✅ Passa — administrador**
```
Token:  "adm"
Equals: "adm" == "adm" ✅

Resultado: Role: ADMINISTRADOR
```

**✅ Passa — usuário válido**
```
Token:  "usr_joaosilva"
Split:  ["usr", "joaosilva"]  →  2 partes ✅
Nome:   "joaosilva"  →  9 caracteres, apenas letras ✅
Role:   "usr" ✅

Resultado: Role: USUÁRIO | Nome: joaosilva
```

**❌ Não passa — role desconhecida**
```
Token:  "mod_carlos"
Split:  ["mod", "carlos"]  →  2 partes ✅
Nome:   "carlos"  →  6 caracteres, apenas letras ✅
Role:   "mod" ❌

Resultado: INVÁLIDO: role desconhecida "mod"
```

---

## 6. Salvando no Banco de Dados

Após o parse, guarde `role` e `nomeUsuario` em **colunas distintas**. Para o administrador, o nome pode ser salvo como um valor fixo ou deixado nulo, dependendo da modelagem do projeto.

```java
public void salvarUsuario(String role, String nomeUsuario) {
    // Com JDBC:
    String sql = "INSERT INTO usuarios (nome, role) VALUES (?, ?)";
    // preparedStatement.setString(1, nomeUsuario);
    // preparedStatement.setString(2, role);

    // Com JPA/Hibernate:
    // Usuario u = new Usuario(nomeUsuario, role);
    // entityManager.persist(u);
}
```

### Estrutura sugerida da tabela:

```sql
CREATE TABLE usuarios (
    id    INT PRIMARY KEY AUTO_INCREMENT,
    nome  VARCHAR(20),          -- nulo para o administrador
    role  VARCHAR(3)  NOT NULL  -- "usr" ou "adm"
);
```

---

## 7. Resumo do Fluxo

```
Token recebido
      |
      v
  Está vazio? ──── SIM ──► INVÁLIDO
      |
      NÃO
      v
  token == "adm"? ──── SIM ──► ADMINISTRADOR ✅
      |
      NÃO
      v
  split("_")
      |
      v
  Tem 2 partes? ──── NÃO ──► INVÁLIDO
      |
      SIM
      v
  Nome é alfanumérico
  entre 5 e 20 chars? ──── NÃO ──► INVÁLIDO
      |
      SIM
      v
  role == "usr"? ──► USUÁRIO ✅
  outro?         ──► INVÁLIDO ❌
```

---

*Mini aula produzida para a disciplina de Sistemas Distribuídos.*