# CryptoBlockchain-ETL
---
## **1. Úvod a popis zdrojových dát**
V tejto práci sa zameriavame na analýzu transakčných dát z blockchain sietí kryptomien Bitcoin, Ethereum a Litecoin. Cieľom je porozumieť:
- dynamike transakcií v jednotlivých blockchainoch,
- efektivite poplatkov za transakcie (gas fees),
- správe adries a kontraktov,
- trendom v objeme transakcií a ich časovom správaní.

Zdrojové dáta pochádzajú z databázy Amberdata Blockchain dostupnej prostredníctvom Snowflake Marketplace na adrese BLOCKCHAIN_CRYPTO_DATA.AMBERDATA_BLOCKCHAIN. Dataset obsahuje štruktúrované údaje o pending transakciách z troch hlavných kryptomien, ktoré boli spracované do jednotnej staging tabuľky UNIFIED_STG.

Účelom ELT procesu bolo zlúčiť heterogénne blockchain dáta, očistiť ich a transformovať do hviezdneho schémy, ktorá umožňuje komplexnú analýzu transakčnej aktivity naprieč rôznymi kryptomenami.

#### Biznis kontext a význam analýzy
Analyzované dáta podporujú blockchain monitoring a kryptomenové analytické procesy v oblasti:
- Tréningu ML modelov na predikciu poplatkov
- Rizikového manažmentu adries (whitelisting/blacklisting)
- Optimalizácie transakcií podľa siete a času
- Benchmarkingu výkonu rôznych blockchainov

Výsledná hviezdna schéma umožňuje identifikovať trendy ako najaktívnejšie hodiny, najefektívnejšie siete alebo anomálie v transakčnej aktivite, čím poskytuje praktické insighty pre kryptomenové burzy, wallet providerov a DeFi platformy.
---
## **3. ELT proces v Snowflake**
Proces ELT zahŕňa tri základné kroky – extrakciu (Extract), načítanie (Load) a transformáciu (Transform) dát. V prostredí Snowflake bol tento postup aplikovaný s cieľom spracovať a pripraviť údaje zo staging vrstvy do podoby viacdimenzionálneho dátového modelu, ktorý umožňuje efektívnu analýzu a následnú tvorbu vizualizačných výstupov.

---
### **3.1 Extract (Extrahovanie dát)**
Údaje pochádzali zo zdrojového datasetu dostupného v službe Snowflake Marketplace a boli spracované prostredníctvom vytvorenia staging tabuľky. Na tento účel bol použitý SQL príkaz, ktorý zabezpečil načítanie dát z existujúcej databázy do dočasnej staging vrstvy určenéj na ďalšie spracovanie. Tento krok zároveň vytvoril základ pre následné transformácie a modelovanie dát do viacdimenzionálnej štruktúry typu hviezda.

#### Príklad kódu:
```sql
CREATE OR REPLACE TABLE STG_BTC_PENDING AS
SELECT * FROM BLOCKCHAIN_CRYPTO_DATA.AMBERDATA_BLOCKCHAIN.BITCOIN_PENDING_TRANSACTION;
```

---
### **3.2 Load (Načítanie dát)**

V rámci fázy spracovania dát v staging vrstve bol realizovaný proces zlúčenia viacerých dočasných tabuliek do jednotnej štruktúry. Na tento účel bol v prostredí Snowflake použitý SQL príkaz CREATE OR REPLACE TABLE, ktorým sa vytvorila integrovaná staging tabuľka s názvom UNIFIED_STG.

V uvedenom postupe boli dáta z troch samostatných zdrojov – STG_BTC_PENDING, STG_ETH_PENDING a STG_LTC_PENDING – skombinované prostredníctvom operátora UNION ALL. Tento prístup umožnil spojiť údaje o viacerých kryptomenách do jednej konsolidovanej tabuľky. Súčasťou každého výberu bolo aj doplnenie nového atribútu coin_name, ktorý identifikuje príslušnosť záznamu ku konkrétnej kryptomene (napr. Bitcoin, Ethereum, Litecoin).

Po vytvorení tabuľky sa uskutočnila kontrola jej obsahu príkazom SELECT * FROM UNIFIED_STG, čím sa overila správnosť zlúčenia dát. Následne bol použitý príkaz DESCRIBE TABLE UNIFIED_STG na zobrazenie metadát tabuľky, ako sú názvy stĺpcov, ich dátové typy a štruktúra výsledného datasetu.
Do stage boli následne nahraté súbory obsahujúce údaje o knihách, používateľoch, hodnoteniach, zamestnaniach a úrovniach vzdelania. Dáta boli importované do staging tabuliek pomocou príkazu `COPY INTO`. Pre každú tabuľku sa použil podobný príkaz:

#### Príklad kódu:
```sql
CREATE OR REPLACE TABLE UNIFIED_STG AS
SELECT *, 'BITCOIN' AS coin_name
FROM STG_BTC_PENDING
UNION ALL
SELECT *,'ETHEREUM' AS coin_name
FROM STG_ETH_PENDING
UNION ALL
SELECT *,'LITECOIN' AS coin_name
FROM STG_LTC_PENDING;
SELECT * FROM UNIFIED_STG;
```

---
### **3.3 Transfor (Transformácia dát)**
Tu je tvoj text prepísaný inými slovami do formálneho akademického štýlu, pričom som nahradil pôvodné príklady tvojimi SQL príkazmi pre dimenzie a faktovú tabuľku:

V tejto etape spracovania boli údaje zo staging vrstvy vyčistené, transformované a obohatené s cieľom vytvoriť dimenzie a faktovú tabuľku, ktoré tvoria základ hviezdneho schémy pre efektívnu analytickú činnosť.

Dimenzie boli navrhnuté tak, aby poskytovali kontextuálne informácie pre faktovú tabuľku a podporovali multidimenzionálnu analýzu. Každá dimenzia využíva hash funkcie na vytvorenie unikátnych surrogátnych kľúčov, čím sa zabezpečuje konzistentnosť a efektívna integrita referencií.

Príkladom je dimenzia DIM_COIN, ktorá agreguje informácie o kryptomenách vrátane identifikátorov blockchainu a overenia ich platnosti:

#### Príklad kódu: dim_coin
```sql
CREATE OR REPLACE TABLE DIM_COIN AS (
    SELECT DISTINCT coin_name, "blockchainId",
        CASE "blockchainId"
            WHEN '408fa195a34b533de9ad9889f076045e' THEN 'BITCOIN'
            WHEN '1c9c969065fcd1cf' THEN 'ETHEREUM'
            WHEN 'f94be61fd9f4fa684f992ddfd4e92272' THEN 'LITECOIN'
            ELSE 'UNKNOWN'
        END AS blockchain_id_check,
        HASH(coin_name, "blockchainId") AS id_coin
    FROM UNIFIED_STG
);
```
Podobne boli vytvorené ďalšie dimenzie:

DIM_BLOCK – informácie o blokoch (hash, číslo bloku)

DIM_TRANSACTION_TYPE – typy transakcií (status, coinbase)

DIM_ADDRESS – adresy z poľa from, to, contractAddress

DIM_TIME – časové atribúty (dátum, rok, mesiac, hodina)

DIM_GAS – metriky súvisiace s plynom

DIM_OP_PROTOCOL – protokolové parametre transakcií

Faktová tabuľka FACT_TRANSACTIONS obsahuje kľúčové metriky transakcií (value, fee, nonce) spolu s surrogátnymi kľúčmi všetkých dimenzií. Obsahuje aj pokročilé analytické metriky získané window funkciami, ako je poradie transakcie v bloku (tx_order_in_block) a kumulatívny súčet poplatkov (cumulative_fee_in_block).

#### Príklad kódu: fact_transactions
```sql
CREATE OR REPLACE TABLE FACT_TRANSACTIONS AS (
    SELECT stg."hash" AS transaction_hash, c.id_coin, b.id_block, tt.id_transaction_type,
           ti.id_time, g.id_gas, op.id_op_protocol, af.id_address AS id_from_address,
           at.id_address AS id_to_address, ac.id_address AS id_contract_address,
           -- METRIKY
           stg."value", stg."fee", stg."nonce", stg."numLogs", stg."transactionIndex", stg."status",
           -- WINDOW FUNCTION #1
           ROW_NUMBER() OVER (PARTITION BY b.id_block ORDER BY stg."transactionIndex") AS tx_order_in_block,
           SUM(stg."fee") OVER (PARTITION BY b.id_block ORDER BY stg."transactionIndex" 
                                ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS cumulative_fee_in_block,
           -- TECHNICKÉ
           stg."input", stg."logsBloom", stg."r", stg."s", stg."v", stg."accessList"
    FROM UNIFIED_STG stg
    JOIN DIM_COIN c ON stg.coin_name = c.coin_name
    JOIN DIM_BLOCK b ON stg."blockHash" = b."blockHash"
    JOIN DIM_TRANSACTION_TYPE tt ON stg."transactionTypeId" = tt."transactionTypeId" 
                                  AND stg."type" = tt."type" AND stg."status" = tt."status"
    JOIN DIM_TIME ti ON stg."timestamp" = ti."timestamp"
    LEFT JOIN DIM_GAS g ON stg."gas" = g."gas"
    LEFT JOIN DIM_OP_PROTOCOL op ON stg."opProSize" = op."opProSize"
    LEFT JOIN DIM_ADDRESS af ON stg."from" = af.address
    LEFT JOIN DIM_ADDRESS at ON stg."to" = at.address
    LEFT JOIN DIM_ADDRESS ac ON stg."contractAddress" = ac.address
);
```
V rámci realizácie ELT procesu v prostredí Snowflake boli spracované pôvodné údaje pochádzajúce z databázového zdroja dostupného prostredníctvom Snowflake Marketplace. Proces pozostával z krokov čistenia, obohacovania a reorganizácie dát s cieľom vytvoriť viacdimenzionálny dátový model typu hviezda. Tento model umožňuje komplexnú analýzu, predstavuje základ pre následnú tvorbu vizualizácií a analytických reportov.

Po úspešnom vytvorení dimenzionálnych a faktových tabuliek boli údaje nahrané do finálnej dátovej štruktúry. Následne boli staging tabuľky odstránené, aby sa zabezpečila efektívna správa a optimalizované využitie úložného priestoru.

#### Príklad kódu:
```sql
drop table STG_BTC_PENDING;
drop table STG_ETH_PENDING;
drop table STG_LTC_PENDING;
drop table UNIFIED_STG;
```
---
## **4 Vizualizácia dát**
Dashboard obsahuje `5 vizualizácií`.

<p align="center">
  <img src="https://github.com/Matus-Konecny/CryptoBlockchain-ETL/blob/main/sql/dashboard.sql" alt="Dashboard">
  <br>
  <em>Obrázok 5 Dashboard Blockchain Crypto datasetu</em>
</p>

---
### **Graf 1: Počet transakcií podľa blockchainu**

```sql
SELECT 
    c.coin_name,
    COUNT(*) AS transaction_count
FROM FACT_TRANSACTIONS ft
JOIN DIM_COIN c ON ft.id_coin = c.id_coin
GROUP BY c.coin_name
ORDER BY transaction_count DESC;
```
---
### **Graf 2: Počet transakcií v každom bloku**

```sql
SELECT 
    b."blockNumber",
    COUNT(*) AS tx_in_block,
    ROUND(SUM(ft."fee"), 2) AS total_fees_in_block,
    ROUND(AVG(ft."fee"), 2) AS avg_fee_in_block
FROM FACT_TRANSACTIONS ft
JOIN DIM_BLOCK b ON ft.id_block = b.id_block
GROUP BY b."blockNumber"
ORDER BY tx_in_block DESC
LIMIT 20;
```
---
### **Graf 3: Distribúcia hodnôt transakcií**

```sql
SELECT 
    CASE 
        WHEN ft."value" = 0 THEN '0'
        WHEN ft."value" > 0 AND ft."value" <= 1000000 THEN '1 - 1M'
        WHEN ft."value" > 1000000 AND ft."value" <= 10000000 THEN '1M - 10M'
        WHEN ft."value" > 10000000 AND ft."value" <= 100000000 THEN '10M - 100M'
        ELSE '100M+'
    END AS value_range,
    COUNT(*) AS tx_count,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) AS percentage
FROM FACT_TRANSACTIONS ft
GROUP BY value_range
ORDER BY 
    CASE value_range
        WHEN '0' THEN 1
        WHEN '1 - 1M' THEN 2
        WHEN '1M - 10M' THEN 3
        WHEN '10M - 100M' THEN 4
        ELSE 5
    END;
```
---
### **Graf 4: Počet Fee vs Počet Value**

```sql
SELECT 
    c.coin_name,
    ft."value",
    ft."fee",
    ft."status",
    COUNT(*) AS tx_count
FROM FACT_TRANSACTIONS ft
JOIN DIM_COIN c ON ft.id_coin = c.id_coin
GROUP BY c.coin_name, ft."value", ft."fee", ft."status"
ORDER BY ft."value" DESC, ft."fee" DESC;
```
---
### **Graf 5: Priemerná hodnota transakcie s kontraktom**

```sql
SELECT 
    c.coin_name,
    CASE 
        WHEN ac.id_address IS NOT NULL THEN 'With Contract'
        ELSE 'No Contract'
    END AS contract_type,
    ROUND(AVG(ft."value"), 2) AS avg_value,
FROM FACT_TRANSACTIONS ft
JOIN DIM_COIN c ON ft.id_coin = c.id_coin
LEFT JOIN DIM_ADDRESS ac ON ft.id_contract_address = ac.id_address
GROUP BY c.coin_name, contract_type
ORDER BY c.coin_name, contract_type;
```
---

**Autor:** Matúš Konečný

---
