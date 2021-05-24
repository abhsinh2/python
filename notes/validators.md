# Overview

Often we write validations for APIs. Sometime validations are dependent and sometime independent. Sometime validations need to be grouped together and sometime not.

For example, suppose we take an IP to configure a device. We need to add validation like:
1. Is IP non empty?
2. If 1. is true then where format of IP is correct.
3. If 2. is true then check whether IP is pingable and IP belongs to matching subnet.

In the above '2' can only executed if '1' pass, same is true for 3. But in '3', we can validate the two independently.

Another example, suppose you take IP, username, password in an API. Then you would like group your validations into two, one of IP and second for credentials.

# Implementation

## Define error

`mylib/validators/errors.py`

```python
class ValidationError(Exception):
    def __init__(self, *args):
        super().__init__(*args)
```

## Define base classes

`mylib/validators/base.py`

```python
from abc import ABC, abstractmethod
from enum import Enum
from typing import Dict, Any, List, Optional

import attr

from mylib.validators.errors import ValidationError


class StatusType(Enum):
    SUCCESS = "SUCCESS"
    FAILURE = "FAILURE"
    SKIP = "SKIP"

    @staticmethod
    def from_value(value: str) -> 'StatusType':
        for data in StatusType:
            if data.value == value:
                return data
        return StatusType.SKIP


@attr.s
class ValidationResult:
    label = attr.ib(type=str, default="")
    status = attr.ib(type=StatusType, default=StatusType.SKIP)
    error = attr.ib(type=ValidationError, default=None)

    def has_error(self) -> bool:
        return self.error is not None

    def to_json(self) -> Dict[str, Any]:
        return {
            'label': self.label,
            'status': self.status.value,
            'error': str(self.error)
        }

    @classmethod
    def from_json(cls, json_data: Dict[str, Any]) -> 'ValidationResult':
        label = json_data.get('label')
        status = StatusType.from_value(json_data.get('status'))
        error = None
        if json_data.get('error'):
            error = ValidationError(json_data.get('error'))
        return ValidationResult(label=label, status=status, error=error)


@attr.s
class GroupValidationResult:
    group = attr.ib(type=str, default="Unknown")
    status = attr.ib(type=StatusType, default=StatusType.SKIP)
    results = attr.ib(type=List[ValidationResult], default=attr.Factory(list))

    def add_result(self, result: ValidationResult):
        if result is not None:
            self.results.append(result)

    def has_errors(self) -> bool:
        if len(self.to_errors()) > 0:
            return True
        return False

    def calculate_status(self) -> 'GroupValidationResult':
        if self.has_errors():
            self.status = StatusType.FAILURE
        else:
            self.status = StatusType.SUCCESS
        return self

    def to_errors(self) -> List[ValidationError]:
        all_errors = list()
        for each_result in self.results:
            if each_result.error is not None:
                all_errors.append(each_result.error)
        return all_errors

    @classmethod
    def create_from_error(cls, label: str, error: ValidationError = None) -> 'GroupValidationResult':
        return GroupValidationResult(
            status=StatusType.FAILURE,
            results=[
                ValidationResult(
                    label=label,
                    status=StatusType.FAILURE,
                    error=error
                )
            ]
        )

    def to_json(self) -> Dict[str, Any]:
        result_json = list(map(lambda each_result: each_result.to_json(), self.results))
        return {
            'group': self.group,
            'status': self.status.value,
            'results': result_json
        }

    @classmethod
    def from_json(cls, json_data: Dict[str, str]) -> 'GroupValidationResult':
        group = json_data.get('group')
        status = StatusType.from_value(json_data.get('status'))
        obj = GroupValidationResult(group=group, status=status)

        for each_result in json_data.get('results'):
            obj.add_result(ValidationResult.from_json(each_result))
        return obj


class ValidationBuilder:
    def __init__(self, error: ValidationError = None):
        self.error = error

    def to_error(self) -> Optional[ValidationError]:
        return self.error

    def has_error(self) -> bool:
        return self.error is not None

    def to_result(self, success: str = None, failure: str = None) -> Optional[ValidationResult]:
        if self.error:
            return ValidationResult(
                label=failure,
                status=StatusType.FAILURE,
                error=self.error
            )
        else:
            return ValidationResult(
                label=success,
                status=StatusType.SUCCESS
            )

    def raise_exception(self, raise_exception: bool = False) -> 'ValidationBuilder':
        error = self.to_error()
        if raise_exception is True and error is not None:
            raise error
        return self


class Validator(ABC):
    @abstractmethod
    def validate(self, raise_exception: bool = False) -> ValidationBuilder:
        pass


class SimpleValidator(Validator):
    def __init__(self, error: ValidationError = None):
        self.error = error

    def validate(self, raise_exception: bool = False) -> ValidationBuilder:
        return ValidationBuilder(error=self.error)


class GroupValidator(ABC):
    def __init__(self, name: str):
        self.name = name
        self.__group = GroupValidationResult(group=name)

    def add_validator(self, validator: Validator) -> 'GroupValidator':
        self.__group.add_result(validator.validate().to_result())
        return self

    def add_validation_result(self, result: ValidationResult) -> 'GroupValidator':
        self.__group.add_result(result)
        return self

    def validate(self) -> GroupValidationResult:
        return self.__group.calculate_status()

```

## Define validators

`mylib/validators/validators.py`

```python
import ipaddress
from typing import Optional

from mylib.validators.base import Validator, ValidationBuilder
from mylib.validators.errors import ValidationError


class NonEmptyParamValidator(Validator):
    def __init__(self, param: str, error: ValidationError):
        self.param = param
        self.error = error

    def validate(self, raise_exception: bool = False) -> ValidationBuilder:
        result = ValidationBuilder()

        if self.is_param_valid(self.param) is False:
            result = ValidationBuilder(
                error=self.error
            )

        return result.raise_exception(raise_exception=raise_exception)

    @classmethod
    def is_param_valid(cls, param):
        if param is None or len(str(param).strip()) == 0:
            return False
        return True


class IpValidator(Validator):
    def __init__(self, ip: str, error: Optional[ValidationError] = None):
        self.ip = ip
        self.error = error if error is not None else ValidationError("IP {} is invalid.".format(self.ip))

    def validate(self, raise_exception: bool = False) -> ValidationBuilder:
        result = ValidationBuilder()
        try:
            ipaddress.ip_address(address=self.ip)
        except ValueError:
            result = ValidationBuilder(
                error=self.error
            )

        return result.raise_exception(raise_exception=raise_exception)


class PingValidator(Validator):
    def __init__(self, ip: str, port: Optional[int] = None):
        self.ip = ip
        self.port = port

    def ping(self) -> bool:
        return False

    def validate(self, raise_exception: bool = False) -> ValidationBuilder:
        result = ValidationBuilder()

        if self.ping() is False:
            result = ValidationBuilder(
                error=ValidationError("Cannot ping {}".format(self.ip))
            )

        return result.raise_exception(raise_exception=raise_exception)


class CredentialValidator(Validator):
    def __init__(self, username: str, password: str):
        self.username = username
        self.password = password

    def check_credentials(self) -> bool:
        return False

    def validate(self, raise_exception: bool = False) -> ValidationBuilder:
        result = ValidationBuilder()

        if self.check_credentials() is False:
            result = ValidationBuilder(
                error=ValidationError("Invalid credentials.")
            )

        return result.raise_exception(raise_exception=raise_exception)

```

# Test

`tests/validators/test_validators.py`

```python
from unittest import TestCase

from mylib.validators.base import GroupValidator
from mylib.validators.errors import ValidationError
from mylib.validators.validators import IpValidator, PingValidator


class TestValidators(TestCase):
    def test_single(self):
        # Use of has error
        validator = IpValidator(
            ip="a.a.a"
        ).validate()
        self.assertEqual(validator.has_error(), True)

        # provider nice messages if success or failed
        validator_result = IpValidator(
            ip="a.a.a"
        ).validate().to_result(
            success="Database Ip {} is valid.".format("a.a.a"),
            failure="Database Ip {} is invalid.".format("a.a.a")
        )
        self.assertEqual(validator_result.label, "Database Ip a.a.a is invalid.")
        self.assertEqual(str(validator_result.error), "IP a.a.a is invalid.")

        # Raise Exception
        with self.assertRaises(ValidationError):
            IpValidator(
                ip="a.a.a"
            ).validate(raise_exception=True)

    def test_if_succeed(self):
        validator_result = IpValidator(
            ip="a.a.a"
        ).validate().to_result(
            success="Database Ip {} is valid.".format("a.a.a"),
            failure="Database Ip {} is invalid.".format("a.a.a")
        )

        if validator_result.has_error() is False:
            validator_result = PingValidator(
                ip="1.1.1.1"
            ).validate().to_result(
                success="Database Ip {} is pingable.".format("1.1.1.1"),
                failure="Database Ip {} is not pingable.".format("1.1.1.1")
            )

    def test_group(self):
        validator = GroupValidator(
            name="IP Validation"
        ).add_validation_result(
            IpValidator(
                ip="a.a.a"
            ).validate().to_result(
                success="Database Ip {} is valid.".format("a.a.a"),
                failure="Database Ip {} is invalid.".format("a.a.a")
            )
        ).add_validation_result(
            IpValidator(
                ip="b.b.b"
            ).validate().to_result(
                success="Web server Ip {} is valid.".format("b.b.b"),
                failure="Web server Ip {} is invalid.".format("b.b.b")
            )
        ).validate()

        self.assertEqual(len(validator.to_errors()), 2)

```
