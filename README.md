# Documentação: Sistema de Autenticação de Dois Fatores em Go

## 1. Introdução

Este documento descreve um sistema de autenticação de dois fatores (2FA) implementado em Go, utilizando o framework Gin para a API web e o banco de dados MySQL para o armazenamento de usuários. O sistema gera um código de verificação aleatório e o envia por e-mail após a autenticação bem-sucedida com senha.  O usuário então precisa inserir o código recebido para concluir o processo de login.

## 2. Dependências

O projeto utiliza as seguintes bibliotecas Go:

* **`gin-gonic/gin`**: Framework web para construção da API RESTful.
* **`go-sql-driver/mysql`**: Driver MySQL para interação com o banco de dados.
* **`golang.org/x/crypto/bcrypt`**: Biblioteca para hashing de senhas usando o algoritmo bcrypt.
* **`gopkg.in/gomail.v2`**: Biblioteca para envio de e-mails.
* **`bufio`**: Biblioteca para leitura de entrada do usuário.
* **`crypto/rand`**: Biblioteca para geração de números aleatórios.
* **`database/sql`**: Biblioteca para interação com bancos de dados SQL.
* **`encoding/base64`**: Biblioteca para codificação base64.
* **`fmt`**: Biblioteca para formatação de strings e entrada/saída.
* **`log`**: Biblioteca para registro de erros e mensagens.
* **`os`**: Biblioteca para interação com o sistema operacional.
* **`strings`**: Biblioteca para manipulação de strings.

Para instalar as dependências, execute o comando `go get` para cada pacote:

```bash
go get github.com/gin-gonic/gin
go get github.com/go-sql-driver/mysql
go get golang.org/x/crypto/bcrypt
go get gopkg.in/gomail.v2
```

## 3. Arquitetura

O sistema possui os seguintes componentes principais:

* **API RESTful (Gin):**  Fornece endpoints para criação de usuários (`/create_user`) e login (`/login`).
* **Banco de Dados MySQL:** Armazena informações de usuários, incluindo nome, email e senha criptografada.
* **Módulo de envio de e-mail (Gomail):** Envia o código de verificação para o email do usuário.
* **Módulo de Geração de Código:** Gera um código aleatório para a verificação de dois fatores.
* **Módulo de Verificação de Código:** Verifica se o código inserido pelo usuário é o mesmo que o código enviado por email.

A interação entre os componentes ocorre da seguinte maneira:

1. O usuário realiza uma requisição POST para `/create_user` ou `/login`.
2. A API valida a requisição e interage com o banco de dados.
3. No caso de login, após a autenticação com a senha, um código de verificação é gerado e enviado via email.
4. O usuário insere o código recebido.
5. O sistema verifica se o código é válido.


## 4. Análise do Código

### 4.1 Variáveis Globais

* `code_email string`: Armazena o código de verificação gerado.
* `db *sql.DB`: Ponteiro para a conexão com o banco de dados MySQL.

### 4.2 Funções

* **`Check_code()`:**
    * **Propósito:** Verifica se o código inserido pelo usuário é igual ao código armazenado em `code_email`.
    * **Entrada:** Nenhuma (lê do stdin).
    * **Saída:** Nenhuma (imprime mensagem na console).
    * **Lógica:** Lê o código do usuário, remove espaços em branco e compara com `code_email`.

* **`Send_emailCode(email string)`:**
    * **Propósito:** Envia um email com o código de verificação para o endereço especificado.
    * **Entrada:** `email string` - endereço de email do usuário.
    * **Saída:** Nenhuma.
    * **Lógica:** Cria uma mensagem de email com o código `code_email`, configura as credenciais do SMTP do Gmail e envia o email.  Utiliza `gomail`.  Trata erros com `panic`.

* **`Get_hash(email string, password string) string`:**
    * **Propósito:** Recupera a senha hash do banco de dados para um determinado email.
    * **Entrada:** `email string`, `password string`.
    * **Saída:** `string` - senha hash.
    * **Lógica:** Consulta o banco de dados para obter o ID do usuário pelo email, em seguida recupera a senha hash pelo ID. Trata erros com `panic`.  **Nota:** Essa função retorna a senha hash diretamente, o que é uma vulnerabilidade de segurança.

* **`Create_user(name string, email string, password string) string`:**
    * **Propósito:** Cria um novo usuário no banco de dados.
    * **Entrada:** `name string`, `email string`, `password string`.
    * **Saída:** `string` - "True" se a criação for bem sucedida.
    * **Lógica:** Cria a tabela `users` se ela não existir e insere um novo usuário. Trata erros com `panic`.

* **`Connect_db()`:**
    * **Propósito:** Estabelece a conexão com o banco de dados MySQL.
    * **Entrada:** Nenhuma.
    * **Saída:** Nenhuma.
    * **Lógica:** Abre uma conexão com o banco de dados usando as credenciais especificadas na string de conexão. Trata erros com `panic`.

* **`GenerateRandomCode(length int) string`:**
    * **Propósito:** Gera um código aleatório de um determinado comprimento.
    * **Entrada:** `length int` - comprimento do código.
    * **Saída:** `string` - código aleatório.
    * **Lógica:** Gera bytes aleatórios e os codifica em base64.  Trata erros silenciosamente (o que não é uma boa prática).

* **`main()`:**
    * **Propósito:** Função principal do programa. Inicializa o servidor web e define os endpoints.
    * **Entrada:** Nenhuma.
    * **Saída:** Nenhuma.
    * **Lógica:** Define dois endpoints: `/create_user` e `/login`.  `/create_user` cria um novo usuário com a senha hash usando bcrypt. `/login` realiza a autenticação com senha e, se bem sucedida, envia um código de verificação por email.


## 5. Fluxo de Execução Principal

1. O programa inicia o servidor web Gin.
2.  Uma requisição POST para `/create_user` é recebida, criando um novo usuário no banco de dados com a senha criptografada usando bcrypt.
3. Uma requisição POST para `/login` é recebida.
4. A senha fornecida é comparada com a senha hash armazenada no banco de dados usando `bcrypt.CompareHashAndPassword`.
5. Se a autenticação for bem-sucedida, um código aleatório é gerado e enviado por email usando `Send_emailCode`.
6. `Check_code()` é chamado para verificar o código inserido pelo usuário.

## 6. Tratamento de Erros e Exceções

O código utiliza `panic` para lidar com erros em diversas funções.  Isso não é uma boa prática para produção, pois interrompe a execução do programa.  Um tratamento de erro mais robusto usando `recover` ou retornos explícitos de erros seria necessário.

## 7. Configurações

As credenciais do banco de dados e do SMTP estão codificadas diretamente no código (`connect` string e credenciais do Gmail em `Send_emailCode`).  Isso é uma séria vulnerabilidade de segurança.  Deve-se utilizar variáveis de ambiente ou um arquivo de configuração separado para armazenar essas informações sensíveis.

## 8. Exemplos de Uso

**Criando um usuário:**

```bash
curl -X POST -H "Content-Type: application/json" -d '{"name": "John Doe", "email": "john.doe@example.com", "password": "mypassword"}' http://localhost:8080/create_user
```

**Realizando login:**

```bash
curl -X POST -H "Content-Type: application/json" -d '{"email": "john.doe@example.com", "password": "mypassword"}' http://localhost:8080/login
```

## 9. Boas Práticas de Programação

* Uso do framework Gin para a construção da API.
* Uso do bcrypt para criptografar senhas.
* Utilização de `sql.DB` para a conexão com o banco de dados.

## 10. Áreas de Melhoria

* **Tratamento de erros:** Substituir `panic` por um tratamento de erros mais robusto.
* **Segurança:** Armazenar credenciais sensíveis em variáveis de ambiente ou arquivo de configuração.  Não retornar a senha hash diretamente em `Get_hash`.  Implementar validação de entrada completa.  Proteger contra ataques de injeção de SQL.
* **Escalabilidade:** Considerar o uso de uma solução de envio de email mais robusta e escalável para um ambiente de produção.
* **Testes:** Implementar testes unitários e de integração.


## 11. Testes

Não há testes incluídos no código fornecido.  Testes unitários devem ser implementados para cada função, focando em cenários de sucesso e falha.  Testes de integração devem verificar a comunicação entre os componentes (API, banco de dados, envio de email).


## 12. Considerações de Segurança

* **Criptografia de Senha:** O uso do bcrypt é uma boa prática, mas a força da senha deve ser verificada e reforçada.
* **Armazenamento de Credenciais:** As credenciais do banco de dados e do SMTP **NÃO** devem ser armazenadas diretamente no código.
* **Validação de Entrada:**  Implementar validação completa de entrada para prevenir ataques de injeção de SQL e outros ataques.
* **Proteção contra ataques CSRF e XSS:**  Implementar medidas para proteger contra estes tipos de ataques comuns em aplicações web.

## 13. Desempenho

O desempenho do sistema depende principalmente do desempenho do banco de dados e do serviço de email.  O uso de pool de conexões para o banco de dados pode melhorar o desempenho.

## 14. Conclusão

Este sistema de autenticação de dois fatores fornece uma base funcional para a autenticação segura de usuários, mas requer melhorias significativas no tratamento de erros, segurança e testes para ser considerado adequado para um ambiente de produção. As sugestões de melhoria apresentadas neste documento devem ser implementadas para garantir a segurança, robustez e escalabilidade do sistema.


---


# 🛡️ GoAuth: Seu Sistema de Autenticação de Dois Fatores em Go!

Este projeto apresenta um sistema de autenticação de dois fatores (2FA) robusto e eficiente construído com Go. Diga adeus a senhas frágeis e olá à segurança reforçada para suas aplicações!  GoAuth protege seus usuários com uma camada extra de segurança, tornando suas contas muito mais resistentes a acessos não autorizados.  Prepare-se para uma experiência de login segura e tranquila!


## Principais Vantagens 🚀

* **Segurança Aprimorada:** 🔒  Adicione uma camada extra de segurança ao seu sistema com autenticação de dois fatores.
* **Fácil Integração:** ⚙️  Integre facilmente ao seu projeto Go existente.
* **Código Limpo e Bem Documentado:** 📚  Facilita a compreensão e manutenção do código.
* **Escalável e Robusto:** 💪  Construído para lidar com um grande volume de usuários e solicitações.
* **Código Aberto:** 🤝  Contribua para a comunidade e ajude a melhorar o projeto.


## Instalação ⬇️

1. **Clone o repositório:**

```bash
git clone <URL_DO_REPOSITORIO>
cd GoAuth
```

2. **Instale as dependências:**

```bash
go mod tidy
```

3. **Configure o banco de dados:**  Crie um banco de dados MySQL chamado `goAuth` e ajuste as credenciais no arquivo `main.go` (variável `connect`).


## Como Usar ⚙️

Para executar o servidor:

```bash
go run main.go
```

O sistema expõe duas rotas:

* **/create_user**:  Cria um novo usuário. Envie uma requisição POST com JSON no corpo:

```json
{
  "name": "Seu Nome",
  "email": "seu_email@example.com",
  "password": "SuaSenhaForte"
}
```

* **/login**:  Realiza o login do usuário. Envie uma requisição POST com JSON no corpo:

```json
{
  "email": "seu_email@example.com",
  "password": "SuaSenhaForte"
}
```

Após a autenticação bem-sucedida, um código de verificação será enviado para o email fornecido.  O usuário precisará inserir este código para concluir o login.


## Docker 🐳

1. **Faça o pull da imagem (substitua `britoarthur855482/creat_readmi_documentation:001` pelo nome da sua imagem):**

```bash
docker pull <your_docker_image>
```

2. **Execute a imagem (substitua `<your_docker_image>` pelo nome da sua imagem e ajuste a porta se necessário):**

```bash
docker run -p 8501:8501 <your_docker_image>
```


## Pontos Fortes ✨

* **Segurança:**  Utiliza o bcrypt para o hash de senhas, garantindo a segurança das informações dos usuários.
* **Eficiência:**  Código otimizado para desempenho.
* **Flexibilidade:**  Fácil de adaptar para diferentes cenários de autenticação.
* **Escalabilidade:**  Projetado para suportar um grande número de usuários.
