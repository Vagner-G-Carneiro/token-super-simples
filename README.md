# Autenticação por Token Simples em Java
### Proposta

---

## 1. Contexto

Em sistemas distribuídos, é comum usar **tokens** para identificar e autenticar usuários sem precisar consultar o banco de dados a cada requisição. Neste exemplo, usamos um formato minimalista e facil de indentificar quem é o usuário e o que ele pode fazer

---

## 2. Formato do Token

O token segue uma estrutura simples com **duas partes separadas por `_`**:

```
[ROLE]_[NOME_ÚNICO]
```

| Role | Prefixo | Exemplo |
|------|---------|---------|
| Usuário comum | `usr` | `usr_joaosilva` |
| Administrador | `adm` | `adm_mariaoliveira` |
| Inválido | qualquer outro | `mod_carlos` ❌ |

### Regras do nome de usuário:

- Apenas letras e números — **sem caracteres especiais, sem espaços, sem `_`**
- Mínimo de **5 caracteres**
- Máximo de **20 caracteres**

### Exemplos:

| Token | Resultado |
|-------|-----------|
| `usr_joaosilva` | ✅ Válido — usuário |
| `adm_mariaoliveira` | ✅ Válido — administrador |
| `usr_jo` | ❌ Nome muito curto (menos de 5 caracteres) |
| `adm_nomemuitolongoultrapassandolimite` | ❌ Nome muito longo (mais de 20 caracteres) |
| `usr_nome_com_underline` | ❌ Underscore no nome gera 3+ partes no split |
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

### Passo 2 — Dividir o token com `split("_")`

Como o nome **não possui `_`**, usamos o split sem limite de partes. Isso nos permite usar o número de partes como validação: se vier mais ou menos que 2, o formato está errado.

```java
String[] partes = token.split("_");
```

| Token | Resultado do split | Partes |
|---|---|---|
| `usr_joaosilva` | `["usr", "joaosilva"]` | 2 ✅ |
| `semunderline` | `["semunderline"]` | 1 ❌ |
| `usr_nome_invalido` | `["usr", "nome", "invalido"]` | 3 ❌ |

---

### Passo 3 — Exigir exatamente 2 partes

```java
if (partes.length != 2) {
    System.out.println("INVÁLIDO: formato incorreto (esperado ROLE_NOME)");
    return;
}
```

---

### Passo 4 — Extrair role e nome

```java
String role        = partes[0]; // "usr" ou "adm"
String nomeUsuario = partes[1]; // "joaosilva", "mariaoliveira", etc.
```

---

### Passo 5 — Validar o nome com regex e tamanho

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

### Passo 6 — Verificar a role

```java
if (role.equals("usr")) {
    System.out.println("Role: USUÁRIO | Nome: " + nomeUsuario);

} else if (role.equals("adm")) {
    System.out.println("Role: ADMINISTRADOR | Nome: " + nomeUsuario);

} else {
    System.out.println("INVÁLIDO: role desconhecida -> " + role);
}
```

> 💡 Use sempre `.equals()` para comparar Strings em Java, nunca `==`.

---

## 4. Código Completo

```java
public class TokenParser {

    private static final String ROLE_USER  = "usr";
    private static final String ROLE_ADMIN = "adm";

    public static void parseToken(String token) {
        // Passo 1: proteção contra entrada vazia
        if (token == null || token.isEmpty()) {
            System.out.println("  INVÁLIDO: token vazio");
            return;
        }

        // Passo 2: dividir pelo separador _
        String[] partes = token.split("_");

        // Passo 3: exigir exatamente 2 partes
        if (partes.length != 2) {
            System.out.println("  INVÁLIDO: formato incorreto (esperado ROLE_NOME)");
            return;
        }

        // Passo 4: extrair as partes
        String role        = partes[0];
        String nomeUsuario = partes[1];

        // Passo 5: validar nome — alfanumérico, entre 5 e 20 caracteres
        if (!nomeUsuario.matches("[a-zA-Z0-9]{5,20}")) {
            System.out.println("  INVÁLIDO: nome deve ter entre 5 e 20 caracteres alfanuméricos");
            return;
        }

        // Passo 6: verificar role
        if (role.equals(ROLE_USER)) {
            System.out.println("  Role: USUÁRIO (usr)");
            System.out.println("  Nome: " + nomeUsuario);

        } else if (role.equals(ROLE_ADMIN)) {
            System.out.println("  Role: ADMINISTRADOR (adm)");
            System.out.println("  Nome: " + nomeUsuario);

        } else {
            System.out.println("  INVÁLIDO: role desconhecida \"" + role + "\"");
        }
    }
}
```

---

## 5. Exemplos de Validação

**✅ Passa — usuário válido**
```
Token:  "usr_joaosilva"
Split:  ["usr", "joaosilva"]  →  2 partes ✅
Nome:   "joaosilva"  →  9 caracteres, apenas letras ✅
Role:   "usr" ✅

Resultado: Role: USUÁRIO | Nome: joaosilva
```

**✅ Passa — administrador válido**
```
Token:  "adm_mariaoliveira"
Split:  ["adm", "mariaoliveira"]  →  2 partes ✅
Nome:   "mariaoliveira"  →  13 caracteres, apenas letras ✅
Role:   "adm" ✅

Resultado: Role: ADMINISTRADOR | Nome: mariaoliveira
```

**❌ Não passa — nome muito curto**
```
Token:  "usr_jo"
Split:  ["usr", "jo"]  →  2 partes ✅
Nome:   "jo"  →  2 caracteres, abaixo do mínimo de 5 ❌

Resultado: INVÁLIDO: nome deve ter entre 5 e 20 caracteres alfanuméricos
```

---

## 6. Salvando no Banco de Dados

Após o parse, você terá `role` e `nomeUsuario` separados. Guarde-os em **colunas distintas** — nunca o token bruto inteiro.

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
    nome  VARCHAR(20)  NOT NULL,  -- máximo de 20 respeitando a regra do token
    role  VARCHAR(3)   NOT NULL   -- "usr" ou "adm"
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
  split("_")
      |
      v
  Tem 2 partes? ──── NÃO ──► INVÁLIDO  (sem _ ou _ no nome)
      |
      SIM
      v
  Nome é alfanumérico
  entre 5 e 20 chars? ──── NÃO ──► INVÁLIDO
      |
      SIM
      v
  role == "usr"? ──► USUÁRIO ✅
  role == "adm"? ──► ADMIN   ✅
  outro?         ──► INVÁLIDO ❌
```

---

*Documento produzido por solicitação do Professor na aula de Quarta-Feira 22/04 Sistemas Distribuídos.*
