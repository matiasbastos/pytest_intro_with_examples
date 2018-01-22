# Pytest: introduction with examples

### Features:

- Simple tests with no boilerplate:
  - With unittest: 

  ```python
  self.assertEqual(1 + 1, 2)
  ```

  - With pytest: 

  ```python
  assert 1 + 1 == 2
  ```


- Dependency injection model with fixtures! 

  ```python
  import pytest

  @pytest.fixture
  def foo():
      return 42

  def test_foo(foo):
      assert foo == 42
  ```

- No setUp and tearDown functions.

- Fine grained control over the life span of the objects with 'scope'.

- Promotes code reusability. 

- No big base classes with tons of black magic.

- Can run unittest or nose tests but fails on some complex tests :-/



## Fixtures!

### Basic fixture:

```python
@pytest.fixture
def player(database_service, cache_service):
    fake = Factory.create('en_US')
    return create_player(isActive=True, mobile_number=fake.msisdn())
    
def test_player_login(player):
    result = Player.login(username=player._plain_username,
                          password=player._plain_password,
                          ip='127.0.0.1',
                          is_mobile=False)
    assert type(result) == dict
```

### Dependant fixtures: 

```python
@pytest.fixture
def player(database_service, cache_service):
    fake = Factory.create('en_US')
    return create_player(isActive=True, mobile_number=fake.msisdn())
    
@pytest.fixture
def player_banned(player):
    player.isBanned = True
    return player._plain_username, player._plain_password
    
def test_player_login_banned(player_banned):
    result = Player.login(username=player_banned[0],
                          password=player_banned[1],
                          ip='127.0.0.1',
                          is_mobile=False)
    assert result == ERR_USER_BANNED
```

### Fixtures scope and tear down with yield:

Fixtures requiring network access depend on connectivity and are usually time-expensive to create. We can add a `scope='module'` parameter to the`@pytest.fixture` invocation to cause the decorated `database_service`  fixture function to only be invoked once per test *module* (the default is to invoke once per test *function*). Multiple test functions in a test module will thus each receive the same `db.transaction` fixture instance, thus saving time.

Scope options: “function” (default), “class”, “module” or “session”.

Pytest supports execution of fixture specific finalization code when the fixture goes out of scope. By using a `yield` statement instead of `return`, all the code after the *yield* statement serves as the teardown code.

```python
@pytest.fixture(scope="module")
def database_service(request):
    db = DatabaseTestService()
    db.on_setup_class()
    db.on_setup()
    yield db.transaction
    db.on_tear_down()

@pytest.fixture(scope="module")
def cache_service():
    cache = CacheTestService()
    yield cache.on_setup_class()
    
@pytest.fixture
def player(database_service, cache_service):
    fake = Factory.create('en_US')
    return create_player(isActive=True, mobile_number=fake.msisdn())
    
def test_player_login(player):
    result = Player.login(username=player._plain_username,
                          password=player._plain_password,
                          ip='127.0.0.1',
                          is_mobile=False)
    assert type(result) == dict    
```

### Parametrization:

```python
@pytest.fixture
def player(database_service, cache_service):
    fake = Factory.create('en_US')
    return create_player(isActive=True, mobile_number=fake.msisdn())

@pytest.mark.parametrize('country', ['PT', 'MT', None])
def test_player_login_stores_country_code(player, country):
    """ Test if the login is getting stored
    :params player: player object
    :params country: country code
    """
    result = Player.login(username=player._plain_username,
                          password=player._plain_password,
                          ip='127.0.0.1',
                          country_code=country,
                          tool_name='internal')
    assert type(result) == dict
    last_login = Login.get_last_login(player.player_id)
    assert last_login.country_code == country
```

### Lazy fixtures:

It helps to use fixtures in pytest.mark.parametrize: https://github.com/TvoroG/pytest-lazy-fixture

```python
@pytest.fixture
def player_inactive_with_email(player):
    player.isActive = False
    player.activated_email = False
    return player._plain_username, player._plain_password

@pytest.fixture
def player_inactive_with_phone(player):
    player.isActive = False
    player.activated_sms = False
    with LoginFields('phone'):
        yield player.mobile_number, player._plain_password
        
@pytest.mark.parametrize('user',
                         [pytest.lazy_fixture('player_inactive_with_email'),
                          pytest.lazy_fixture('player_inactive_with_phone')])
def test_player_login_user_inactive(user):
    result = Player.login(username=user[0],
                          password=user[1],
                          ip='127.0.0.1',
                          is_mobile=False)
    assert result == ERR_USER_INACTIVE
```

### Parameter tuples:

```python
@pytest.fixture
def player_banned(player):
    player.isBanned = True
    return player._plain_username, player._plain_password
  
@pytest.fixture
def player_inactive(player):
    player.isActive = False
    return player._plain_username, player._plain_password
  
@pytest.mark.parametrize('user,result', 
                         [(pytest.lazy_fixture('player_banned'), ERR_USER_BANNED),
                          (pytest.lazy_fixture('player_inactive'), ERR_USER_INACTIVE)])
def test_player_login_errors(user, result):
    """ Test login errors
    :params user: username/pass tuple
    :params result: expected error for the user param login
    """
    response = Player.login(username=user[0],
    						password=user[1],
    						ip='127.0.0.1',
    						is_mobile=False)
    assert result == response
```

### Marks:

Tests can be marked and run using cli filters:

```python
@pytest.mark.mymarker
def test_something():
    pass
  
@pytest.mark.myothermarker
def test_something():
    pass  
```

```bash
$ pytest -m "mymarker"
$ pytest -m "not mymarker"
```

### Auto use fixtures:

This fixtures are used without explicit request (global).

```python
@pytest.mark.linux
def test_mem_stack():
    assert MemSizes().stack == 42

@pytest.fixture(autouse=True)
def _platform_skip(request):
    marker = request.node.get_marker('linux')
    if marker and platform.system() != 'Linux':
        pytest.skip('N/A on {}'.format(platform.system()))
```



## Tests Structure

How tests could be organized (just an example):

```bash
tests
├── __init__.py
├── conftest.py
├── fixtures
│   ├── games.py
│   ├── players.py
│   └── transactions.py
├── functional
│   ├── fixtures
│   │   ├── gateways.py
│   │   └── mail.py
│   ├── test_campaigns.py
│   └── test_payment_gateway.py
└── unit
    ├── test_bets.py
    ├── test_player.py
    └── test_transactions.py
```

Test files could end in '_pytest.py' to make it easier to filter which tests are run by nosetests and which by pytest, for the case you can't migrate completely to pytest. 

### conftest.py

```python
# -*- coding: utf-8 -*-
import pytest
from fixtures import *


def pytest_make_parametrize_id(config, val):
    """ Provides better parameter ids
    """
    # hook to name parametrized tests
    if type(val).__name__ == 'LazyFixture':
        return val.name
    return repr(val)
```



## Usage tricks

Recommended site: https://hackebrot.github.io/pytest-tricks

### Assert exceptions:

```python
import pytest
def f():
    raise SystemExit(1)

def test_mytest():
    with pytest.raises(SystemExit):
        f()
```

### Access request info:

Fixture function can accept the `request` object to introspect the “requesting” test function, class or module context.

```python
import pytest
import smtplib

@pytest.fixture(scope="module")
def smtp(request):
    server = getattr(request.module, "smtpserver", "smtp.gmail.com")
    smtp = smtplib.SMTP(server, 587, timeout=5)
    yield smtp
    print ("finalizing %s (%s)" % (smtp, server))
    smtp.close()
    
@pytest.fixture
def db(request):
    if 'transactional_db' in request.fixturenames:
        pytest.fail('Conflicting fixtures')
    return no_transactions_db()
```

### CLI parameters:

```python
#conftest.py
def pytest_addoption(parser):
    parser.addoption('--ci', action='store_true',
                     help='Indicate tests are run on CI server')

@pytest.fixture
def fix(request):
    ci = request.config.getoption('ci')

# Test module
def test_foo(pytestconfig):
    ci = pytestconfig.getoption('ci')
```

### Skipping/failing tests

```python
@pytest.fixture(scope='session')
def redis_client(request):
    servers = ['localhost', 'staging']
    for hostname in servers:
        try:
            return redis.StrictRedis(hostname)
        except redis.ConnectionError:
            continue
    else:
        if request.config.getoption('ci'):
            pytest.fail('No Redis server found')
        else:
            pytest.skip('No Redis server found')
```

### Skipping parameters

```python
try:
    import cx_Oracle as ora
except ImportError:
    ora = None

needs_ora = pytest.mark.skipif(ora is None, reason='No Oracle installed')

@pytest.fixture(params=[
    'pg',
    needs_ora('ora'),
])
def dburi(request):
    return create_db_uri(request.param) 
```



## Plugins

### Included:

| `_pytest.assertion`     | support for presenting detailed information in failing assertions. |
| ----------------------- | ---------------------------------------- |
| `_pytest.cacheprovider` | merged implementation of the cache provider |
| `_pytest.capture`       | per-test stdout/stderr capturing mechanism. |
| `_pytest.config`        | command line options, ini-file and conftest.py processing. |
| `_pytest.doctest`       | discover and run doctests in modules and test files. |
| `_pytest.helpconfig`    | version info, help messages, tracing configuration. |
| `_pytest.junitxml`      | report test results in JUnit-XML format, |
| `_pytest.mark`          | generic mechanism for marking and selecting python functions. |
| `_pytest.monkeypatch`   | monkeypatching and mocking functionality. |
| `_pytest.nose`          | run test suites written for nose.        |
| `_pytest.pastebin`      | submit failure or test session information to a pastebin service. |
| `_pytest.debugging`     | interactive debugging with PDB, the Python Debugger. |
| `_pytest.pytester`      | (disabled by default) support for testing pytest and pytest plugins. |
| `_pytest.python`        | Python test discovery, setup and run of test functions. |
| `_pytest.recwarn`       | recording warnings during test function execution. |
| `_pytest.resultlog`     | log machine-parseable test session result information in a plain |
| `_pytest.runner`        | basic collect and runtest protocol implementations |
| `_pytest.main`          | core implementation of testing process: init, session, runtest loop. |
| `_pytest.skipping`      | support for skip/xfail functions and markers. |
| `_pytest.terminal`      | terminal reporting of the full testing process. |
| `_pytest.tmpdir`        | support for providing temporary directories to test functions. |
| `_pytest.unittest`      | discovery and running of std-library “unittest” style tests. |

### Community:

- [pytest-django](http://pytest-django.readthedocs.org/): `--create-db`, fixtures (rf, client, admin_client...)
- [pytest-cov](https://pypi.python.org/pypi/pytest-cov): `--cov myproj tests/`
- [pytest-cache](https://pythonhosted.org/pytest-cache/readme.html): `--lf`

### Parallel tests (WIP):

Run parallel tests using [pytest-xdist](https://pytest.org/latest/xdist.html) plugin:

```bash
$ pytest -n 2
```

```python
@pytest.fixture(scope="module")
def parallel_database_service(request, worker_id):
    import sqlalchemy
    from settings.testing.database import DATABASE_PARAMS
    # create db
    DATABASE_PARAMS['name'] = '_'.join([DATABASE_PARAMS['name'], worker_id])
    engine = sqlalchemy.create_engine('mysql://{user}:{pass}@{host}:{port}'.format(**DATABASE_PARAMS)) # connect to server
    connection = engine.connect()
    engine.execute("CREATE DATABASE IF NOT EXISTS %s CHARACTER SET UTF8 collate utf8_general_ci;" % DATABASE_PARAMS['name'])
    # hack settings
    settings.sqlalchemy = 'mysql+mysqldb://{user}:{pass}@{host}:{port}/{name}?charset=utf8&use_unicode=1'.format(**DATABASE_PARAMS)
    # db connection
    db = DatabaseTestService()
    db.on_setup_class()
    db.on_setup()
    yield db.transaction
    db.on_tear_down()
    # drop db
    engine.execute("DROP DATABASE %s;" % DATABASE_PARAMS['name'])
    connection.close()
```

For more info about parallel tests: http://developer.paylogic.com/articles/test-p14n.html



## TOX

### Why?

#### Standardize testing in Python

`tox` aims to automate and standardize testing in Python. 

#### What is tox?

Tox is a generic [virtualenv](https://pypi.python.org/pypi/virtualenv) management and test command line tool you can use for:

- Checking your package installs correctly with different Python versions and interpreters
- Running your tests in each of the environments, configuring your test tool of choice
- Acting as a frontend to Continuous Integration servers, greatly reducing boilerplate and merging CI and shell-based testing.

### tox.ini - for coreapi:

```ini
[tox]
envlist = py27
skipsdist = True

[testenv]
setenv = PYTHONPATH = {toxinidir}
commands =
    pytest -v
    nosetests -v --ignore-files="(^.*pytest\.py$)"
deps = -r{toxinidir}/requirements.txt

[pytest]
python_files = *pytest.py
norecursedirs = .tox .git
```
