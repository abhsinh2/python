When we write Python code and we do test the code by writing unit tests. When development code involves any IO operation then we try to mock it using python mock functions. 

One approach which I do not like is decorating test function with multiple patch statements. For example:

```python
from unittest.mock import patch
>>> @patch('module.ClassName2')
... @patch('module.ClassName1')
... def test(MockClass1, MockClass2):
...     module.ClassName1()
...     module.ClassName2()
...     assert MockClass1 is module.ClassName1
...     assert MockClass2 is module.ClassName2
...     assert MockClass1.called
...     assert MockClass2.called
... 
```

Above example taken from Python documentation.

This is how I plan my python projects and unit tests:

## Folder structure:

```
Project Folder:
|--mylib
        |--services
            __init__.py
            http_service.py
            users.py
        __init__.py
|--tests
        |--services
            __init__.py
            test_users.py
        __init__.py
```

For every dev module, like services above, I create corresponding module in tests.

## Development Code:

`mylib/services/http_service.py`

```python
import requests


class HttpService:
    @classmethod
    def get(cls, url, **kwargs) -> requests.Response:
        return requests.get(url=url, **kwargs)

```

`mylib/services/users.py`

```python
from mylib.services.http_service import HttpService


class UserService:
    def __init__(self, host: str, port: int):
        self.host = host
        self.port = port

    def __get_url(self, uri):
        return "http://{}:{}/{}".format(self.host, self.port, uri)

    def get_users(self):
        response = HttpService.get(url=self.__get_url(uri="users"))
        if response.status_code == 200:
            return response.json()
        return {}

```

## Unit test Code:

First I create some utility patch functions in `tests/__init__.py`

```python
import asyncio
from typing import Dict
from unittest.mock import patch
from unittest import mock


def create_patch(test, name):
    patcher = patch(name)
    thing = patcher.start()
    test.addCleanup(patcher.stop)
    return thing


def mock_test(test, module, return_value=None, side_effect=None):
    _mock_module = create_patch(test, module)

    if side_effect is not None:
        _mock_module.side_effect = side_effect
    else:
        def _side_effect(*args, **kwargs):
            return return_value

        _mock_module.side_effect = _side_effect

    return _mock_module


def mock_async_test(test, module, return_value=None, side_effect=None):
    _mock_module = create_patch(test, module)

    if side_effect is not None:
        _mock_module.side_effect = asyncio.coroutine(side_effect)
    else:
        def _side_effect(*args, **kwargs):
            return return_value

        _mock_module.side_effect = asyncio.coroutine(_side_effect)

    return _mock_module


def mock_http_response(
        status_code: int = None,
        headers: Dict[str, str] = None,
        content: str = None,
        json_data=None):
    mock_resp = mock.Mock()

    if status_code is None:
        mock_resp.status_code = 200
    else:
        mock_resp.status_code = status_code

    if headers is None:
        mock_resp.headers = dict()
    else:
        mock_resp.headers = headers

    if content is not None:
        mock_resp.content = content.encode()
        mock_resp.text = content

    if json_data is not None:
        mock_resp.headers['Content-Type'] = 'application/json'

        mock_resp.json = mock.Mock(
            return_value=json_data
        )

    return mock_resp


def _mock_async_http_response(
    status_code: int = None, 
    headers: Dict[str, str] = None,
    content: str = None, 
    json_data=None):
    mock_resp = mock.Mock()

    if status_code is None:
        mock_resp.status = 200
    else:
        mock_resp.status = status_code

    if headers is None:
        mock_resp.headers = dict()
    else:
        mock_resp.headers = headers

    if content is not None:
        mock_resp.text.side_effect = asyncio.coroutine(mock.MagicMock(
            return_value=content
        ))

    if json_data is not None:
        mock_resp.headers['Content-Type'] = 'application/json'

        mock_resp.json.side_effect = asyncio.coroutine(mock.MagicMock(
            return_value=json_data
        ))

    return mock_resp
```

Then I create mock function for every class in its test module file. For example,
I have development code HttpService and UserService at `mylib/services` to I will create mock functions in `tests/services/__init__.py`

```python
from tests import mock_test


class MockHttpService:
    class_name = "mylib.services.http_service.HttpService.{}"

    @classmethod
    def get(cls, test, return_value=None, side_effect=None):
        return mock_test(test, cls.class_name.format('get'), return_value, side_effect)


class MockUserService:
    class_name = "mylib.services.users.UserService.{}"

    @classmethod
    def get_users(cls, test, return_value=None, side_effect=None):
        return mock_test(test, cls.class_name.format('get_users'), return_value, side_effect)

```

Basically I create Mock<ClassName> and keep adding functions which I want to mock. In the above code if I have to add mock for post then I will add 
 
```python
@classmethod
def post(cls, test, return_value=None, side_effect=None):
    return mock_test(test, cls.class_name.format('post'), return_value, side_effect)
```

These mock functions will be used in writing unit tests, like

```python
from unittest import TestCase

from mylib.services.users import UserService
from tests import mock_http_response
from tests.services import MockHttpService
from requests.exceptions import ConnectionError


class TestUserService(TestCase):
    def test_get_users_response_sends_500(self):
        service = UserService(host="127.0.0.1", port=8080)
        MockHttpService.get(self, return_value=mock_http_response(status_code=500))
        users = service.get_users()
        self.assertEqual({}, users)

    def test_get_users_get_raise_connection_timeout(self):
        service = UserService(host="127.0.0.1", port=8080)

        def http_get(*args, **kwargs):
            raise ConnectionError("some exception")

        MockHttpService.get(self, side_effect=http_get)

        with self.assertRaises(ConnectionError):
            service.get_users()

    def test_get_users_valid_response(self):
        service = UserService(host="127.0.0.1", port=8080)
        MockHttpService.get(self, return_value=mock_http_response(
            status_code=200,
            json_data=[
                {
                    "name": "a1"
                }
            ]
        ))
        users = service.get_users()
        self.assertEqual(len(users), 1)

```

When I read above test, then I know dev code is using HttpService.get function which we mocking here. We can mock not only the return value of the function but also whole function as you can see in `test_get_users_get_raise_connection_timeout`.
