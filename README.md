# Dispensador de Medicamentos – Firmware Arduino Mega 2560

Firmware do protótipo de dispensador de medicamentos controlados para ambiente hospitalar, desenvolvido como parte de um Trabalho de Conclusão de Curso.  
O sistema faz o controle de acesso via biometria (digital), registra usuários com um ID numérico, permite retirada de medicamentos, aciona motores do protótipo e integra um leitor de código de barras e um sensor fotoelétrico para conferência do medicamento.

---

## Visão geral

A ideia do projeto é ter um dispensador que:

- Só libera o acesso após autenticação biométrica.
- Associa cada digital a um ID numérico (por exemplo, matrícula, código interno etc.).
- Disponibiliza um menu de usuário simples para:
  - Retirar medicamento.
  - Ver a lista de usuários cadastrados.
- Possui uma área administrativa protegida por senha, para:
  - Registrar novos usuários.
  - Listar usuários.
  - Excluir usuários.

Toda a interface é feita em um LCD 20x4 I2C com um encoder rotativo para navegação e dois botões físicos (Confirmar / Voltar).

---

## Funcionalidades principais

- **Autenticação por digital**
  - Utiliza o sensor de digitais compatível com a biblioteca `Adafruit_Fingerprint`.
  - Verifica se a digital está associada a um usuário cadastrado.

- **Cadastro de usuários (Admin)**
  - Associação: digital + ID numérico de 8 dígitos.
  - Armazena as informações na EEPROM do Arduino Mega.
  - Busca o primeiro “slot” livre de usuário automaticamente.

- **Menu do usuário**
  - `Retirar Medic.` – inicia o ciclo de retirada do medicamento (motores + sensor + leitor de código de barras).
  - `Ver Lista` – lista os IDs numéricos dos usuários cadastrados.
  - `Voltar` – retorna para a tela inicial.

- **Área administrativa**
  - Protegida por senha numérica (`ADMIN_PASSWORD` no código).
  - Opções:
    - `Registrar` – novo cadastro de usuário.
    - `Lista Usuarios` – exibe IDs dos usuários.
    - `Excluir` – remove usuário (digital + ID) da memória.
    - `Voltar` – retorna à tela inicial.

- **Retirada de medicamento**
  - Sequência automática:
    1. Usuário confirma a retirada.
    2. Motor do tambor gira por um tempo configurado, liberando a caixinha.
    3. Motor da correia move o medicamento até a região do sensor.
    4. Sensor fotoelétrico detecta a presença da caixinha.
    5. Ao detectar, o sistema dispara o trigger do leitor de código de barras.
    6. Código lido é enviado pela Serial para conferência.
    7. Ciclo termina e o sistema volta para o menu do usuário.

---

## Hardware utilizado

- **Placa**: Arduino Mega 2560.
- **Leitor biométrico**: sensor de digitais compatível com `Adafruit_Fingerprint`.
- **Leitor de código de barras**: módulo MH-ETLiveScanner (ou similar).
- **Display**: LCD 20x4 I2C (endereço padrão `0x27`).
- **Encoder rotativo**: usado para navegação de menus e seleção de dígitos.
- **Botões físicos**:
  - Botão de **Confirmar** .
  - Botão de **Voltar/Cancelar** .
- **Sensor fotoelétrico**: saída digital (HIGH quando detecta o medicamento).
- **Motores / drivers**:
  - Motor do tambor.
  - Motor da correia transportadora.

> Obs.: os motores devem ser ligados através de drivers/relés próprios. O Arduino só fornece o sinal lógico (HIGH/LOW).

---

## Mapeamento de pinos

### Interface de usuário

| Função                | Pino Arduino | Observações                            |
|-----------------------|-------------:|----------------------------------------|
| Encoder CLK           | 45           | `INPUT_PULLUP`                         |
| Encoder DT            | 44           | `INPUT_PULLUP`                         |
| Botão Confirmar       | 22           | `INPUT_PULLUP`, ativo em nível LOW     |
| Botão Voltar/Cancelar | 24           | `INPUT_PULLUP`, ativo em nível LOW     |
| LCD I2C SDA           | SDA (20)     | Barramento I2C                         |
| LCD I2C SCL           | SCL (21)     | Barramento I2C                         |

### Biometria e scanner

| Função                     | Pino Arduino | Observações                                |
|----------------------------|-------------:|--------------------------------------------|
| Leitor biométrico (RX/TX)  | Serial2      | Usa `Serial2` do Mega (`fingerSerial`)     |
| Leitor código de barras RX | 52           | `mySerial(52, 53)`                         |
| Leitor código de barras TX | 53           |                                            |

### Atuadores e sensores

| Função              | Pino Arduino | Observações                         |
|---------------------|-------------:|-------------------------------------|
| Motor do tambor     | 28           | Saída digital para driver de motor |
| Motor da correia    | 30           | Saída digital para driver de motor |
| Sensor fotoelétrico | 26           | Entrada digital (HIGH quando detecta)|

---

## Fluxo de telas e estados

A lógica do firmware é organizada em estados principais:

- `STATE_MAIN` – Tela inicial:
  - Opções: `Entrar` / `Admin`.
- `STATE_ENTER` – Leitura da digital:
  - Em caso de sucesso, vai para `STATE_USER_MENU`.
- `STATE_USER_MENU` – Menu do usuário:
  - `Retirar Medic.`
  - `Ver Lista`
  - `Voltar`
- `STATE_ADMIN_MENU` – Menu do administrador:
  - `Registrar`
  - `Lista Usuarios`
  - `Excluir`
  - `Voltar`
- `STATE_ENROLL` – Cadastro de usuário:
  - Coleta digital.
  - Pede ID numérico (8 dígitos) via encoder.
- `STATE_DELETE` – Exclusão de usuário.
- `STATE_LIST` – Lista de usuários (IDs numéricos).
- `STATE_WITHDRAW` – Ciclo de retirada de medicamento.

A navegação entre opções é feita girando o encoder, e as ações são confirmadas com o botão **Confirmar** ou canceladas com o botão **Voltar**.

---

## Estrutura básica do código

O código principal (arquivo `.ino`) é organizado em blocos:

- **Configurações e constantes**
  - Defines de estados, limites, tempos de acionamento e pinos.
  - Definição da senha de administrador:  
    ```cpp
    const char ADMIN_PASSWORD[] = "012345";
    ```
- **Estrutura `User`**
  - Marca se o slot está em uso (`inUse`) e guarda o `numericID` (8 dígitos).
- **Encoder**
  - Implementação em polling, com tabela de transição em quadratura e filtro de debounce.
- **Leitura de botões**
  - Função `getConfirmCancelInput()` centraliza o debounce e o mapeamento para `'C'` (confirmar) e `'E'` (voltar).
- **Menus / LCD**
  - Funções de desenho de tela:
    - `showMainMenu()`
    - `showAdminMenu()`
    - `showUserMenu()`
    - `showWithdrawIntro()`
  - Função de utilidade `lcd_write_line()` para alinhar texto nas 20 colunas.
- **Lista de usuários**
  - `handleList()` administra o estado `STATE_LIST` utilizando o encoder e os botões.
- **Cadastro e exclusão**
  - `runEnrollment()` – fluxo de registro de digital + ID numérico.
  - `runDelete()` – seleção de usuário e chamada de `deleteFingerprint()`.
- **Biometria**
  - `getFingerprintEnroll()` – cadastro da digital no sensor.
  - `getFingerprintID()` – identificação.
  - `deleteFingerprint()` – exclusão de template do sensor + limpeza da EEPROM.
- **EEPROM**
  - `loadUsersFromEEPROM()` – recupera todos os usuários na inicialização.
  - `saveUserToEEPROM()` – grava/atualiza o slot de um usuário.
- **Retirada de medicamento**
  - `handleWithdraw()` – máquina de estados que coordena:
    - Motor do tambor.
    - Motor da correia.
    - Leitura do sensor fotoelétrico.
    - Trigger do leitor de código de barras.
- **Scanner de código de barras**
  - `sendTrigger()` – envia comando de leitura (`"~T."`).
  - `scannerStateMachine()` – trata recebimento dos dados, filtro de ACK e repetição de trigger em caso de timeout.

---

## Como compilar e gravar

1. Abrir o código no **Arduino IDE**.
2. Selecionar a placa:
   - `Ferramentas > Placa > Arduino AVR > Arduino Mega or Mega 2560`.
3. Selecionar a porta correta (COM) onde o Mega está conectado.
4. Instalar as bibliotecas necessárias:
   - **Adafruit Fingerprint Sensor Library**
     - (Geralmente listada como “Adafruit Fingerprint Sensor Library” na Library Manager)
   - **LiquidCrystal_I2C** (alguma implementação compatível com `LiquidCrystal_I2C lcd(0x27, 20, 4);`)
   - `SoftwareSerial`, `EEPROM`, `Wire` já são nativas.
5. Verificar (⌁) e enviar (→) o código para a placa.

---

## Configurações importantes

- **Senha do administrador**

  A senha é definida por uma string numérica:

  ```cpp
  const char ADMIN_PASSWORD[] = "012345";
