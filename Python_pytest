# 📘 API Testing Documentation

## Comprehensive Guide for Testing REST API Endpoints using Python, Pytest, Allure & Excel Reporting

**Author:** Deepak-Kumar
**Date:** 2025-02-25
**Version:** 1.0

---

## 📁 Project Folder Structure

```
api-test-suite/
│
├── config/
│   ├── __init__.py
│   ├── config.py                  # Base URLs, credentials, env configs
│   └── endpoints.py               # API endpoint definitions
│
├── test_data/
│   ├── get_test_data.json         # Test data for GET requests
│   ├── post_test_data.json        # Test data for POST requests
│   └── put_test_data.json         # Test data for PUT requests
│
├── tests/
│   ├── __init__.py
│   ├── conftest.py                # Shared fixtures & hooks
│   ├── test_get_endpoints.py      # GET endpoint test cases
│   ├── test_post_endpoints.py     # POST endpoint test cases
│   └── test_put_endpoints.py      # PUT endpoint test cases
│
├── utils/
│   ├── __init__.py
│   ├── api_client.py              # Reusable HTTP client wrapper
│   ├── assertions.py              # Custom assertion helpers
│   ├── excel_reporter.py          # Excel report generation
│   └── logger.py                  # Logging configuration
│
├── reports/
│   ├── allure-results/            # Raw Allure results (JSON)
│   ├── allure-report/             # Generated Allure HTML report
│   └── excel/
│       └── test_results.xlsx      # Excel test result summary
│
├── logs/
│   └── test_run.log               # Detailed log file
│
├── pytest.ini                     # Pytest configuration
├── conftest.py                    # Root-level conftest (Excel hook)
├── requirements.txt               # Python dependencies
└── README.md                      # Project overview
```

---

## 1️⃣ Prerequisites & Installation

### Install Dependencies

```bash
pip install -r requirements.txt
```

### `requirements.txt`

```text
pytest==7.4.4
requests==2.31.0
allure-pytest==2.13.2
openpyxl==3.1.2
pytest-html==4.1.1
python-dotenv==1.0.0
jsonschema==4.21.1
```

### Install Allure Command-Line Tool

```bash
# macOS
brew install allure

# Windows (via Scoop)
scoop install allure

# Linux (via npm)
npm install -g allure-commandline
```

---

## 2️⃣ Configuration Files

### `config/config.py`

```python
import os

class Config:
    """Central configuration for the API test suite."""

    BASE_URL = os.getenv("API_BASE_URL", "https://jsonplaceholder.typicode.com")
    TIMEOUT = 30
    HEADERS = {
        "Content-Type": "application/json",
        "Accept": "application/json",
    }

    # Environment-specific overrides
    ENVIRONMENTS = {
        "dev": "https://dev-api.example.com",
        "staging": "https://staging-api.example.com",
        "production": "https://api.example.com",
    }

    @classmethod
    def get_base_url(cls, env="dev"):
        return cls.ENVIRONMENTS.get(env, cls.BASE_URL)
```

### `config/endpoints.py`

```python
class Endpoints:
    """API endpoint path definitions."""

    # User endpoints
    USERS = "/users"
    USER_BY_ID = "/users/{user_id}"

    # Post endpoints
    POSTS = "/posts"
    POST_BY_ID = "/posts/{post_id}"

    # Comment endpoints
    COMMENTS = "/comments"
    COMMENT_BY_ID = "/comments/{comment_id}"
```

---

## 3️⃣ Utility Modules

### `utils/api_client.py`

```python
import requests
import allure
from config.config import Config
from utils.logger import get_logger

logger = get_logger(__name__)


class APIClient:
    """Reusable HTTP client wrapper with Allure logging."""

    def __init__(self, base_url=None):
        self.base_url = base_url or Config.BASE_URL
        self.session = requests.Session()
        self.session.headers.update(Config.HEADERS)

    @allure.step("GET request to: {endpoint}")
    def get(self, endpoint, params=None, headers=None):
        url = f"{self.base_url}{endpoint}"
        logger.info(f"GET {url} | Params: {params}")
        response = self.session.get(url, params=params, headers=headers, timeout=Config.TIMEOUT)
        self._log_response(response)
        return response

    @allure.step("POST request to: {endpoint}")
    def post(self, endpoint, payload=None, headers=None):
        url = f"{self.base_url}{endpoint}"
        logger.info(f"POST {url} | Payload: {payload}")
        response = self.session.post(url, json=payload, headers=headers, timeout=Config.TIMEOUT)
        self._log_response(response)
        return response

    @allure.step("PUT request to: {endpoint}")
    def put(self, endpoint, payload=None, headers=None):
        url = f"{self.base_url}{endpoint}"
        logger.info(f"PUT {url} | Payload: {payload}")
        response = self.session.put(url, json=payload, headers=headers, timeout=Config.TIMEOUT)
        self._log_response(response)
        return response

    def _log_response(self, response):
        logger.info(f"Response Status: {response.status_code}")
        logger.debug(f"Response Body: {response.text[:500]}")
        allure.attach(
            body=str(response.status_code),
            name="Status Code",
            attachment_type=allure.attachment_type.TEXT,
        )
        allure.attach(
            body=response.text,
            name="Response Body",
            attachment_type=allure.attachment_type.JSON,
        )
```

### `utils/assertions.py`

```python
import allure


class Assertions:
    """Custom assertion helpers with Allure reporting."""

    @staticmethod
    @allure.step("Assert status code is {expected_code}")
    def assert_status_code(response, expected_code):
        actual = response.status_code
        assert actual == expected_code, (
            f"Expected status {expected_code}, got {actual}. Body: {response.text}"
        )

    @staticmethod
    @allure.step("Assert response contains key: {key}")
    def assert_response_has_key(response, key):
        json_data = response.json()
        assert key in json_data, f"Key '{key}' not found in response: {json_data}"

    @staticmethod
    @allure.step("Assert response key '{key}' equals '{expected_value}'")
    def assert_response_value(response, key, expected_value):
        json_data = response.json()
        actual = json_data.get(key)
        assert actual == expected_value, (
            f"Expected '{key}' = '{expected_value}', got '{actual}'"
        )

    @staticmethod
    @allure.step("Assert response is a non-empty list")
    def assert_non_empty_list(response):
        json_data = response.json()
        assert isinstance(json_data, list), f"Expected list, got {type(json_data)}"
        assert len(json_data) > 0, "Response list is empty"

    @staticmethod
    @allure.step("Assert response time is under {max_ms}ms")
    def assert_response_time(response, max_ms):
        elapsed_ms = response.elapsed.total_seconds() * 1000
        assert elapsed_ms < max_ms, (
            f"Response took {elapsed_ms:.2f}ms, exceeding {max_ms}ms limit"
        )
```

### `utils/logger.py`

```python
import logging
import os
from datetime import datetime


def get_logger(name):
    """Configure and return a logger with file and console handlers."""
    logger = logging.getLogger(name)
    logger.setLevel(logging.DEBUG)

    # Ensure logs directory exists
    os.makedirs("logs", exist_ok=True)

    # File handler
    log_filename = f"logs/test_run_{datetime.now().strftime('%Y%m%d_%H%M%S')}.log"
    file_handler = logging.FileHandler(log_filename)
    file_handler.setLevel(logging.DEBUG)

    # Console handler
    console_handler = logging.StreamHandler()
    console_handler.setLevel(logging.INFO)

    # Formatter
    formatter = logging.Formatter(
        "%(asctime)s | %(name)s | %(levelname)s | %(message)s"
    )
    file_handler.setFormatter(formatter)
    console_handler.setFormatter(formatter)

    logger.addHandler(file_handler)
    logger.addHandler(console_handler)

    return logger
```

### `utils/excel_reporter.py`

```python
import os
from datetime import datetime
from openpyxl import Workbook
from openpyxl.styles import Font, PatternFill, Alignment, Border, Side


class ExcelReporter:
    """Generates Excel reports from test results."""

    def __init__(self, output_dir="reports/excel"):
        os.makedirs(output_dir, exist_ok=True)
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        self.filepath = os.path.join(output_dir, f"test_results_{timestamp}.xlsx")
        self.results = []

    def add_result(self, test_name, method, endpoint, status_code,
                   expected_status, result, duration_ms, error_message=""):
        self.results.append({
            "Test Name": test_name,
            "HTTP Method": method,
            "Endpoint": endpoint,
            "Status Code": status_code,
            "Expected Status": expected_status,
            "Result": result,
            "Duration (ms)": round(duration_ms, 2),
            "Error Message": error_message,
            "Timestamp": datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
        })

    def generate_report(self):
        wb = Workbook()

        # --- Summary Sheet ---
        ws_summary = wb.active
        ws_summary.title = "Summary"
        self._build_summary_sheet(ws_summary)

        # --- Detailed Results Sheet ---
        ws_details = wb.create_sheet("Detailed Results")
        self._build_details_sheet(ws_details)

        # --- Failures Sheet ---
        ws_failures = wb.create_sheet("Failures")
        self._build_failures_sheet(ws_failures)

        wb.save(self.filepath)
        print(f"\n📊 Excel report generated: {self.filepath}")
        return self.filepath

    def _build_summary_sheet(self, ws):
        total = len(self.results)
        passed = sum(1 for r in self.results if r["Result"] == "PASS")
        failed = total - passed
        pass_rate = (passed / total * 100) if total > 0 else 0

        header_font = Font(bold=True, size=14, color="FFFFFF")
        header_fill = PatternFill(start_color="2F5496", end_color="2F5496", fill_type="solid")

        ws.merge_cells("A1:D1")
        ws["A1"] = "API Test Execution Summary"
        ws["A1"].font = header_font
        ws["A1"].fill = header_fill
        ws["A1"].alignment = Alignment(horizontal="center")

        summary_data = [
            ("Total Tests", total),
            ("Passed", passed),
            ("Failed", failed),
            ("Pass Rate", f"{pass_rate:.1f}%"),
            ("Execution Date", datetime.now().strftime("%Y-%m-%d %H:%M:%S")),
        ]

        green_fill = PatternFill(start_color="C6EFCE", end_color="C6EFCE", fill_type="solid")
        red_fill = PatternFill(start_color="FFC7CE", end_color="FFC7CE", fill_type="solid")

        for idx, (label, value) in enumerate(summary_data, start=3):
            ws[f"A{idx}"] = label
            ws[f"A{idx}"].font = Font(bold=True)
            ws[f"B{idx}"] = value
            if label == "Passed":
                ws[f"B{idx}"].fill = green_fill
            elif label == "Failed":
                ws[f"B{idx}"].fill = red_fill

        ws.column_dimensions["A"].width = 20
        ws.column_dimensions["B"].width = 25

    def _build_details_sheet(self, ws):
        headers = list(self.results[0].keys()) if self.results else []
        header_fill = PatternFill(start_color="4472C4", end_color="4472C4", fill_type="solid")
        header_font = Font(bold=True, color="FFFFFF")
        thin_border = Border(
            left=Side(style="thin"), right=Side(style="thin"),
            top=Side(style="thin"), bottom=Side(style="thin"),
        )

        for col_idx, header in enumerate(headers, start=1):
            cell = ws.cell(row=1, column=col_idx, value=header)
            cell.font = header_font
            cell.fill = header_fill
            cell.border = thin_border
            cell.alignment = Alignment(horizontal="center")

        green_fill = PatternFill(start_color="C6EFCE", end_color="C6EFCE", fill_type="solid")
        red_fill = PatternFill(start_color="FFC7CE", end_color="FFC7CE", fill_type="solid")

        for row_idx, result in enumerate(self.results, start=2):
            for col_idx, key in enumerate(headers, start=1):
                cell = ws.cell(row=row_idx, column=col_idx, value=result[key])
                cell.border = thin_border
                if key == "Result":
                    cell.fill = green_fill if result[key] == "PASS" else red_fill

        for col_idx, header in enumerate(headers, start=1):
            ws.column_dimensions[chr(64 + col_idx)].width = max(len(header) + 5, 15)

    def _build_failures_sheet(self, ws):
        failures = [r for r in self.results if r["Result"] == "FAIL"]
        header_fill = PatternFill(start_color="C00000", end_color="C00000", fill_type="solid")
        header_font = Font(bold=True, color="FFFFFF")

        fail_headers = ["Test Name", "HTTP Method", "Endpoint", "Expected Status",
                        "Status Code", "Error Message", "Timestamp"]

        for col_idx, header in enumerate(fail_headers, start=1):
            cell = ws.cell(row=1, column=col_idx, value=header)
            cell.font = header_font
            cell.fill = header_fill

        for row_idx, failure in enumerate(failures, start=2):
            for col_idx, key in enumerate(fail_headers, start=1):
                ws.cell(row=row_idx, column=col_idx, value=failure.get(key, ""))

        if not failures:
            ws.merge_cells("A2:G2")
            ws["A2"] = "✅ No failures detected!"
            ws["A2"].font = Font(size=14, color="00B050", bold=True)
            ws["A2"].alignment = Alignment(horizontal="center")
```

---

## 4️⃣ Test Data (JSON)

### `test_data/get_test_data.json`

```json
{
    "test_get_all_users": {
        "endpoint": "/users",
        "expected_status": 200,
        "description": "Fetch all users"
    },
    "test_get_user_by_id": {
        "endpoint": "/users/1",
        "expected_status": 200,
        "expected_name": "Leanne Graham",
        "description": "Fetch user by valid ID"
    },
    "test_get_invalid_user": {
        "endpoint": "/users/99999",
        "expected_status": 404,
        "description": "Fetch user with non-existent ID"
    }
}
```

### `test_data/post_test_data.json`

```json
{
    "test_create_post_valid": {
        "endpoint": "/posts",
        "payload": {
            "title": "Test Post Title",
            "body": "This is a test post body created by pytest.",
            "userId": 1
        },
        "expected_status": 201,
        "description": "Create a new post with valid data"
    },
    "test_create_post_empty_body": {
        "endpoint": "/posts",
        "payload": {
            "title": "",
            "body": "",
            "userId": 1
        },
        "expected_status": 201,
        "description": "Create a post with empty title and body"
    }
}
```

### `test_data/put_test_data.json`

```json
{
    "test_update_post_valid": {
        "endpoint": "/posts/1",
        "payload": {
            "id": 1,
            "title": "Updated Title",
            "body": "Updated body content via PUT request.",
            "userId": 1
        },
        "expected_status": 200,
        "description": "Update an existing post with valid data"
    },
    "test_update_post_partial": {
        "endpoint": "/posts/1",
        "payload": {
            "title": "Only Title Updated"
        },
        "expected_status": 200,
        "description": "Update post with partial payload"
    }
}
```

---

## 5️⃣ Conftest & Fixtures

### `tests/conftest.py`

```python
import json
import pytest
from utils.api_client import APIClient


@pytest.fixture(scope="session")
def api_client():
    """Provide a shared API client for the entire test session."""
    return APIClient()


@pytest.fixture(scope="session")
def get_test_data():
    """Load GET test data from JSON."""
    with open("test_data/get_test_data.json", "r") as f:
        return json.load(f)


@pytest.fixture(scope="session")
def post_test_data():
    """Load POST test data from JSON."""
    with open("test_data/post_test_data.json", "r") as f:
        return json.load(f)


@pytest.fixture(scope="session")
def put_test_data():
    """Load PUT test data from JSON."""
    with open("test_data/put_test_data.json", "r") as f:
        return json.load(f)
```

### `conftest.py` (Root — Excel Hook)

```python
import pytest
from utils.excel_reporter import ExcelReporter

# Global Excel reporter instance
excel_reporter = ExcelReporter()


@pytest.hookimpl(tryfirst=True, hookwrapper=True)
def pytest_runtest_makereport(item, call):
    """Capture test results and feed them to the Excel reporter."""
    outcome = yield
    report = outcome.get_result()

    if report.when == "call":
        # Extract metadata from test markers
        method = "UNKNOWN"
        endpoint = "UNKNOWN"
        status_code = 0
        expected_status = 0

        for marker in item.iter_markers("test_meta"):
            method = marker.kwargs.get("method", method)
            endpoint = marker.kwargs.get("endpoint", endpoint)
            status_code = marker.kwargs.get("status_code", status_code)
            expected_status = marker.kwargs.get("expected_status", expected_status)

        result = "PASS" if report.passed else "FAIL"
        error_msg = str(report.longrepr) if report.failed else ""
        duration_ms = report.duration * 1000

        excel_reporter.add_result(
            test_name=item.name,
            method=method,
            endpoint=endpoint,
            status_code=status_code,
            expected_status=expected_status,
            result=result,
            duration_ms=duration_ms,
            error_message=error_msg[:500],
        )


def pytest_sessionfinish(session, exitstatus):
    """Generate the Excel report at the end of the session."""
    if excel_reporter.results:
        excel_reporter.generate_report()
```

---

## 6️⃣ Test Scripts

### `tests/test_get_endpoints.py`

```python
import pytest
import allure
from utils.assertions import Assertions

# Custom marker for Excel metadata
test_meta = pytest.mark.test_meta


@allure.epic("API Testing")
@allure.feature("GET Endpoints")
class TestGetEndpoints:

    @allure.story("Get All Users")
    @allure.severity(allure.severity_level.CRITICAL)
    @allure.title("TC-GET-001: Fetch all users")
    @allure.description("Verify GET /users returns a list of users with status 200")
    def test_get_all_users(self, api_client, get_test_data, request):
        data = get_test_data["test_get_all_users"]
        response = api_client.get(data["endpoint"])

        # Attach metadata for Excel reporting
        request.node.add_marker(test_meta(
            method="GET",
            endpoint=data["endpoint"],
            status_code=response.status_code,
            expected_status=data["expected_status"],
        ))

        Assertions.assert_status_code(response, data["expected_status"])
        Assertions.assert_non_empty_list(response)
        Assertions.assert_response_time(response, max_ms=2000)

    @allure.story("Get User by ID")
    @allure.severity(allure.severity_level.CRITICAL)
    @allure.title("TC-GET-002: Fetch user by valid ID")
    @allure.description("Verify GET /users/1 returns correct user data")
    def test_get_user_by_id(self, api_client, get_test_data, request):
        data = get_test_data["test_get_user_by_id"]
        response = api_client.get(data["endpoint"])

        request.node.add_marker(test_meta(
            method="GET",
            endpoint=data["endpoint"],
            status_code=response.status_code,
            expected_status=data["expected_status"],
        ))

        Assertions.assert_status_code(response, data["expected_status"])
        Assertions.assert_response_has_key(response, "name")
        Assertions.assert_response_value(response, "name", data["expected_name"])

    @allure.story("Get Invalid User")
    @allure.severity(allure.severity_level.NORMAL)
    @allure.title("TC-GET-003: Fetch user with non-existent ID")
    @allure.description("Verify GET /users/99999 returns 404")
    def test_get_invalid_user(self, api_client, get_test_data, request):
        data = get_test_data["test_get_invalid_user"]
        response = api_client.get(data["endpoint"])

        request.node.add_marker(test_meta(
            method="GET",
            endpoint=data["endpoint"],
            status_code=response.status_code,
            expected_status=data["expected_status"],
        ))

        Assertions.assert_status_code(response, data["expected_status"])

    @allure.story("Get Users with Query Parameters")
    @allure.severity(allure.severity_level.NORMAL)
    @allure.title("TC-GET-004: Fetch users filtered by username")
    def test_get_users_with_params(self, api_client, request):
        endpoint = "/users"
        params = {"username": "Bret"}
        response = api_client.get(endpoint, params=params)

        request.node.add_marker(test_meta(
            method="GET", endpoint=endpoint,
            status_code=response.status_code, expected_status=200,
        ))

        Assertions.assert_status_code(response, 200)
        json_data = response.json()
        assert len(json_data) == 1, f"Expected 1 user, got {len(json_data)}"
        assert json_data[0]["username"] == "Bret"
```

### `tests/test_post_endpoints.py`

```python
import pytest
import allure
from utils.assertions import Assertions

test_meta = pytest.mark.test_meta


@allure.epic("API Testing")
@allure.feature("POST Endpoints")
class TestPostEndpoints:

    @allure.story("Create New Post")
    @allure.severity(allure.severity_level.CRITICAL)
    @allure.title("TC-POST-001: Create a new post with valid data")
    @allure.description("Verify POST /posts creates a new resource and returns 201")
    def test_create_post_valid(self, api_client, post_test_data, request):
        data = post_test_data["test_create_post_valid"]
        response = api_client.post(data["endpoint"], payload=data["payload"])

        request.node.add_marker(test_meta(
            method="POST",
            endpoint=data["endpoint"],
            status_code=response.status_code,
            expected_status=data["expected_status"],
        ))

        Assertions.assert_status_code(response, data["expected_status"])
        Assertions.assert_response_has_key(response, "id")
        Assertions.assert_response_value(response, "title", data["payload"]["title"])

    @allure.story("Create Post with Empty Body")
    @allure.severity(allure.severity_level.MINOR)
    @allure.title("TC-POST-002: Create post with empty title and body")
    def test_create_post_empty_body(self, api_client, post_test_data, request):
        data = post_test_data["test_create_post_empty_body"]
        response = api_client.post(data["endpoint"], payload=data["payload"])

        request.node.add_marker(test_meta(
            method="POST",
            endpoint=data["endpoint"],
            status_code=response.status_code,
            expected_status=data["expected_status"],
        ))

        Assertions.assert_status_code(response, data["expected_status"])
        Assertions.assert_response_has_key(response, "id")

    @allure.story("Create Post Without Auth")
    @allure.severity(allure.severity_level.CRITICAL)
    @allure.title("TC-POST-003: Verify response schema on post creation")
    def test_create_post_schema_validation(self, api_client, request):
        endpoint = "/posts"
        payload = {
            "title": "Schema Validation Test",
            "body": "Checking response schema",
            "userId": 5,
        }
        response = api_client.post(endpoint, payload=payload)

        request.node.add_marker(test_meta(
            method="POST", endpoint=endpoint,
            status_code=response.status_code, expected_status=201,
        ))

        Assertions.assert_status_code(response, 201)
        json_data = response.json()

        # Schema validation
        required_keys = ["id", "title", "body", "userId"]
        for key in required_keys:
            assert key in json_data, f"Missing key '{key}' in response"
```

### `tests/test_put_endpoints.py`

```python
import pytest
import allure
from utils.assertions import Assertions

test_meta = pytest.mark.test_meta


@allure.epic("API Testing")
@allure.feature("PUT Endpoints")
class TestPutEndpoints:

    @allure.story("Update Existing Post")
    @allure.severity(allure.severity_level.CRITICAL)
    @allure.title("TC-PUT-001: Update an existing post with valid data")
    @allure.description("Verify PUT /posts/1 updates the resource and returns 200")
    def test_update_post_valid(self, api_client, put_test_data, request):
        data = put_test_data["test_update_post_valid"]
        response = api_client.put(data["endpoint"], payload=data["payload"])

        request.node.add_marker(test_meta(
            method="PUT",
            endpoint=data["endpoint"],
            status_code=response.status_code,
            expected_status=data["expected_status"],
        ))

        Assertions.assert_status_code(response, data["expected_status"])
        Assertions.assert_response_value(response, "title", data["payload"]["title"])
        Assertions.assert_response_value(response, "body", data["payload"]["body"])

    @allure.story("Update Post with Partial Payload")
    @allure.severity(allure.severity_level.NORMAL)
    @allure.title("TC-PUT-002: Update post with partial payload")
    def test_update_post_partial(self, api_client, put_test_data, request):
        data = put_test_data["test_update_post_partial"]
        response = api_client.put(data["endpoint"], payload=data["payload"])

        request.node.add_marker(test_meta(
            method="PUT",
            endpoint=data["endpoint"],
            status_code=response.status_code,
            expected_status=data["expected_status"],
        ))

        Assertions.assert_status_code(response, data["expected_status"])
        Assertions.assert_response_value(response, "title", data["payload"]["title"])

    @allure.story("Update Non-Existent Post")
    @allure.severity(allure.severity_level.NORMAL)
    @allure.title("TC-PUT-003: Attempt to update a non-existent resource")
    def test_update_non_existent_post(self, api_client, request):
        endpoint = "/posts/99999"
        payload = {"title": "Ghost Post", "body": "Should not exist", "userId": 1}
        response = api_client.put(endpoint, payload=payload)

        request.node.add_marker(test_meta(
            method="PUT", endpoint=endpoint,
            status_code=response.status_code, expected_status=500,
        ))

        # JSONPlaceholder returns 500 for non-existent IDs on PUT
        Assertions.assert_status_code(response, 500)
```

---

## 7️⃣ Pytest Configuration

### `pytest.ini`

```ini
[pytest]
testpaths = tests
markers =
    test_meta: Attach metadata for Excel reporting (method, endpoint, status_code, expected_status)
addopts =
    -v
    --tb=short
    --alluredir=reports/allure-results
    --clean-alluredir
log_cli = true
log_cli_level = INFO
```

---

## 8️⃣ Running the Tests

### Run All Tests

```bash
# Run all tests with Allure and Excel reporting
pytest
```

### Run Specific Test Modules

```bash
# Only GET tests
pytest tests/test_get_endpoints.py -v

# Only POST tests
pytest tests/test_post_endpoints.py -v

# Only PUT tests
pytest tests/test_put_endpoints.py -v
```

### Run Tests with Markers or Keywords

```bash
# Run a specific test by name
pytest -k "test_get_all_users" -v

# Run tests matching a pattern
pytest -k "test_create" -v
```

### Run with Parallel Execution (Optional)

```bash
pip install pytest-xdist
pytest -n 4 --alluredir=reports/allure-results
```

---

## 9️⃣ Generating Reports

### Allure Report

```bash
# Generate and open the Allure HTML report
allure serve reports/allure-results

# Or generate static report
allure generate reports/allure-results -o reports/allure-report --clean
allure open reports/allure-report
```

### Excel Report

The Excel report is **automatically generated** at the end of each test session via the `pytest_sessionfinish` hook. Find it at:

```
reports/excel/test_results_YYYYMMDD_HHMMSS.xlsx
```

The Excel workbook contains three sheets:

| Sheet              | Content                                          |
|--------------------|--------------------------------------------------|
| **Summary**        | Total, passed, failed counts and pass rate       |
| **Detailed Results** | Every test case with method, endpoint, status, duration |
| **Failures**       | Only failed tests with full error messages       |

---

## 🔟 Test Results Folder Structure After Execution

```
reports/
├── allure-results/
│   ├── <uuid>-container.json      # Test container metadata
│   ├── <uuid>-result.json         # Individual test results
│   ├── <uuid>-attachment.json     # Response body attachments
│   └── environment.properties     # Environment info for report
│
├── allure-report/
│   ├── index.html                 # 🌐 Main Allure dashboard
│   ├── data/
│   │   ├── test-cases/            # Per-test detailed reports
│   │   ├── categories.json        # Failure categorization
│   │   └── suites.json            # Test suite breakdown
│   ├── widgets/
│   │   ├── summary.json           # Pass/fail summary widget
│   │   └── duration-trend.json    # Duration trend data
│   └── history/
│       └── history-trend.json     # Historical trend data
│
└── excel/
    └── test_results_20260225_143022.xlsx
        ├── Sheet: Summary           # Pass/Fail overview
        ├── Sheet: Detailed Results  # All test case details
        └── Sheet: Failures          # Failure-only view
```

---

## 1️⃣1️⃣ Identifying Failures & Successes

### In Allure Reports

| View              | What It Shows                                           |
|-------------------|---------------------------------------------------------|
| **Overview**      | Pie chart of pass/fail/broken/skipped percentages       |
| **Suites**        | Hierarchical view grouped by `@allure.epic/feature/story` |
| **Graphs**        | Duration trends, severity distribution, status breakdown |
| **Timeline**      | Parallel execution timeline per thread                  |
| **Categories**    | Auto-categorized failures (e.g., Product Defects, Test Defects) |
| **Behaviors**     | BDD-style grouping via epic → feature → story           |

### In Excel Reports

- **Summary Sheet**: Instant pass/fail count with color-coded cells (green = pass, red = fail)
- **Detailed Results Sheet**: Every test row is color-coded by result
- **Failures Sheet**: Dedicated view listing only failures with full error messages and timestamps

### In Logs

```
logs/test_run_20260225_143022.log
```

Search for `ERROR` or `FAIL` in log files to quickly identify failures:

```bash
grep -i "error\|fail" logs/test_run_*.log
```

---

## 1️⃣2️⃣ Adding Allure Environment Info

Create `reports/allure-results/environment.properties` before generating the report:

```properties
Base.URL=https://jsonplaceholder.typicode.com
Environment=Staging
Python.Version=3.11
Framework=Pytest
Tester=Deepak-Kumar
```

This appears on the Allure report's **Overview** page for traceability.

---

## 1️⃣3️⃣ CI/CD Integration (GitHub Actions Example)

```yaml
name: API Test Suite

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 6 * * *'   # Daily at 6 AM UTC

jobs:
  api-tests:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install Dependencies
        run: pip install -r requirements.txt

      - name: Run API Tests
        run: pytest --alluredir=reports/allure-results -v
        continue-on-error: true

      - name: Upload Allure Results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: allure-results
          path: reports/allure-results/

      - name: Upload Excel Report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: excel-test-report
          path: reports/excel/

      - name: Generate Allure Report
        uses: simple-elf/allure-report-action@master
        if: always()
        with:
          allure_results: reports/allure-results

      - name: Deploy Allure Report to GitHub Pages
        uses: peaceiris/actions-gh-pages@v3
        if: always()
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: allure-history
```

---

## 1️⃣4️⃣ Best Practices

| Practice | Description |
|----------|-------------|
| **Data-driven testing** | Store test data in JSON files, not hardcoded in tests |
| **Fixture reuse** | Use `scope="session"` for API clients to avoid reconnections |
| **Allure annotations** | Use `@allure.epic`, `@allure.feature`, `@allure.story` for hierarchy |
| **Custom markers** | Use `@pytest.mark.test_meta` for Excel metadata injection |
| **Assertion helpers** | Centralize assertions in `utils/assertions.py` for consistency |
| **Environment isolation** | Use `config.py` with env-based URLs for multi-env testing |
| **Logging** | Log every request/response for post-failure debugging |
| **CI/CD** | Run tests on every PR and schedule nightly regression runs |
| **Report archiving** | Upload both Allure & Excel reports as CI artifacts |

---

## 1️⃣5️⃣ Quick Start Checklist

```bash
# 1. Clone/create project structure
mkdir -p api-test-suite/{config,test_data,tests,utils,reports/{allure-results,allure-report,excel},logs}

# 2. Install dependencies
pip install -r requirements.txt

# 3. Run all tests
pytest

# 4. View Allure report
allure serve reports/allure-results

# 5. Open Excel report
open reports/excel/test_results_*.xlsx    # macOS
start reports/excel/test_results_*.xlsx   # Windows
```

---

> **📝 Note:** This documentation uses [JSONPlaceholder](https://jsonplaceholder.typicode.com) as a
> demo API. Replace the `BASE_URL` in `config/config.py` and update the test data JSON files to
> point to your actual API endpoints.
