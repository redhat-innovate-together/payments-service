Mikroslužba: Platby (Payments)
Tento dokument poskytuje detailní popis a technickou dokumentaci pro mikroslužbu Platby, která je klíčovou součástí e-commerce platformy rachota.cz.

🚀 Popis a účel
Mikroslužba Platby je zodpovědná za kompletní životní cyklus platebních transakcí. Jejím hlavním úkolem je vytvářet platby, bezpečně komunikovat s externími platebními bránami (např. Adyen, GoPay, PayPal), zpracovávat jejich odpovědi (callbacky) a udržovat konzistentní stav plateb napříč celým systémem.

Služba funguje jako centrální autorita pro všechny finanční transakce spojené s objednávkami a zajišťuje oddělení platební logiky od ostatních částí systému.

Klíčové funkce
Vytvoření platby: Iniciace nové platby pro konkrétní objednávku.

Zpracování různých platebních metod: Podpora plateb kartou, bankovním převodem, na dobírku, Apple Pay, Google Pay atd.

Zpracování callbacků: Příjem a validace asynchronních notifikací od platebních bran o změně stavu platby (zaplaceno, selhalo, zrušeno).

Dotazování na stav: Poskytování aktuálního stavu platby ostatním mikroslužbám (např. Objednávkám).

Správa vratek (refunds): Zajištění procesu vrácení peněz zákazníkovi.

Zabezpečení: Zajištění bezpečnosti citlivých dat a komunikace v souladu se standardy PCI DSS.

🛠️ Technologický stack
Jazyk & Framework: C# / .NET 8

Datová vrstva: PostgreSQL, Dapper

Asynchronní komunikace: RabbitMQ (princip Event-Driven Architecture)

Kontejnerizace: Docker

Orchestrace: Kubernetes

Caching: Redis

📖 API Dokumentace (OpenAPI 3.0)
Následuje specifikace RESTful API, které služba poskytuje pro synchronní komunikaci.

openapi: 3.0.1
info:
  title: Rachota Payments API
  description: |-
    API pro správu platebních transakcí v rámci ekosystému rachota.cz.
    Umožňuje vytvářet platby, zjišťovat jejich stav a iniciovat vratky.
  version: 1.0.0
servers:
  - url: https://api.rachota.cz/payments
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
          description: Platba úspěšně vytvořena. V odpovědi je `redirectUrl` pro platební bránu.
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
  /api/v1/payments/{paymentId}:
    get:
      tags:
        - Payments
      summary: Získání stavu platby
      description: Vrátí detailní informace a aktuální stav konkrétní platby.
      operationId: getPaymentById
      parameters:
        - name: paymentId
          in: path
          required: true
          description: Unikátní identifikátor platby (UUID).
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Úspěšně nalezeno.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/PaymentDetailedResponse'
        '404':
          description: Platba s daným ID nebyla nalezena.
  /api/v1/payments/{paymentId}/refunds:
    post:
      tags:
        - Refunds
      summary: Iniciace vratky (refund)
      description: Zahájí proces vrácení peněz pro již zaplacenou transakci. Lze vrátit celou nebo jen část částky.
      operationId: createRefund
      parameters:
        - name: paymentId
          in: path
          required: true
          description: ID původní platby, která má být vrácena.
          schema:
            type: string
            format: uuid
      requestBody:
        description: Specifikace částky k vrácení. Pokud není uvedeno, vrací se celá částka.
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/RefundRequest'
      responses:
        '202':
          description: Požadavek na vrácení byl přijat ke zpracování.
        '400':
          description: Neplatný požadavek (např. platba není ve stavu pro vrácení).
        '404':
          description: Původní platba nebyla nalezena.

components:
  schemas:
    PaymentRequest:
      type: object
      required:
        - orderId
        - amount
        - currency
        - paymentMethod
        - redirectUrls
      properties:
        orderId:
          type: integer
          description: ID objednávky, ke které platba patří.
          example: 123456789
        amount:
          type: number
          format: double
          description: Celková částka k zaplacení.
          example: 1599.90
        currency:
          type: string
          description: Měna platby (ISO 4217 kód).
          example: "CZK"
        paymentMethod:
          type: string
          description: Zvolená platební metoda.
          enum: [CARD, BANK_TRANSFER, APPLE_PAY, GOOGLE_PAY]
          example: "CARD"
        redirectUrls:
          type: object
          properties:
            successUrl:
              type: string
              format: uri
              description: URL, na kterou bude uživatel přesměrován po úspěšné platbě.
              example: "https://www.rachota.cz/objednavka/dekujeme"
            failureUrl:
              type: string
              format: uri
              description: URL, na kterou bude uživatel přesměrován po neúspěšné platbě.
              example: "https://www.rachota.cz/objednavka/platba-se-nezdarila"

    PaymentResponse:
      type: object
      properties:
        paymentId:
          type: string
          format: uuid
          description: Unikátní identifikátor platby vygenerovaný systémem.
        status:
          type: string
          description: Aktuální stav platby.
          enum: [PENDING, PAID, FAILED, CANCELED, REFUNDED]
        redirectUrl:
          type: string
          format: uri
          description: URL platební brány, na kterou má být zákazník přesměrován.

    PaymentDetailedResponse:
      allOf:
        - $ref: '#/components/schemas/PaymentResponse'
        - type: object
          properties:
            orderId:
              type: integer
              description: ID přidružené objednávky.
            amount:
              type: number
              format: double
            currency:
              type: string
            createdAt:
              type: string
              format: date-time
            updatedAt:
              type: string
              format: date-time

    RefundRequest:
      type: object
      properties:
        amount:
          type: number
          format: double
          description: Částka k vrácení. Pokud není specifikováno, vrací se celá původní částka.
          example: 500.00
        reason:
          type: string
          description: Důvod vrácení peněz.
          example: "Vrácení zboží"

    ErrorResponse:
      type: object
      properties:
        errorCode:
          type: string
          example: "INVALID_CURRENCY"
        message:
          type: string
          example: "Měna 'XYZ' není podporována."

📨 Asynchronní komunikace (Eventy)
Služba intenzivně využívá asynchronní komunikaci pro zajištění oddělení (decoupling) od ostatních částí systému.

Publikované eventy
Když dojde k významné změně stavu platby, služba publikuje zprávu do RabbitMQ.

payment.succeeded

Routing Key: payment.succeeded

Payload: Obsahuje paymentId, orderId, amount, paidAt.

Konzumenti: Služba Objednávky (pro posun objednávky do stavu "Zaplaceno"), Sklad (pro zahájení expedice).

payment.failed

Routing Key: payment.failed

Payload: Obsahuje paymentId, orderId, reason.

Konzumenti: Služba Objednávky (pro zrušení objednávky nebo notifikaci zákazníka), Notifikační služba.

payment.refunded

Routing Key: payment.refunded

Payload: Obsahuje paymentId, orderId, refundedAmount.

Konzumenti: Účetnictví, Služba Objednávky.

💻 Spuštění v lokálním prostředí
Pro spuštění služby na lokálním stroji postupujte následovně.

Požadavky
.NET 8 SDK

Docker Desktop (pro spuštění databáze a RabbitMQ)

Postup
Naklonování repozitáře

git clone https://github.com/rachota-cz/payments-service.git
cd payments-service

Spuštění závislostí
Spusťte databázi (PostgreSQL) a message broker (RabbitMQ) pomocí Docker Compose.

docker-compose up -d

Konfigurace
Zkopírujte appsettings.template.json na appsettings.Development.json a vyplňte potřebné connection stringy a klíče k platebním bránám (pro testovací prostředí).

{
  "ConnectionStrings": {
    "Database": "Host=localhost;Port=5432;Database=rachota_payments;Username=user;Password=password"
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

Spuštění aplikace

dotnet run --project src/Rachota.Payments.Api/Rachota.Payments.Api.csproj

Služba bude ve výchozím stavu naslouchat na http://localhost:5010.

📞 Kontakty
Tým: Phoenix

Team Lead: František Vomáčka

Slack kanál: #team-rachota-payments
