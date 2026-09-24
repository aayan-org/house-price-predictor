
<img width="1055" height="1491" alt="mlops-workflows" src="https://github.com/user-attachments/assets/6cf276f3-8256-4a63-bf76-d9777de806ae" />

Sure. The diagram represents a complete **end-to-end MLOps lifecycle** for the House Price Prediction project. The key idea is to separate **CI (validation/training)** from **CD (release/deployment)**, with MLflow providing experiment/model tracking and monitoring creating the feedback loop.

* **1. Developer triggers the pipeline**

  * The workflow starts from GitHub.
  * A **Pull Request** triggers CI to validate proposed changes before they are merged.
  * A **push/merge to `main`** runs CI again on the accepted code and can then allow CD to proceed.
  * A **version tag**, such as `v1.0.0`, represents an explicit versioned release.
  * This gives three distinct concepts: development validation, integration into `main`, and versioned production releases.

* **2. CI pipeline starts — `ci.yml`**

  * GitHub Actions starts the Continuous Integration workflow.
  * CI is responsible for answering: **“Is this code, data pipeline, model, and container safe enough to become a release candidate?”**
  * CI does **not push the production Docker image** during Pull Request validation.

* **3. Code checkout and environment setup**

  * GitHub Actions checks out the repository.
  * Python 3.11 is configured.
  * Required Python dependencies are installed from `requirements.txt`.
  * Dependency caching can be used to make subsequent pipeline executions faster.
  * The runner now contains the source code, configuration, tests, preprocessing scripts, feature engineering code, and training code.

* **4. Code-quality validation**

  * Static code checks are executed before expensive ML operations.
  * Tools such as **Ruff/Flake8** can check formatting, unused imports, common programming mistakes, and coding standards.
  * Python syntax can also be validated.
  * The objective is to fail quickly when there is a basic software problem rather than wasting time training a model.

* **5. Unit and integration testing**

  * `pytest` executes automated tests.
  * Tests can cover preprocessing functions, feature transformations, model utilities, configuration loading, API functionality, and other business logic.
  * Test coverage can optionally be generated.
  * If critical tests fail, the CI pipeline stops.
  * This introduces normal software-engineering quality controls into the ML project.

* **6. Raw data enters the ML pipeline**

  * For this project, the initial dataset is:
    `data/raw/house_data.csv`
  * CI first verifies that the expected input exists and is readable.
  * In a larger production system, this stage could instead retrieve versioned data from S3, Azure Blob Storage, GCS, a database, or a feature store.

* **7. Data preprocessing**

  * The preprocessing program loads the raw house-price dataset.
  * It performs operations such as cleaning invalid records, handling missing values, treating inconsistent values, and applying required preprocessing rules.
  * Your command is conceptually:
    `run_processing.py → raw data → cleaned data`
  * The resulting dataset becomes:
    `data/processed/cleaned_house_data.csv`
  * A reusable preprocessing object may also be produced so exactly the same transformation logic can later be applied during inference.

* **8. Feature engineering**

  * The cleaned dataset moves into the feature-engineering stage.
  * Numerical values may be scaled or normalized.
  * Categorical columns may be encoded.
  * New predictive features may be derived from existing columns.
  * The fitted transformations are captured in:
    `models/trained/preprocessor.pkl`
  * The final training dataset becomes:
    `data/processed/featured_house_data.csv`
  * Saving the preprocessor is especially important because production requests must receive the **same transformations used during training**.

* **9. Model training**

  * The engineered dataset is supplied to `train_model.py`.
  * Training configuration comes from:
    `configs/model_config.yaml`
  * The script trains the configured regression algorithm—for example Random Forest, XGBoost, or another regression model depending on the project configuration.
  * The trained model is written under:
    `models/trained/`
  * Keeping configuration outside the Python implementation makes experiments easier to reproduce.

* **10. MLflow experiment tracking**

  * During training, the application communicates with the **MLflow Tracking Server**.
  * Each training execution creates an MLflow run.
  * MLflow can record:

    * hyperparameters;
    * algorithm/configuration;
    * RMSE, MAE, MSE and R²;
    * model artifacts;
    * plots;
    * preprocessing information;
    * dataset/model metadata.
  * This provides traceability between an experiment and its results instead of relying only on console output.

* **11. MLflow storage architecture**

  * The MLflow Tracking Server acts as the interface used by the training process.
  * For simple CI experimentation, SQLite can act as the backend metadata store.
  * For a production architecture, the diagram shows a more durable design:
    `MLflow → PostgreSQL + object storage`
  * PostgreSQL can retain experiment/model metadata.
  * S3, Azure Blob Storage, or GCS can retain larger model artifacts.
  * This makes experiment history persistent even when a GitHub Actions runner disappears.

* **12. CI artifacts are generated**

  * Important outputs from CI are retained for downstream stages.
  * These can include:

    * `featured_house_data.csv`;
    * `preprocessor.pkl`;
    * trained model files;
    * pytest/test reports;
    * coverage reports;
    * MLflow run information;
    * build logs.
  * Artifacts provide traceability and prevent downstream stages from unnecessarily repeating work.

* **13. Model evaluation begins**

  * Successfully training a model does **not automatically mean it should be released**.
  * The trained model therefore passes through a model-evaluation stage.
  * Typical regression metrics include RMSE, MAE, MSE, and R².
  * Cross-validation can also be performed.
  * The candidate can be compared with an existing baseline or currently deployed model.

* **14. Model quality gate**

  * CI applies explicit acceptance criteria to the evaluation results.
  * The diagram illustrates thresholds such as:

    * RMSE below a defined limit;
    * MAE below a defined limit;
    * R² above a defined limit;
    * all automated tests passing.
  * Those values are examples—the actual thresholds should come from the model's business and statistical requirements.
  * Data-schema validation and inference tests can also be included here.

* **15. Gate makes the release decision**

  * If quality checks fail:
    `Model → FAIL → pipeline stops`
  * The failed model is not passed to CD.
  * Developers can inspect GitHub Actions logs and MLflow metrics to determine what went wrong.
  * If all checks pass:
    `Model → PASS → eligible for CD`
  * This is an important MLOps control because deployment depends on measurable quality rather than merely successful execution.

* **16. Docker validation occurs during CI**

  * CI builds the application's Docker image.
  * The trained model and inference application are packaged together according to the `Dockerfile`.
  * Basic container tests can verify that the image builds successfully, the application starts, required files exist, and an API/health endpoint responds.
  * On a Pull Request, the image is **built for validation but not pushed to Docker Hub**.
  * This prevents unapproved PR code from becoming a distributable production image.

* **17. CI completes**

  * At this point, CI has validated the complete chain:
    `Code → Tests → Data → Features → Training → Evaluation → Docker`
  * A successful CI run therefore means substantially more than “Python compiled successfully.”
  * It demonstrates that the ML application can be rebuilt from source and satisfy the configured validation requirements.

* **18. CD pipeline starts — `cd.yml`**

  * Continuous Deployment/Delivery is separated from CI.
  * CD should run only for an approved event such as successful integration into `main` or creation of a release tag.
  * Its purpose is different from CI.
  * CI asks **“Is this candidate valid?”**
  * CD asks **“How do we safely release this validated candidate?”**

* **19. CD verifies CI status**

  * Before doing anything destructive or externally visible, CD verifies that CI succeeded for the exact commit being released.
  * It should check the corresponding Git SHA rather than simply assuming that the latest successful CI execution belongs to the release.
  * If CI failed or no successful validation exists, deployment stops.
  * This creates the core release gate:
    `CI success → CD allowed`
    `CI failure → CD blocked`

* **20. CD retrieves the validated model**

  * CD obtains the model that was validated by CI.
  * In a small implementation, that may come from a CI artifact.
  * In a more mature architecture, CD retrieves a particular model version from the **MLflow Model Registry**.
  * The important principle is that CD should deploy the **same model that passed evaluation**, rather than silently retraining another model.

* **21. Model Registry can manage promotion**

  * MLflow Model Registry can retain model versions and their lifecycle metadata.
  * A candidate model can be registered after successful training/evaluation.
  * Promotion can conceptually follow:
    `Candidate → validated version → production-approved version`
  * The registry also helps answer which model version is deployed and which experiment produced it.

* **22. Production Docker image is built**

  * CD builds the deployable Docker image using the validated model.
  * The image contains the inference application, runtime dependencies, model/preprocessor, and application configuration required for serving.
  * A final smoke test can be performed before publication.

* **23. Docker image is versioned**

  * Instead of relying exclusively on `latest`, several tags can identify the exact build.
  * For example:
    `house-price-model:latest`
    `house-price-model:v1.0.0`
    `house-price-model:<git-sha>`
  * The Git SHA provides strong traceability:
    `Docker image → commit → CI execution → model → MLflow experiment`
  * A semantic version such as `v1.0.0` provides a human-friendly release identifier.

* **24. Image is pushed to the container registry**

  * CD authenticates to Docker Hub using GitHub Actions secrets.
  * The validated image is pushed to the registry.
  * Docker Hub therefore becomes the distribution point for the production container.
  * Other registries such as GHCR, AWS ECR, Azure Container Registry, or Google Artifact Registry could serve the same role.

* **25. Deployment infrastructure pulls the image**

  * The target environment retrieves the approved image from the container registry.
  * The diagram allows several deployment choices:

    * Kubernetes;
    * VM/server;
    * AWS;
    * Azure;
    * GCP.
  * The CI/CD architecture is therefore largely independent of the final hosting platform.

* **26. Deployment occurs**

  * CD updates the target environment to the new container version.
  * Kubernetes could update a Deployment manifest or Helm release.
  * A VM deployment could stop the old container and launch the new one.
  * More mature environments can use rolling, blue/green, or canary deployment strategies.
  * These approaches reduce disruption and make rollback easier.

* **27. Post-deployment checks run**

  * Deployment success should not be determined merely by whether Docker started.
  * CD runs smoke/health tests against the deployed service.
  * Checks can verify API availability, model loading, test inference, expected response schema, latency, and service health.
  * If these checks fail, the deployment can be marked unsuccessful and potentially rolled back.

* **28. Model inference service becomes available**

  * The deployed container exposes the trained model through an API, commonly implemented with FastAPI or Flask.
  * A client sends house information to the REST endpoint.
  * Conceptually:
    `Client → REST API → Preprocessor → Model → Predicted Price`
  * The API validates the incoming request before inference.

* **29. Production preprocessing is reused**

  * The saved `preprocessor.pkl` transforms incoming production data using the exact transformations learned during training.
  * This avoids one of the classic ML production bugs:
    `training transformation ≠ production transformation`
  * The transformed feature vector is then sent to the trained regression model.

* **30. Prediction is returned**

  * The model calculates the house-price prediction.
  * The API converts the result into a defined response format, usually JSON.
  * For example, the logical flow is:
    `House attributes → API → transformation → model → predicted house price → API response`

* **31. Production monitoring starts**

  * Once deployed, the MLOps lifecycle continues.
  * Application monitoring can track request rate, error rate, response time, CPU/memory consumption, and availability.
  * Prometheus and Grafana are common examples of tools that can support infrastructure/application monitoring.

* **32. Model monitoring is also required**

  * ML monitoring goes beyond normal application monitoring.
  * The system can watch input distributions, prediction distributions, feature drift, data drift, model performance when ground truth becomes available, and unusual prediction patterns.
  * This helps identify situations where the service is technically healthy but the model has become less useful.

* **33. Alerts can be generated**

  * When predefined thresholds are crossed, monitoring can generate alerts.
  * Notifications could be sent through Slack, email, incident-management tooling, or another operational system.
  * Examples include high API error rate, abnormal latency, data drift, prediction anomalies, or degraded model accuracy.

* **34. Monitoring closes the feedback loop**

  * Production information feeds back into future model development.
  * New data can be collected.
  * Drift or degradation can trigger investigation and potentially retraining.
  * A new candidate model then goes through the same CI quality controls before it can replace the existing production version.
  * This produces the continuous lifecycle:
    `Develop → Validate → Train → Evaluate → Release → Deploy → Monitor → Improve → Retrain`

* **35. End-to-end traceability**

  * Ideally, every production prediction can ultimately be traced to a deployed model version.
  * That model version can be traced to a Docker image.
  * The Docker image can be traced to a Git commit.
  * The commit can be traced to its CI execution.
  * The model can be traced to its MLflow experiment, parameters, metrics, data version, and preprocessing version.
  * Conceptually:
    `Production → Docker tag → Git SHA → CI run → MLflow run → Model/Data/Config`

* **36. Overall architecture**

  * The complete workflow can be summarized as:
    `Developer → GitHub → CI → Data Processing → Feature Engineering → Training → MLflow → Model Evaluation → Quality Gate → Docker Validation → CD → Model/Artifact Retrieval → Docker Registry → Deployment → REST API → Monitoring → Feedback → Retraining`
  * The crucial separation is that **CI creates and validates a release candidate**, while **CD releases an already validated candidate**. MLflow supplies experiment/model traceability, Docker supplies reproducible packaging, and monitoring closes the ML lifecycle.
