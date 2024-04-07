# Testing

We use GitHub actions for testing and maintaining code quality. There are two sets of templates that you can use for running the tests and analyzers.

## Without using docker

If the library doesn't require docker to function and run tests, use these set of templates to create GitHub action for those libraries.

1. tests.yml

```yml
name: "Tests"

on: [pull_request]
jobs:
  lint:
    name: Tests ${{ matrix.php-versions }}
    runs-on: ubuntu-latest
    strategy:
      matrix:
        php-versions: ['8.1', '8.2', '8.3', 'nightly']

    steps:
    - name: Checkout repository
      uses: actions/checkout@v3

    - name: Setup PHP ${{ matrix.php-versions }}
      uses: shivammathur/setup-php@v2
      with:
        php-version: ${{ matrix.php-versions }}

    - name: Validate composer.json and composer.lock
      run: composer validate --strict
    
    - name: Compose install
      run: composer install --ignore-platform-reqs

    - name: Run tests
      run: composer test
```

2. linter.yml

```yml
name: "Linter"

on: [pull_request]
jobs:
  lint:
    name: Linter
    runs-on: ubuntu-latest

    steps:
    - name: Checkout repository
      uses: actions/checkout@v3
      with:
        fetch-depth: 2

    - run: git checkout HEAD^2

    - name: Run Linter
      run: |
        docker run --rm -v $PWD:/app composer sh -c \
        "composer install --profile --ignore-platform-reqs && composer lint"
```

3. codeql-analysis.yml

```yml
name: "CodeQL"

on: [pull_request]
jobs:
  lint:
    name: CodeQL
    runs-on: ubuntu-latest

    steps:
    - name: Checkout repository
      uses: actions/checkout@v3

    - name: Run CodeQL
      run: |
        docker run --rm -v $PWD:/app composer sh -c \
        "composer install --profile --ignore-platform-reqs && composer check"
```


## Using docker

If the library requires docker to function and run tests use these set of templates to create GitHub action for those libraries.

1. tests.yml

```yaml
name: "Tests"

on: [ pull_request ]
jobs:
  lint:
    name: Tests
    runs-on: ubuntu-latest
    strategy:
      matrix:
        php-versions: ['8.1', '8.2', '8.3'] # add PHP versions as required

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3
        with:
          fetch-depth: 2

      - run: git checkout HEAD^2

      - name: Build
        run: |
          export PHP_VERSION=${{ matrix.php-versions }}
          docker compose build
          docker compose up -d
          sleep 10

      - name: Run Tests
        run: docker compose exec tests vendor/bin/phpunit --configuration phpunit.xml tests
```

You also need to create a corresponding Dockerfile for each PHP version in the format `Dockerfile-php-[PHP_VERSION]`. and in the docker compose the service should be defined as the following so that it uses the corresponding Dockerfile for each PHP version test. Where `8.3` is a default version if `PHP_VERSION` environmanet variable is not specified.

```yml
tests:
    build:
      context: .
      dockerfile: Dockerfile.php-${PHP_VERSION:-8.3}
```

For the above test file where you are testing 3 php versions, following Dockerfile should exist.

```
- Dockerfile-php-8.1
- Dockerfile-php-8.2
- Dockerfile-php-8.3
```

For linter and code analysis we can use only one PHP version.

2. linter.yml

```yml
name: "Linter"

on: [ pull_request ]
jobs:
  lint:
    name: Linter
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3
        with:
          fetch-depth: 2

      - run: git checkout HEAD^2

      - name: Run Linter
        run: |
          docker run --rm -v $PWD:/app composer sh -c \
          "composer install --profile --ignore-platform-reqs && composer lint"
```

3. codeql-analysis.yml

```yml
name: "CodeQL"

on: [ pull_request ]
jobs:
  lint:
    name: CodeQL
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3
        with:
          fetch-depth: 2

      - run: git checkout HEAD^2

      - name: Run CodeQL
        run: |
          docker run --rm -v $PWD:/app composer sh -c \
          "composer install --profile --ignore-platform-reqs && composer check"
```

> Don't forget to define the scripts `check`, `lint` and `test` on `composer.json`