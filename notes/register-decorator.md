## Define base class for executing handlers

`mylib/handlers/base.py`

```python
from abc import ABC, abstractmethod
from enum import Enum


class HandlerStatus(Enum):
    SUCCESS = 'success'
    FAILURE = 'failed'


class Handler(ABC):
    @abstractmethod
    async def handle(self, data) -> HandlerStatus:
        pass


class TaskHandler(Handler, ABC):
    @abstractmethod
    async def on_start(self):
        pass

    @abstractmethod
    async def on_failure(self, error: Exception):
        pass

    @abstractmethod
    async def on_success(self):
        pass

    @abstractmethod
    async def execute_task(self, data) -> HandlerStatus:
        pass

    def get_class_name(self) -> str:
        return self.__class__.__name__

    async def handle(self, data) -> HandlerStatus:
        try:
            await self.on_start()

            status = await self.execute_task(data=data)
            if status is None or status == HandlerStatus.FAILURE:
                # Intentionally raising exception with "" because workflow may not raise exception and return FAILURE
                raise Exception("")
            await self.on_success()
            return status
        except Exception as ex:
            print("Exception in running workflow. {}".format(ex))
            return await self.__handle_error(ex)

    async def __handle_error(self, error: Exception) -> HandlerStatus:
        try:
            await self.on_failure(error=error)
        except Exception as ex:
            print("Exception in handling error. {}".format(ex))
        return HandlerStatus.FAILURE
```

## Create class to register handlers using decorator.

`mylib/handlers/register.py`

```python
from typing import Optional, Dict
from mylib.handlers.base import Handler


class HandlerRegistration:
    def __init__(self):
        self.__handlers = dict()

    def register(self, states):
        def decorator_register(func):
            for __st in states:
                __source = __st.get('source')
                __state = __st.get('state')

                __all_source_handlers = self.__handlers.get(__source)
                if __all_source_handlers is None:
                    self.__handlers[__source] = dict()
                    __all_source_handlers = self.__handlers.get(__source)

                __all_source_handlers[__state] = func
            return func

        return decorator_register

    def get_handler_class(self, source: str, state: str) -> Optional[Handler]:
        __all_handlers = self.__handlers.get(source)
        if __all_handlers is None:
            return None
        return __all_handlers.get(state)

    def get_handlers_for_source(self, source: str) -> Optional[Dict[str, Handler]]:
        return self.__handlers.get(source)

    def get_all_handlers(self) -> Dict[str, Dict[str, Handler]]:
        return self.__handlers


handler = HandlerRegistration()
```

## Write a manager to call handlers

`mylib/handlers/manager.py`

```python
from typing import Optional

from mylib.handlers.base import Handler, HandlerStatus
from mylib.handlers.register import handler


class HandlerManager:
    @classmethod
    def get_handler_class(cls, source: str, state: str) -> Optional[Handler]:
        return handler.get_handler_class(source=source, state=state)

    @classmethod
    def get_handler_obj(cls, source: str, state: str) -> Optional[Handler]:
        handler_class = cls.get_handler_class(source=source, state=state)
        if handler_class:
            print("Got handler {} for source {}".format(handler_class, source))
            return handler_class()
        return None

    @classmethod
    async def call_handler(cls, source: str, state: str, data=None) -> Optional[HandlerStatus]:
        print("Checking if any handler exists for {} on source {}".format(state, source))
        # logger.info("Event {}".format(event))

        handler_obj = cls.get_handler_obj(source=source, state=state)

        if handler_obj is not None:
            try:
                result = await handler_obj.handle(data)
                print("Got result {} from handler.".format(result))
                return result
            except Exception as ex:
                print("Exception while executing handler {}. {}".format(handler_obj, ex))
        else:
            print("No handler exists for state {} for source {}".format(state, source))
        return None
```

## Define your handlers

`mylib/handlers/myhandler.py`

```python
from mylib.handlers.base import Handler, HandlerStatus, TaskHandler
from mylib.handlers.register import handler


@handler.register(
    states=[
        {'source': 'local', 'state': "REGISTERED"},
    ]
)
class RegisteredHandler(Handler):
    async def handle(self, data) -> HandlerStatus:
        print("Executing RegisteredHandler")
        return HandlerStatus.SUCCESS


@handler.register(
    states=[
        {'source': 'local', 'state': "CONFIGURED"},
    ]
)
class ConfiguredHandler(TaskHandler):
    async def on_start(self):
        print("Starting {}".format(self.get_class_name()))

    async def on_failure(self, error: Exception):
        print("failed {}".format(self.get_class_name()))

    async def on_success(self):
        print("Succeed {}".format(self.get_class_name()))

    async def execute_task(self, data) -> HandlerStatus:
        print("Executing {}".format(self.get_class_name()))
        return HandlerStatus.SUCCESS
```

## Test

```python
import asyncio
import importlib
from unittest import TestCase

from mylib.handlers.manager import HandlerManager


class TestHandlers(TestCase):
    @classmethod
    def setUpClass(cls) -> None:
        importlib.import_module('mylib.handlers.myhandlers')

    def setUp(self):
        self.set_up_event_loop()

    def tearDown(self) -> None:
        self.tear_down_event_loop()

    def set_up_event_loop(self):
        self.event_loop = asyncio.new_event_loop()
        asyncio.set_event_loop(self.event_loop)

    def tear_down_event_loop(self):
        if hasattr(self, 'event_loop'):
            self.event_loop.close()

    def run_in_async_loop(self, run_test):
        # Run the async test
        coro = asyncio.coroutine(run_test)
        try:
            self.event_loop.run_until_complete(coro())
        finally:
            self.event_loop.close()

    def test_registered(self):
        async def run_test():
            await HandlerManager.call_handler(source="local", state="CONFIGURED")
            await HandlerManager.call_handler(source="local", state="REGISTERED")

        self.run_in_async_loop(run_test)
```

Note that importlib is required other decorators will not be executed and handlers will not be registered.
