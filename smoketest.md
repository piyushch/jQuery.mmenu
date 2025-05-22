name: Build and Smoke Test

on:
  push:
    branches: [ main ]
  pull_request:

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.10'

      - name: Install Python dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r smoke-test/requirements.txt  # Adjust path if needed

      - name: Build Spring Boot app
        run: ./mvnw clean package

      - name: Start Spring Boot app
        run: |
          java -jar target/your-app-name.jar &
          echo $! > app.pid

      - name: Wait for app to start
        run: |
          sleep 10  # OR write a smarter health check loop here

      - name: Run smoke test
        run: python smoke-test/smoke_test.py

      - name: Stop Spring Boot app
        run: |
          kill $(cat app.pid)
