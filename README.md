# Mikroslužba: Platby Astra

Tento dokument poskytuje detailní popis a technickou dokumentaci pro mikroslužbu **Platby**, která je klíčovou součástí e-commerce platformy **Astra**.

---

## 🚀 Popis a účel

Mikroslužba **Platby** je zodpovědná za kompletní životní cyklus platebních transakcí v platformě Astra. Jejím hlavním úkolem je vytvářet platby, bezpečně komunikovat s externími platebními bránami (např. Adyen, GoPay, PayPal), zpracovávat jejich odpovědi (callbacky) a udržovat konzistentní stav plateb napříč celým systémem.

Služba funguje jako centrální autorita pro všechny finanční transakce spojené s objednávkami a zajišťuje oddělení platební logiky od ostatních částí systému.

### Klíčové funkce

* **Vytvoření platby**: Iniciace nové platby pro konkrétní objednávku.
* **Zpracování různých platebních metod**: Podpora plateb kartou, bankovním převodem, na dobírku, Apple Pay, Google Pay atd.
* **Zpracování callbacků**: Příjem a validace asynchronních notifikací od platebních bran o změně stavu platby (zaplaceno, selhalo, zrušeno).
* **Dotazování na stav**: Poskytování aktuálního stavu platby ostatním mikroslužbám (např. Objednávkám).
* **Správa vratek (refunds)**: Zajištění procesu vrácení peněz zákazníkovi.
* **Zabezpečení**: Zajištění bezpečnosti citlivých dat a komunikace v souladu se standardy PCI DSS.

---

## 🛠️ Technologický stack mikroslužby Platby Astra

* **Jazyk & Framework**: C# / .NET 8
* **Datová vrstva**: PostgreSQL, Dapper
* **Asynchronní komunikace**: RabbitMQ (princip Event-Driven Architecture)
* **Kontejnerizace**: Docker
* **Orchestrace**: Kubernetes
* **Caching**: Redis

---

## 📖 API Dokumentace (AstraAPI 3.0)

```yaml
Následuje specifikace RESTful API, které služba poskytuje pro synchronní komunikaci:


openapi: 3.0.1
info:
  title: Astra Payments API
  description: |-
    API pro správu platebních transakcí v rámci ekosystému Astra.
    Umožňuje vytvářet platby, zjišťovat jejich stav a iniciovat vratky.
  version: 1.0.0
servers:
  - url: [https://api.astra.cz/payments](https://api.astra.cz/payments)
    description: Produkční prostředí
  - url: http://localhost:5010
    description: Lokální vývojové prostředí

paths:
  /api/v1/payments:
    post:
      tags:
        - Payments
      summary: Vytvoření nové platby
      description: Vytvoří novou platební transakci pro zadanou objednávku a vrátí URL pro přesměrování na platební bránu.
      operationId: createPayment
      requestBody:
        description: Data pro vytvoření platby
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/PaymentRequest'
      responses:
        '201':
          description: Platba úspěšně vytvořena. V odpovědi je redirectUrl pro platební bránu.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/PaymentResponse'
        '400':
          description: Neplatný požadavek (např. chybějící data, nevalidní měna).
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'

```          
---

## 📨 Asynchronní komunikace (Eventy)

Služba Astra intenzivně využívá asynchronní komunikaci pro zajištění oddělení (decoupling) od ostatních částí systému.

### Publikované eventy

Když dojde k významné změně stavu platby, služba publikuje zprávu do RabbitMQ.

**`payment.succeeded`**
* **Routing Key**: `payment.succeeded`
* **Payload**: Obsahuje `paymentId`, `orderId`, `amount`, `paidAt`.
* **Konzumenti**: Služba Objednávky (pro posun objednávky do stavu "Zaplaceno"), Sklad (pro zahájení expedice).

**`payment.failed`**
* **Routing Key**: `payment.failed`
* **Payload**: Obsahuje `paymentId`, `orderId`, `reason`.
* **Konzumenti**: Služba Objednávky (pro zrušení objednávky nebo notifikaci zákazníka), Notifikační služba.

**`payment.refunded`**
* **Routing Key**: `payment.refunded`
* **Payload**: Obsahuje `paymentId`, `orderId`, `refundedAmount`.
* **Konzumenti**: Účetnictví, Služba Objednávky.

---

## 💻 Spuštění Mikroslužby Astra v lokálním prostředí

Pro spuštění služby na lokálním stroji postupujte následovně.

### Požadavky

* .NET 8 SDK
* Docker Desktop (pro spuštění databáze a RabbitMQ)

### Postup

1.  **Naklonování repozitáře Astra**
    ```bash
    git clone [https://github.com/astra-cz/payments-service.git](https://github.com/astra-cz/payments-service.git)
    cd payments-service
    ```

2.  **Spuštění závislostí Astra**
    Spusťte databázi (PostgreSQL) a message broker (RabbitMQ) pomocí Docker Compose.
    ```bash
    docker-compose up -d
    ```

3.  **Konfigurace Astra**
    Zkopírujte `appsettings.template.json` na `appsettings.Development.json` a vyplňte potřebné connection stringy a klíče k platebním bránám (pro testovací prostředí).
    ```json
    {
      "ConnectionStrings": {
        "Database": "Host=localhost;Port=5432;Database=astra_payments;Username=user;Password=password"
      },
      "RabbitMQ": {
        "HostName": "localhost"
      },
      "PaymentGateways": {
        "Adyen": {
          "ApiKey": "..."
        }
      }
    }
    ```

4.  **Spuštění aplikace Astra**
    ```bash
    dotnet run --project src/Astra.Payments.Api/Astra.Payments.Api.csproj
    ```
    Služba bude ve výchozím stavu naslouchat na `http://localhost:5010`.
