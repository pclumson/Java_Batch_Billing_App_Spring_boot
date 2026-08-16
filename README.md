# Java_Batch_Billing_App_Spring_boot
Building a Batch Application with Java Spring Batch
<img width="774" height="549" alt="Screenshot_20260131_020609" src="https://github.com/user-attachments/assets/c0fb4195-d8c4-4593-aee6-2bf324378f2f" />


<img width="736" height="675" alt="Screenshot_20260131_020713" src="https://github.com/user-attachments/assets/9cae4d09-9c9a-46cc-981b-46c9bb39ae30" />


<img width="766" height="560" alt="Screenshot_20260131_020757" src="https://github.com/user-attachments/assets/0ac2a865-e648-4132-b3c6-dacf493f1da5" />


# Water Billing Batch (Spring Batch, database-driven)

A Spring Boot + Spring Batch application that reads unbilled water meter
readings from a database table, prices each into a bill (every line item a
standard printed statement shows), writes the bill to a `bills` table, and
renders a matching PDF statement — all in one chunk-oriented job.

## How it works

```
meter_readings table (processed = FALSE)
        │
        ▼
   [Reader]  JdbcCursorItemReader → MeterReading
        │
        ▼
 [Validator + BillProcessor]  → tiered/flat rate calculation → Bill
        │
        ▼
 [Composite Writer], in order:
   1. BillPdfWriter        → output/bills/<account>_<date>.pdf   (sets Bill.pdfPath)
   2. billJdbcWriter        → INSERT INTO bills (...)             (persists pdfPath too)
   3. markReadingProcessedWriter → UPDATE meter_readings SET processed = TRUE
```

All three writers act on the *same* chunk of `Bill` objects in that order, so
the PDF path set by writer 1 is visible to the DB insert in writer 2, and a
reading is only flagged `processed` (writer 3) once its bill has actually
been priced, persisted, and printed. If the job crashes mid-chunk, none of
the three side effects for that chunk commit (single Spring-managed
transaction per chunk), so a re-run picks the reading back up cleanly.

## Database schema

Two tables, defined in `src/main/resources/schema.sql`:

- **`meter_readings`** — the source data: account/customer/meter info,
  previous & present reads, previous balance, net payments, and flags
  (`past_due`, `reclaimed_water_account`, `auto_pay`, `final_bill`). A
  `processed` boolean lets the reader's query (`WHERE processed = FALSE`)
  pick up only unbilled rows, so re-running the job is safe.
- **`bills`** — one row per generated bill: every itemized charge, the
  summary-of-account fields, the computed `amount_due`, the `pdf_path` of
  the rendered statement, and `source_reading_id` (FK back to the reading
  it came from).

`src/main/resources/data.sql` seeds `meter_readings` with 11 demo rows
(mirroring the earlier CSV sample set — including a bad-data row and a
missing-customer-name row to exercise the skip/validation path). **Both
scripts run automatically on startup** via `spring.sql.init.mode=always`.
That's convenient for local dev (fresh data every run) but wrong for
production — see "Moving to production" below.

Spring Batch's own job/step execution history (the `BATCH_*` tables) lives
in the *same* datasource, via `spring.batch.jdbc.initialize-schema=always`.

## Job/Step

- **Reader** (`BatchConfig#meterReadingReader`): `JdbcCursorItemReader`
  streaming `SELECT ... FROM meter_readings WHERE processed = FALSE ORDER BY id`,
  mapped to `MeterReading` by `MeterReadingRowMapper`.
- **Validator + Processor** (`BillProcessor`): same tiered/flat-rate pricing
  logic as before — Customer Service Charge, Purchase Water Pass-Thru, Water
  Base/Usage Charge (tiered), Sewer Base/Usage Charge (capped), optional
  Reclaimed Water Charge, and a Late Payment Charge when the prior balance is
  past due. Rows with `present_read < previous_read` are filtered (returns
  `null`); rows with no account number or a negative read are *skipped* via
  `ValidationException` (up to `skipLimit(50)`), not fatal to the run.
- **`BillPdfWriter`**: renders the same one-page PDF statement layout as
  before (header identity bar, meter table, charges/summary columns,
  consumption-history chart, payment stub) — now sourced from DB rows
  instead of CSV.
- **`billJdbcWriter`** / **`markReadingProcessedWriter`**:
  `JdbcBatchItemWriter<Bill>` beans, using named-parameter SQL bound
  straight from `Bill`'s bean properties (`beanMapped()`).
- **`JobCompletionListener`**: logs a run summary (read/written/skipped).

## Rate schedule & branding

Unchanged from before — both are `@ConfigurationProperties` beans
(`RateSchedule`, `UtilityBranding`) bound from `billing.rates.*` and
`billing.utility.*` in `application.properties`. See that file for the full
list of tunable rates (all placeholder values — replace with your utility's
actual approved tariff).

## Balance formula

```
Total Account Charges = Total Service Address Charges (recurring water/sewer charges)
Bill Adjustments      = Total Miscellaneous Charges (e.g. late fee)
Amount Due            = Previous Balance + Net Payments (payments are negative)
                         + Total Account Charges + Bill Adjustments
```

## Project layout

```
water-billing-batch/
├── pom.xml
└── src/
    ├── main/
    │   ├── java/com/example/waterbilling/
    │   │   ├── WaterBillingBatchApplication.java   # entry point, launches the job
    │   │   ├── config/BatchConfig.java              # reader/processor/writer/job/step beans
    │   │   ├── config/RateSchedule.java              # @ConfigurationProperties rate table
    │   │   ├── config/UtilityBranding.java           # @ConfigurationProperties header/footer text
    │   │   ├── model/MeterReading.java               # input record (mirrors meter_readings row)
    │   │   ├── model/Bill.java                       # fully priced output record (mirrors bills row)
    │   │   ├── processor/BillProcessor.java          # charge calculation
    │   │   ├── reader/MeterReadingRowMapper.java      # ResultSet -> MeterReading
    │   │   ├── writer/BillPdfWriter.java              # renders the printable PDF statement
    │   │   └── listener/JobCompletionListener.java   # run summary logging
    │   └── resources/
    │       ├── application.properties
    │       ├── schema.sql                            # meter_readings + bills DDL
    │       └── data.sql                               # demo seed data
    └── test/
        └── java/com/example/waterbilling/BillProcessorTest.java
```

## Running it

Requires JDK 17+ and Maven.

```bash
mvn clean package
java -jar target/water-billing-batch-1.0.0.jar
```

Output:
- `bills` table (H2 file DB at `./data/waterbilling.mv.db`) — query it with any
  H2-compatible client, or via the H2 console if you enable it.
- `./output/bills/*.pdf` — one printable statement per account.
- `meter_readings.processed` flipped to `TRUE` for every row that got billed.

Run the tests only:

```bash
mvn test
```

Inspect the results directly with the H2 shell:

```bash
java -cp ~/.m2/repository/com/h2database/h2/2.2.224/h2-2.2.224.jar org.h2.tools.Shell \
  -url "jdbc:h2:file:./data/waterbilling" -user sa
# then: SELECT account_number, amount_due, pdf_path FROM bills;
```

## Moving to production

- **Swap H2 for Postgres/MySQL**: change `spring.datasource.*` in
  `application.properties` and add the matching JDBC driver dependency to
  `pom.xml` — nothing else in the code changes, since everything is plain
  JDBC (`JdbcCursorItemReader` / `JdbcBatchItemWriter`).
- **Stop resetting data on startup**: delete `data.sql` (or set
  `spring.sql.init.mode=never`) and remove the `DROP TABLE` statements from
  `schema.sql` once `meter_readings` is populated by a real upstream
  feed/integration rather than the demo seed. Better yet, replace both
  scripts with versioned migrations (Flyway or Liquibase) so schema changes
  are tracked over time instead of applied ad hoc.
- **Loading real readings**: point whatever loads your meter-data-management
  (MDM) export at `meter_readings` (bulk INSERT, a scheduled ETL job, or a
  CDC pipeline) — the batch job only cares that unbilled rows have
  `processed = FALSE`.
- **Partition for scale**: for very large customer bases, add a partitioned
  step (e.g. by account-number range) so multiple threads/instances process
  different partitions of `meter_readings` concurrently.
- **Retry vs. skip**: the current `skip()` policy handles bad *data*.
  Consider adding a separate `retry()` policy on the writers for transient
  *infrastructure* failures (e.g. a brief DB connection blip), which is a
  different failure mode.
- PDF text rendering uses WinAnsi-safe Standard 14 fonts; non-Latin-1
  characters in customer names/addresses are replaced with `?`. Embed a
  Unicode TrueType font in `BillPdfWriter` if you need full Unicode support.
