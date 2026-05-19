# Contexto da Codebase — Fluxo de Login MFA
## Projetos: `agenciavirtual` + `fe-canaisdigitais`
**Data de geração:** 2026-05-14  
**Finalidade:** Referência para atividades de IA First — entendimento do fluxo de autenticação MFA implementado nas duas codebases.

---

## 1. Visão Geral da Arquitetura

```
┌────────────────────────────────────────────────────────────────────┐
│                      Cliente (navegador)                           │
│              FE-CanaisDigitais — Angular SPA                       │
└──────────────────────────────┬─────────────────────────────────────┘
                               │ /api/**
                               ▼
                   ┌─────────────────────┐
                   │       ms-bff        │  (Node.js BFF / proxy)
                   │  /api → AV e outros │
                   └──────────┬──────────┘
                              │ /tsOneServices/rest/**
                              ▼
              ┌─────────────────────────────────┐
              │         agenciavirtual          │
              │   Java EE · JAX-RS · WildFly    │
              │  JWT · JPA · Flyway · Mockito   │
              └────────────────┬────────────────┘
                               │
                    ┌──────────┴──────────┐
                    │  Banco de Dados     │  (Oracle / H2 testes)
                    │  AV_CODIGO_         │
                    │  VERIFICACAO        │
                    │  AV_CLIENTE         │
                    └─────────────────────┘
```

---

## 2. Projeto: `agenciavirtual` (Backend Java EE)

### 2.1 Tecnologias e Padrões

| Item | Detalhe |
|---|---|
| Linguagem | Java 8 |
| Runtime | WildFly (JAX-RS / CDI / EJB) |
| Auth | JWT próprio (claims customizadas) |
| Testes | JUnit 4 + Mockito |
| Migrations | Flyway |
| Build | Maven (mvnw) |
| Contexto de auth | Anotações próprias (@Authenticated, @TokenAuthenticated, @PasswordExpiredAuthenticated) |

### 2.2 Estrutura de Pacotes — Autenticação MFA

```
agenciavirtual/src/corp/ambiental/agencia/
├── auth/
│   ├── resource/
│   │   └── v4/
│   │       ├── LoginResourceV4.java          ← POST /v4/auth/login
│   │       │                                    POST /v4/auth/redefinir-senha-expirada
│   │       └── LoginMfaResourceV4.java       ← POST /v4/auth/mfa/cliente/{id}/send/{tipo}
│   │                                            POST /v4/auth/mfa/cliente/{id}/validate/{tipo}
│   ├── service/
│   │   ├── AbstractLoginService.java         ← base com helpers de login
│   │   ├── v4/
│   │   │   ├── LoginServiceV4.java           ← lógica de login inicial + reset senha expirada
│   │   │   ├── LoginMfaServiceV4.java        ← lógica pós-validação OTP: gera token final
│   │   │   ├── TokenServiceV4.java           ← envio e confirmação de OTP (SMS/Email)
│   │   │   ├── MfaDeliveryType.java          ← enum: SMS | EMAIL (com validação de canal)
│   │   │   └── dto/
│   │   │       └── ResetPasswordExpiredRequest.java
│   │   └── dto/
│   │       ├── UserLoginDto.java             ← builder com claims: clienteId, mfaTokenIsValid, etc.
│   │       ├── request/LoginRequest.java
│   │       └── response/UserLoggedResponse.java
│   ├── adapter/
│   │   ├── LoginAdapter.java
│   │   └── mapper/UserLoginDtoMapper.java    ← monta UserLoginDto com mfaTokenIsValid
│   ├── exceptions/
│   │   ├── CelularNaoEncontradoException.java
│   │   └── EmailNaoEncontradoException.java
│   └── validation/
│       ├── CodigoVerificacaoValidator.java
│       ├── PhoneNumberValidator.java
│       └── EmailValidator.java
├── core/
│   ├── service/
│   │   └── JWTService.java                  ← gera/valida tokens; enum AuthFlow (MFA | LEGACY)
│   ├── context/
│   │   ├── TokenAuthenticated.java           ← permite token pré-MFA; rejeita refresh tokens
│   │   ├── Authenticated.java                ← exige mfaTokenIsValid=true e passwordExpired=false
│   │   └── PasswordExpiredAuthenticated.java ← exige mfaTokenIsValid=true e passwordExpired=true
│   └── filters/
│       └── AuthenticationContextFilter.java  ← filtro JAX-RS que valida claims por contexto
├── parametrosistema/
│   └── service/
│       └── ParametroSistemaService.java      ← isMfaExpirado(), isSenhaExpirada(), parâmetros dinâmicos
├── cliente/
│   ├── service/
│   │   └── ClienteService.java              ← findClienteDados(), registerMfaValidado()
│   └── adapter/
│       └── ClienteAdapter.java              ← updateClientePassword(), registerMfaValidado()
└── auth/
    └── model/
        └── CodigoVerificacao.java            ← entidade JPA da tabela AV_CODIGO_VERIFICACAO
```

### 2.3 Claims do JWT

| Claim | Tipo | Descrição |
|---|---|---|
| `clienteId` | Long | ID único do cliente |
| `cpfCnpj` | String | CPF ou CNPJ |
| `email` | String | E-mail do cliente |
| `mfaTokenIsValid` | Boolean | `false` = MFA pendente; `true` = MFA concluído |
| `passwordExpired` | Boolean | `true` = troca de senha obrigatória |
| `authFlow` | String | `"mfa"` ou `"legacy"` — discrimina o fluxo |

> **Regra:** tokens com `authFlow=legacy` nunca carregam as claims de MFA; tokens com `authFlow=mfa` sempre as carregam.

### 2.4 Contextos de Autorização (anotações JAX-RS)

| Anotação | Requisito do Token |
|---|---|
| `@TokenAuthenticated` | Qualquer token válido do fluxo de login (inclusive pré-MFA); rejeita refresh tokens |
| `@PasswordExpiredAuthenticated` | `mfaTokenIsValid=true` **E** `passwordExpired=true` |
| `@Authenticated` | `mfaTokenIsValid=true` **E** `passwordExpired=false` (área autenticada final) |

### 2.5 Endpoints REST (v4)

#### POST `/v4/auth/login`
- **Recurso:** `LoginResourceV4`
- **Serviço:** `LoginServiceV4.requestLogin()`
- **Entrada:** `{ email?, cpfCnpj?, senha }`
- **Saída:** `{ accessToken, refreshToken, tokenType, expiresIn, refresh_token_expires_in }`
- **Lógica:** autentica, checa MFA expirado via `ParametroSistemaService`, emite JWT com `authFlow=mfa`

#### POST `/v4/auth/mfa/cliente/{idCliente}/send/{tipoEnvio}`
- **Recurso:** `LoginMfaResourceV4` (`@TokenAuthenticated`)
- **Serviço:** `TokenServiceV4.enviarCodigoVerificacaoSms/Email()`
- **Valida:** `clienteId` do path bate com claim do JWT
- **Destino:** busca telefone/email no cadastro do cliente
- **Armazena:** OTP em `AV_CODIGO_VERIFICACAO` (TTL 5 min, reenvio bloqueado por 150s)
- **tipoEnvio aceito:** `sms` | `email` (validado via `MfaDeliveryType.from()`)

#### POST `/v4/auth/mfa/cliente/{idCliente}/validate/{tipoEnvio}`
- **Recurso:** `LoginMfaResourceV4` (`@TokenAuthenticated`)
- **Serviço:** `TokenServiceV4.confirmarCodigoSms/Email()` → `LoginMfaServiceV4.requestLogin()`
- **Erros funcionais:** `VERIFICATION_CODE_INVALID` | `VERIFICATION_CODE_EXPIRED`
- **Sucesso:** reemite tokens com `mfaTokenIsValid=true`; se senha válida, registra `MFA_VALIDADO` no cliente

#### POST `/v4/auth/redefinir-senha-expirada`
- **Recurso:** `LoginResourceV4` (`@PasswordExpiredAuthenticated`)
- **Serviço:** `LoginServiceV4.resetPasswordExpired()`
- **Entrada:** `{ newPassword, confirmPassword }`
- **Validações:** confirmação de senha, política de senha forte (`StrongPasswordValidator`)
- **Erros:** `PASSWORD_POLICY_VIOLATION` | `PASSWORD_SAME_AS_PREVIOUS`
- **Sucesso:** atualiza senha, registra `MFA_VALIDADO`, reemite token final

### 2.6 Banco de Dados

```sql
-- Tabela de controle de OTP
AV_CODIGO_VERIFICACAO (
  id, clienteId, destino, codigo, tipo (SMS|EMAIL),
  dataHoraCriacao, dataHoraExpiracao, validado
)

-- Campo relevante no cliente
AV_CLIENTE.MFA_VALIDADO  -- data/hora da última validação MFA
AV_CLIENTE.DATA_ALTERACAO_SENHA  -- usado para checar expiração de senha
```

### 2.7 Parâmetros de Sistema Dinâmicos

| Parâmetro | Descrição |
|---|---|
| `EXPIRACAO_MFA` | Janela em dias sem revalidar MFA (Flyway: `V2026.04.08...50005`) |
| `EXPIRACAO_SENHA` | Janela em dias para senha expirar (Flyway: `V2026.04.08...50006`) |

---

## 3. Projeto: `fe-canaisdigitais` (Frontend Angular)

### 3.1 Tecnologias e Padrões

| Item | Detalhe |
|---|---|
| Linguagem | TypeScript |
| Framework | Angular (SSR via server/) |
| Auth | `@acn/angular` — `AcnAuthenticationService` |
| HTTP | `AcnConnectorService` (wrapper de HttpClient) |
| Testes | Jest + jest-setup.ts |
| Estado | NGXS / serviços de store por domínio |
| Build | Angular CLI + Docker |

### 3.2 Estrutura de Arquivos — Autenticação MFA

```
fe-canaisdigitais/src/app/
├── features/login/
│   ├── services/
│   │   ├── login.service.ts               ← orquestrador: onLogin, navigateAfterLogin,
│   │   │                                    completeAuthenticatedSession, updateExpiredPassword
│   │   └── login-mfa-flow.service.ts      ← máquina de estado do fluxo MFA
│   │                                        (sessionStorage, claims JWT, routing)
│   └── components/
│       └── (telas: login, verificação OTP, senha expirada)
├── shared/
│   ├── services/
│   │   └── sms/
│   │       └── sms.service.ts             ← sendLoginMfa(), validateLoginMfa()
│   └── guards/
│       ├── sms-token.guard.ts             ← protege rota de OTP: exige mfaTokenIsValid=false
│       ├── login-mfa-verification.guard.ts← valida contexto resumível de verificação MFA
│       └── login-password-expired.guard.ts← protege rota de senha-expirada via LoginMfaFlowService
└── @acn/angular (lib externa)
    └── AcnAuthenticationService           ← login, logout, getAccessToken, doLoginUser,
                                             getClienteIdFromJwt, jwtHelper.decodeToken
```

### 3.3 `LoginMfaFlowService` — Máquina de Estado

Responsável por gerenciar o estado transiente do fluxo MFA no `sessionStorage`.

**Interface de contexto:**
```typescript
interface ILoginMfaFlowContext {
  clienteId: string;       // ID do cliente (do JWT)
  mfaType: 'sms' | 'email'; // canal escolhido
  initialSendCompleted: boolean; // OTP já foi enviado ao menos 1 vez
  lastSendAt: number;      // timestamp do último envio (controle de 150s)
}
```

**Métodos principais:**

| Método | Responsabilidade |
|---|---|
| `isPreMfaAuthenticated()` | `mfaTokenIsValid !== true` — MFA ainda pendente |
| `canAccessPasswordExpiredRoute()` | `mfaTokenIsValid=true && passwordExpired=true` |
| `resolveAuthenticatedRoute()` | decide rota: `login/verificacao-metodo`, `login/senha-expirada` ou `home-logado` |
| `setInitialSendCompleted(clienteId, mfaType)` | persiste contexto após primeiro envio OTP |
| `getRemainingResendSeconds(windowSeconds)` | calcula segundos restantes para reenvio (janela 150s) |
| `hasResumableVerificationContext()` | verifica se o contexto de verificação pode ser retomado |
| `invalidateTransientAuthentication()` | limpa sessionStorage + faz logout (estado corrompido) |
| `clear()` | limpa contexto do sessionStorage após autenticação final |

**Regra de persistência da sessão:**
- Token **pré-MFA** (`mfaTokenIsValid !== true`) → **nunca persiste** como sessão final
- Token **pós-MFA com senha expirada** (`passwordExpired=true`) → estado transiente, não final
- Token **final** (`mfaTokenIsValid=true && passwordExpired!=true`) → persiste como sessão autenticada

### 3.4 `SmsService` — Integração com API de MFA

```typescript
// Enviar OTP
sendLoginMfa(clienteId: string, mfaType: 'sms' | 'email')
  → POST /api/proxy/agencia-virtual/v4/auth/mfa/cliente/{clienteId}/send/{mfaType}

// Validar OTP
validateLoginMfa(clienteId: string, token: string, mfaType: 'sms' | 'email')
  → POST /api/proxy/agencia-virtual/v4/auth/mfa/cliente/{clienteId}/validate/{mfaType}
     Body: { token: "XXXXX" }
```

**Erros funcionais tratados:**
```typescript
const LOGIN_MFA_ERROR_REASONS = {
  INVALID_CODE: 'VERIFICATION_CODE_INVALID',
  EXPIRED_CODE: 'VERIFICATION_CODE_EXPIRED',
}
```

### 3.5 Guards de Rota

| Guard | Rota Protegida | Lógica |
|---|---|---|
| `SmsTokenGuard` | `/login/sms` (fluxo legado) | verifica `mfaTokenIsValid` diretamente no JWT |
| `LoginMfaVerificationGuard` | `/login/verificacao` | exige `isPreMfaAuthenticated()` + contexto resumível |
| `LoginPasswordExpiredGuard` | `/login/senha-expirada` | exige `canAccessPasswordExpiredRoute()` |

### 3.6 URLs dos Endpoints (via BFF Proxy)

```
POST /api/login                                               ← login inicial (via AcnAuthenticationService)
POST /api/proxy/agencia-virtual/v4/auth/mfa/cliente/{id}/send/{tipo}   ← enviar OTP
POST /api/proxy/agencia-virtual/v4/auth/mfa/cliente/{id}/validate/{tipo} ← validar OTP
POST /api/proxy/agencia-virtual/v4/auth/redefinir-senha-expirada       ← reset senha expirada
POST /api/login/v3/resetSenha                                 ← reset senha por token
```

---

## 4. Estados do Fluxo MFA (Shared)

```
┌─────────────────────────────────────────────────────┐
│                   ESTADOS DO FLUXO                   │
├─────────────┬──────────────────────┬─────────────────┤
│   Estado    │  mfaTokenIsValid     │  passwordExpired │
├─────────────┼──────────────────────┼─────────────────┤
│ Pré-MFA     │  false               │  any             │
│ Pós-MFA     │  true                │  true            │
│ Final       │  true                │  false           │
└─────────────┴──────────────────────┴─────────────────┘
```

### Fluxo completo (happy path):
1. `POST /v4/auth/login` → token pré-MFA (`mfaTokenIsValid=false`)
2. FE detecta → exibe seleção de canal
3. `POST /v4/auth/mfa/.../send/{tipo}` → OTP enviado ao usuário
4. Usuário informa OTP → `POST /v4/auth/mfa/.../validate/{tipo}` → token pós-MFA
5. Se `passwordExpired=true` → `POST /v4/auth/redefinir-senha-expirada` → token final
6. Se `passwordExpired=false` → token já é final → acesso à área logada

---

## 5. Regras de Negócio Críticas

1. **Nunca tratar retorno do login como sessão final** sem inspecionar `mfaTokenIsValid` e `passwordExpired`.
2. **Token pré-MFA** não pode acessar recursos protegidos por `@Authenticated`.
3. **clienteId** do path param de MFA deve bater com `clienteId` da claim do JWT (validação no backend).
4. **Reenvio de OTP** bloqueado por 150 segundos (controlado por `lastSendAt` no sessionStorage + regra backend).
5. **Janela de revalidação MFA**: se o cliente validou MFA recentemente, o login já retorna `mfaTokenIsValid=true` diretamente (parâmetro `EXPIRACAO_MFA`).
6. **Senha forte obrigatória** na redefinição: `StrongPasswordValidator` valida política (`PASSWORD_POLICY_VIOLATION`).
7. **authFlow discriminator**: tokens do fluxo `legacy` nunca carregam claims de MFA; nunca misturar fluxos.

---

## 6. Testes

### agenciavirtual (JUnit 4 + Mockito)

| Classe de Teste | Cobertura |
|---|---|
| `LoginServiceV4Test` | login inicial, reset senha, validações |
| `LoginMfaServiceV4Test` | geração token pós-MFA, senha expirada, busca celular/email |
| `TokenServiceV4Test` | envio e confirmação OTP SMS/Email |
| `SmsTokenServiceV3Test` | fluxo legado de SMS |

**Padrão de testes:**
```java
@RunWith(MockitoJUnitRunner.class)
public class LoginMfaServiceV4Test {
    @Mock ClienteService clienteService;
    @Mock ParametroSistemaService parametroSistemaService;
    @Mock JWTService jwtService;
    // ...
}
```

### fe-canaisdigitais (Jest)

| Arquivo de Teste | Cobertura |
|---|---|
| `login-mfa-flow.service.spec.ts` | estados do fluxo, sessionStorage, claims |
| `sms.service.spec.ts` | sendLoginMfa, validateLoginMfa, erros |
| `login-mfa-verification.guard.spec.ts` | proteção de rota |
| `login-password-expired.guard.spec.ts` | proteção de rota senha expirada |

**Configuração:** `jest-setup.ts` + `jest.config.js`

---

## 7. Arquivos de Referência Rápida

### agenciavirtual
- [src/corp/ambiental/agencia/auth/resource/v4/LoginResourceV4.java](agenciavirtual/src/corp/ambiental/agencia/auth/resource/v4/LoginResourceV4.java)
- [src/corp/ambiental/agencia/auth/resource/v4/LoginMfaResourceV4.java](agenciavirtual/src/corp/ambiental/agencia/auth/resource/v4/LoginMfaResourceV4.java)
- [src/corp/ambiental/agencia/auth/service/v4/LoginServiceV4.java](agenciavirtual/src/corp/ambiental/agencia/auth/service/v4/LoginServiceV4.java)
- [src/corp/ambiental/agencia/auth/service/v4/LoginMfaServiceV4.java](agenciavirtual/src/corp/ambiental/agencia/auth/service/v4/LoginMfaServiceV4.java)
- [src/corp/ambiental/agencia/auth/service/v4/TokenServiceV4.java](agenciavirtual/src/corp/ambiental/agencia/auth/service/v4/TokenServiceV4.java)
- [src/corp/ambiental/agencia/auth/service/v4/MfaDeliveryType.java](agenciavirtual/src/corp/ambiental/agencia/auth/service/v4/MfaDeliveryType.java)
- [src/corp/ambiental/agencia/core/service/JWTService.java](agenciavirtual/src/corp/ambiental/agencia/core/service/JWTService.java)
- [src/corp/ambiental/agencia/parametrosistema/service/ParametroSistemaService.java](agenciavirtual/src/corp/ambiental/agencia/parametrosistema/service/ParametroSistemaService.java)

### fe-canaisdigitais
- [src/app/features/login/services/login.service.ts](fe-canaisdigitais/src/app/features/login/services/login.service.ts)
- [src/app/features/login/services/login-mfa-flow.service.ts](fe-canaisdigitais/src/app/features/login/services/login-mfa-flow.service.ts)
- [src/app/shared/services/sms/sms.service.ts](fe-canaisdigitais/src/app/shared/services/sms/sms.service.ts)
- [src/app/shared/guards/sms-token.guard.ts](fe-canaisdigitais/src/app/shared/guards/sms-token.guard.ts)
- [src/app/shared/guards/login-mfa-verification.guard.ts](fe-canaisdigitais/src/app/shared/guards/login-mfa-verification.guard.ts)
- [src/app/shared/guards/login-password-expired.guard.ts](fe-canaisdigitais/src/app/shared/guards/login-password-expired.guard.ts)
