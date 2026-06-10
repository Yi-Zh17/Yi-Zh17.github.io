+++
date = '2026-04-03T00:00:00Z'
title = 'FraudShield: Running a Neural Network Inside a Spring Boot Service'
ShowToc = true
+++

# FraudShield: Running a Neural Network Inside a Spring Boot Service

I built FraudShield to practice end-to-end ML deployment — specifically the part that tutorials skip: taking a trained model and making it actually work inside a real backend service. The core challenge was figuring out how to ship a PyTorch model as something a Java application could run at inference time, without spinning up a Python sidecar or calling a remote API.

The result is a Spring Boot web application that accepts credit card transaction details, assembles a 75-dimensional feature vector in Java, runs it through a neural network via ONNX Runtime, and persists every prediction to Azure SQL Server for auditing. The whole thing is containerised and deployed to Azure Web App for Containers via a GitHub Actions CI/CD pipeline.

The dataset is the [DataCamp Credit Card Fraud dataset](https://www.datacamp.com/datalab/datasets/dataset-python-credit-card-fraud), which contains ~1.3 million synthetic US credit card transactions labelled as fraudulent or legitimate.

---

## Tech Stack

- **Java 17** / **Spring Boot 3.5.12** — web framework, MVC, JPA
- **ONNX Runtime 1.22** (Java bindings) — in-process model inference
- **PyTorch** — model training (offline)
- **Thymeleaf** — server-side HTML templating
- **Tailwind CSS v4** — utility-first styling
- **Azure SQL Server** + **Azure Identity** — audit log persistence with passwordless Entra ID auth
- **Docker** (multi-stage build with Spring Boot layer extraction)
- **Kubernetes** — deployment and service manifests
- **GitHub Actions** — CI/CD to Azure Web App for Containers

---

# Architecture Overview

The central design decision was keeping inference in-process. Rather than calling a Python model server over HTTP, the ONNX model file is bundled directly into the Spring Boot jar and loaded once at startup via ONNX Runtime's Java bindings. Every prediction call is a local method invocation — no network hop, no serialisation overhead, no separate process to manage.

```
Browser (Thymeleaf + Tailwind CSS)
        │  POST /fraud/check
        ▼
  FraudController
        │  FraudCheckRequest (Java record)
        ▼
  FraudCheckService ──────────────────────► FraudAuditRepository
        │  process(request)                        │
        ▼                                    Azure SQL Server
  FraudCheckUtil                             (fraud_audit_logs)
  (75-dim feature vector)
        │
        ▼
  ModelPredictionService
  (ONNX Runtime session)
        │  float[] → OnnxTensor → session.run()
        ▼
  raw logit → sigmoid → probability → FraudCheckResponse
```

**One request, end to end.** The user submits a form with transaction details. `FraudController` deserialises the form into a `FraudCheckRequest` record and hands it to `FraudCheckService`. The service builds the feature vector via `FraudCheckUtil`, validates it, runs it through `ModelPredictionService`, applies sigmoid once to convert the raw logit to a probability, writes an audit record to Azure SQL, and returns a `FraudCheckResponse` containing the real probability. The controller reads that probability directly for display — no further transformation needed.

---

# The Neural Network

The model is a simple fully-connected network trained in PyTorch on 1,296,675 transactions (80/20 train/val split). Architecture:

```
Input (75) → Linear(75, 75) → ReLU
           → Linear(75, 60) → ReLU
           → Linear(60, 45) → ReLU
           → Linear(45, 30) → ReLU
           → Linear(30, 15) → ReLU
           → Linear(15, 1)   [raw logit]
```

Training used `BCEWithLogitsLoss` (which applies sigmoid internally) and Adam at `lr=1e-3`, batch size 32, for 5 epochs. Validation accuracy across epochs:

| Epoch | Val Accuracy |
|-------|-------------|
| 1 | 99.79% |
| 2 | 99.81% |
| 3 | 99.81% |
| 4 | 99.79% |
| 5 | 99.82% |

The model converges quickly and stays stable — there's no meaningful degradation across 5 epochs, and the variance between runs is under 0.03 percentage points.

Worth keeping in mind: the dataset is heavily imbalanced, with around 0.6% of transactions labelled fraudulent. A naïve classifier that always predicts "not fraud" would score ~99.4% accuracy, so the headline numbers have to be read in that context. Precision and recall on the fraud class would tell a more complete story, and computing them is the natural next step for a more rigorous evaluation.

After training, the model is exported to ONNX using `torch.onnx.export` with `dynamo=True`:

```python
example = torch.randn(1, 75, device='cuda')
torch.onnx.export(
    model, example, "model.onnx",
    export_params=True,
    do_constant_folding=True,
    input_names=['input'],
    output_names=['output'],
    dynamo=True
)
```

The export produces two files: `model.onnx` (the graph, ~20 KB) and `model.onnx.data` (the weights, ~58 KB). Both are committed into `src/main/resources/static/` and bundled into the jar.

---

# Feature Engineering

The raw form inputs are 10 fields. The model expects a 75-dimensional `float32` vector. All the encoding logic that was done in Python at training time has to be replicated precisely in Java — if the encodings differ, the model produces garbage output regardless of how good the training was.

Here's how each field maps to the vector:

| Positions | Field | Encoding |
|-----------|-------|----------|
| 0–2 | `amount`, `city_pop`, `age` | Z-score normalisation (hardcoded μ, σ from training data) |
| 3–16 | `category` (14 values) | One-hot |
| 17–67 | `state` (51 values) | One-hot |
| 68 | `merchant` | Frequency encoding |
| 69 | `gender` | Binary (M→0, F→1) |
| 70 | `job` | Frequency encoding |
| 71–72 | `transaction_hour` | Cyclic: sin/cos over 24 |
| 73–74 | `transaction_dayOfWeek` | Cyclic: sin/cos over 7 |

All of this lives in `FraudCheckUtil`. The Z-score normalisation for amount looks like this:

```java
public double normalization(String name, double num) {
    if (name.equals("amt")) {
        final double amt_mean = 70.35103545607033;
        final double amt_std  = 160.3159767533844;
        return (num - amt_mean) / amt_std;
    } else {
        return -1; // signals invalid call to checkIntegrity()
    }
}
```

The mean and standard deviation are the values computed over the full training set and hardcoded here. This is a deployment constraint: the Java side has no access to the original scaler object, so the statistics have to be copied manually. Any dataset update would require recomputing these values and redeploying.

**Cyclic encoding** is worth explaining briefly. Hour of day and day of week are cyclical — hour 23 and hour 0 are adjacent, not distant. Encoding them as plain integers would mislead the model. Instead, each is encoded as `(sin(2π·n/period), cos(2π·n/period))`, which places adjacent hours close together in a 2D circular space:

```java
public double[] hourEncoding(int num) {
    double[] result = new double[2];
    if (num < 0 || num > 23) return result; // zeros → caught by checkIntegrity
    result[0] = Math.sin(2 * Math.PI * num / 24);
    result[1] = Math.cos(2 * Math.PI * num / 24);
    return result;
}
```

**Frequency encoding** for merchant and job replaces each string value with its relative frequency in the training dataset. A merchant that appeared in 0.1% of training transactions gets encoded as `0.001`. These mappings are pre-computed in Python and shipped as two JSON files (`job_frequencies.json` — 494 entries, `merchant_frequencies.json` — 693 entries). Unknown values default to `0.0` rather than throwing an error, which is a reasonable choice: an unseen merchant is treated as rare.

---

# FraudCheckService: Orchestration and Assembly

`FraudCheckService` is the one place that ties everything together. It owns the model session, calls `FraudCheckUtil` to build the feature vector, validates it, runs inference, and persists the audit record — all in a single `@Transactional` method.

## Model Loading at Startup

ONNX Runtime requires a file path to load a session, not a stream. The model files live inside the jar as classpath resources, which means they need to be copied to a temp directory at startup before a session can be created:

```java
@Autowired
public FraudCheckService(FraudAuditRepository auditRepository) throws Exception {
    this.auditRepository = auditRepository;

    ClassPathResource modelRes = new ClassPathResource("static/model.onnx");
    ClassPathResource dataRes  = new ClassPathResource("static/model.onnx.data");

    Path tempDir   = Files.createTempDirectory("fraud_model");
    Path modelPath = tempDir.resolve("model.onnx");
    Path dataPath  = tempDir.resolve("model.onnx.data");

    Files.copy(modelRes.getInputStream(), modelPath, StandardCopyOption.REPLACE_EXISTING);
    Files.copy(dataRes.getInputStream(),  dataPath,  StandardCopyOption.REPLACE_EXISTING);

    this.model = new ModelPredictionService(modelPath.toString());
}
```

Both files must land in the same directory with the same names — ONNX Runtime resolves `model.onnx.data` by convention relative to the `.onnx` file. The session is created once and reused for the lifetime of the application.

## Feature Assembly

`FraudCheckService.process()` calls each encoding method on `FraudCheckUtil` and assembles the results into the fixed-layout 75-element array using `System.arraycopy` for the sub-arrays:

```java
double[] result = new double[75];

result[0] = amt;       // normalised amount
result[1] = city_pop;  // normalised city population
result[2] = age;       // normalised age
System.arraycopy(category, 0, result, 3,  14); // one-hot category
System.arraycopy(state,    0, result, 17, 51); // one-hot state
result[68] = merchant; // frequency-encoded
result[69] = gender;   // binary
result[70] = job;      // frequency-encoded
System.arraycopy(hour, 0, result, 71, 2);      // cyclic hour
System.arraycopy(day,  0, result, 73, 2);      // cyclic day of week

return result;
```

The layout has to match exactly what the model was trained with. The Python training notebook assembles the DataFrame columns in the same order: normalised numerics first, then one-hot categoricals, then frequency-encoded fields, then cyclic temporals.

## Integrity Validation

Before sending the vector to the model, `checkIntegrity()` validates it:

```java
void checkIntegrity(double[] features) {
    // Normalisations return -1 on invalid input
    if (features[0] == -1 || features[1] == -1 || features[2] == -1)
        throw new RuntimeException("Invalid normalisation input.");

    // One-hot arrays must sum to exactly 1.0
    double sum = 0;
    for (int i = 3; i < 17; i++) sum += features[i];
    if (sum != 1.0) throw new RuntimeException("Invalid category encoding.");

    sum = 0;
    for (int i = 17; i < 68; i++) sum += features[i];
    if (sum != 1.0) throw new RuntimeException("Invalid state encoding.");

    // Gender must be 0.0 or 1.0
    if (features[69] == -1) throw new RuntimeException("Invalid gender input.");

    // Cyclic pairs of (0, 0) signal out-of-range input
    if (features[71] == 0.0 && features[72] == 0.0)
        throw new RuntimeException("Invalid hour input.");
    if (features[73] == 0.0 && features[74] == 0.0)
        throw new RuntimeException("Invalid day input.");
}
```

This is a sanity check rather than a security boundary — the inputs come from a controlled form with predefined options, so malformed data is unlikely in practice. But catching encoding bugs early is useful during development, and the `-1` sentinel from `normalization()` makes invalid calls visible without needing exceptions in the hot path.

---

# ModelPredictionService: ONNX Runtime

`ModelPredictionService` is a thin wrapper around the ONNX Runtime Java API. It holds one `OrtSession` for the lifetime of the application and exposes a single `predict(double[])` method:

```java
public double predict(double[] features) throws OrtException {
    // ONNX Runtime expects float32, Java gives us double
    float[] inputData = new float[features.length];
    for (int i = 0; i < features.length; i++) {
        inputData[i] = (float) features[i];
    }

    long[] shape = {1, features.length};  // batch size 1
    OnnxTensor inputTensor = OnnxTensor.createTensor(
        env, FloatBuffer.wrap(inputData), shape
    );

    OrtSession.Result result = session.run(
        Collections.singletonMap("input", inputTensor)
    );

    float[] output = (float[]) result.get(0).getValue();
    return output[0]; // raw logit — sigmoid is applied in FraudCheckService
}
```

A few things worth noting. The method takes `double[]` but the model expects `float32`. The explicit cast (`(float) features[i]`) causes a small precision loss, which is acceptable for a sigmoid classifier — the output is a probability estimate, not a scientific measurement.

The input name `"input"` must match the `input_names` argument from the `torch.onnx.export` call. If they differ, the session throws at runtime.

The class implements `AutoCloseable` so the session and environment can be shut down cleanly, though in this application the service lives for the full process lifetime and is never explicitly closed.

---

# Persistence: Audit Logging

Every prediction is written to Azure SQL Server via JPA. The `FraudAudit` entity captures a subset of the input alongside the model's output:

```java
@Entity
@Table(name = "fraud_audit_logs")
public class FraudAudit {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private double amount;
    private String state;
    private String merchant;
    private double predictedProbability;
    private boolean isFraud;

    @Column(updatable = false)
    private LocalDateTime timestamp;

    @PrePersist
    protected void onCreate() {
        timestamp = LocalDateTime.now();
    }
}
```

The `@PrePersist` hook ensures the timestamp is set server-side at insert time and is never modifiable afterwards (`updatable = false`). Hibernate with `ddl-auto=update` creates the table automatically on first boot if it doesn't exist.

Authentication to Azure SQL uses passwordless Entra ID via the JDBC connection string:

```
jdbc:sqlserver://${DB_SERVER_NAME}.database.windows.net:1433;
  database=${DB_NAME};
  encrypt=true;
  Authentication=ActiveDirectoryDefault;
```

`ActiveDirectoryDefault` picks up the ambient Azure identity — the managed identity of the web app in production, or your local `az login` credentials in development. No passwords in config or environment variables.

---

# Testing

`FraudCheckUtilTest` covers each encoding function with concrete values sampled from the training data and a few edge cases. For example, the normalisation tests verify both a typical amount and an out-of-range call:

```java
@Test
void testNormalization() {
    assertEquals(-0.4078, util.normalization("amt", 4.97),   1e-4);
    assertEquals( 0.9341, util.normalization("amt", 220.11), 1e-4);
    // Invalid field name → sentinel
    assertEquals(-1, util.normalization("some other string", 122), 1e-8);
}
```

The tolerance of `1e-4` is appropriate here — the hardcoded float statistics mean there's inherent rounding in the expected values. The cyclic encoding tests verify both correct output and the out-of-range sentinel:

```java
@Test
void testHourDayEncoding() {
    assertArrayEquals(new double[]{0.258819, 0.965925}, util.hourEncoding(1), 1e-6);
    // Out-of-range → zero array → caught by checkIntegrity
    assertArrayEquals(new double[2], util.hourEncoding(45));
}
```

There are no integration tests for `FraudCheckService` itself — only the `contextLoads()` smoke test that Spring Boot generates by default. Properly testing the service would require either a live Azure SQL connection or a mock of `FraudAuditRepository`.

A `FraudControllerTest` regression test verifies that a probability of `0.8` is displayed as `80%` — which confirms that sigmoid is applied exactly once (in `FraudCheckService`) and the controller reads the already-converted value directly. Without this test, a future refactor could silently reintroduce the double-application bug.

---

# Deployment

## Docker

The Dockerfile uses a four-stage build to keep the final image lean:

1. **deps** — downloads Maven dependencies into a cache layer
2. **package** — compiles and produces the fat jar
3. **extract** — uses Spring Boot's layer tools to split the jar into `dependencies`, `spring-boot-loader`, `snapshot-dependencies`, and `application`
4. **final** — copies only the extracted layers into a minimal JRE image (`eclipse-temurin:17-jre-jammy`), running as a non-root `appuser`

The layer extraction step matters for iterative deploys: the `dependencies` layer (all the Maven jars) barely changes between pushes, so Docker's cache means only the `application` layer — your compiled classes and resources — gets pushed to the registry on most rebuilds.

## GitHub Actions → Azure

On push to `main`, the pipeline builds the Docker image, pushes it to GitHub Container Registry (`ghcr.io`), and deploys to Azure Web App for Containers using `azure/webapps-deploy`. The two required environment variables (`DB_SERVER_NAME`, `DB_NAME`) are set in the Azure Web App configuration rather than in the image.

## Kubernetes

Deployment and service manifests are included for anyone running this on a cluster. The deployment sets reasonable resource constraints:

```yaml
resources:
  requests:
    memory: "512Mi"
    cpu: "250m"
  limits:
    memory: "1Gi"
    cpu: "500m"
```

The service is a `LoadBalancer` on port 80, forwarding to the container's 8080.

---

# Limitations & future work

**Accuracy metric needs context.** As noted above, accuracy alone isn't the right lens for an imbalanced dataset — precision and recall on the fraud class are the numbers that actually matter. Adding these metrics during training is the most valuable single improvement to the evaluation setup.

**Hardcoded normalisation statistics.** The mean and standard deviation for amount, city population, and age are copied from the training notebook into `FraudCheckUtil` as constants. This works fine while the model and the Java service are developed together, but it means any retrain on updated data requires a manual update to the Java code. A cleaner approach would serialise the scaler parameters alongside the ONNX model and load them at startup, keeping the two artefacts in sync automatically.

**Numeric input validation is minimal.** Categorical fields are covered by predefined dropdowns, so invalid category or state values can't reach the model. Numeric fields (amount, city_pop, age) are validated by the browser's `required` attribute and `checkIntegrity()`'s sentinel check, but there's no server-side range guard. Clamping these to plausible ranges would add a small amount of robustness for free.

**Single model session.** `OrtSession` is shared across all requests. ONNX Runtime's Java bindings handle concurrent CPU inference safely, so this is correct for the current workload. Under sustained high concurrency a session pool would be the natural next step, but it's not needed at this scale.

**Audit log stores a subset of inputs.** `FraudAudit` records amount, state, merchant, the predicted probability, and the fraud flag. This is enough to track outcomes and spot anomalies. Storing the full input or feature vector would make individual predictions fully reproducible from the log, which would be useful if the model is ever retrained and old predictions need to be re-evaluated.
