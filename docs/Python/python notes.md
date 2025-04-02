## Logging class

```py
import logging


class Logger:
    def __init__(self):
			self._logger = None

    def setup_logger(self):
			logger = logging.getLogger("root")

			if logger.handlers:
				return logger

			logger.setLevel(logging.INFO)
			formatter = logging.Formatter("%(asctime)s %(levelname)s %(filename)s:%(lineno)d %(module)s %(funcName)s - %(message)s")
			handler = logging.StreamHandler()
			handler.setFormatter(formatter)
			logger.addHandler(handler)

			return logger

    def get_logger(self):
			if self._logger is None:
				self._logger = self.setup_logger()

			return self._logger
```

To use:

```py
from path.to.logger import Logger

logger = Logger().get_logger()
```

## Creating decorator

```py
def logged_in_user(function) -> Any:
	@wraps(function)
	def get_logged_in_user(*args, **kwargs):
		current_user = get_valid_current_user()
		if not current_user:
			return reject_access()

		return function(*args, **kwargs, current_user=current_user)

	return get_logged_in_user


@app.post('/')
@logged_in_user
def app_root_page(current_user):
  ...
```

**Decorator with params**

```py
def some_decorator(val) -> Any:
    def some_decorator_func(function) -> Any:
        @wraps(function)
        def wrapper(*args, **kwargs):
						some_bool = val > 10
						# do other stuff
            return function(*args, **kwargs, some_bool=some_bool)

        return wrapper

    return some_decorator_func
```

## creating batch from list

```py
def create_batch(self, obj_ls: list["ObjCls"], batch_size: int):
	for i in range(0, len(obj_ls), batch_size):
		yield obj_ls[i : i + batch_size]
```

## List filtering

Supposing we need to filter out the results from the below list of dictionaries

```py
exam_results = [
	{"name": "John", "score": 70, "subject": "German"},
	{"name": "Jane", "score": 80, "subject": "Maths"},
	{"name": "Mary", "score": 60, "subject": "French"},
	{"name": "Sam", "score": 75, "subject": "Graphics"},
	{"name": "Alex", "score": 50, "subject": "Chemistry"},
	{"name": "Theo", "score": 40, "subject": "Chemistry"},
]
```

To retrieve exam results where `score` is higher than 65:

```py
filtered_result = list(filter(lambda x: x["score"] > 65, exam_results))
print(filtered_result)
# [{"name": "John", "score": 70, "subject": "German"}, {"name": "Jane", "score": 80, "subject": "Maths"}, {"name": "Sam", "score": 75, "subject": "Graphics"}]
```

To retrieve exam results where `score` is equal or higher than 50 and subject is either `Chemistry`, `French`, `Graphics`:

```py
filtered_subject_results = list(filter(lambda x, subject=subject_name: x["score"] >= 50 and x["subject"] in (["Chemistry", "French", "Graphics"]), exam_results))

print(filtered_subject_results)
# [{"name": "Mary", "score": 60, "subject": "French"}, {"name": "Sam", "score": 75, "subject": "Graphics"}, {"name": "Alex", "score": 50, "subject": "Chemistry"}]
```

...and with `subject` in specific order using for-loop

```py
filtered_subject_results = []
for subject_name in ["Chemistry", "French", "Graphics"]:
	result = list(filter(lambda x, subject=subject_name: x["score"] >= 50 and x["subject"] == subject_name, exam_results))
	filtered_subject_results = filtered_subject_results + result

print(filtered_subject_results)
# [{"name": "Alex", "score": 50, "subject": "Chemistry"}, {"name": "Mary", "score": 60, "subject": "French"}, {"name": "Sam", "score": 75, "subject": "Graphics"}]
```

...or with `sorted`

```py
filtered_subject_results = list(filter(lambda x, subject=subject_name: x["score"] >= 50 and x["subject"] in (["Chemistry", "French", "Graphics"]), exam_results))
filtered_subject_results = sorted(filtered_subject_results, key=lambda x: (x["subject"]))

print(filtered_subject_results)
# [{"name": "Alex", "score": 50, "subject": "Chemistry"}, {"name": "Mary", "score": 60, "subject": "French"}, {"name": "Sam", "score": 75, "subject": "Graphics"}]
```

## sort & groupby (itertools)

Assuming we have a list of dictionaries of students needed to be grouped by year-class:

```py
students_ls = [
	{"name": "John", "email": "john@random_school.edu.uk", "year": 1, "class": "A"},
	{"name": "Jane", "email": "jane@random_school.edu.uk", "year": 1, "class": "B"},
	{"name": "Mary", "email": "mary@random_school.edu.uk", "year": 2, "class": "C"},
	{"name": "Alex", "email": "alex@random_school.edu.uk", "year": 2, "class": "A"},
	{"name": "Sam", "email": "sam@random_school.edu.uk", "year": 3, "class": "A"},
	{"name": "Hannah", "email": "hannah@random_school.edu.uk", "year": 3, "class": "A"},
	{"name": "Kim", "email": "kim@random_school.edu.uk", "year": 3, "class": "B"},
	{"name": "Ted", "email": "ted@random_school.edu.uk", "year": 3, "class": "A"},
]
```

First, we have to sorted the list by year and class as the [documentation](https://docs.python.org/3/library/itertools.html#itertools.groupby) states that the list need to be sorted before applying `groupby`

```py
sorted_students_ls = sorted(students_ls, key=lambda x: (x["year"], x["class"]))
print(sorted_students_ls)
# [
#   {'class': 'A', 'email': 'john@random_school.edu.uk', 'name': 'John', 'year': 1},
#   {'class': 'B', 'email': 'jane@random_school.edu.uk', 'name': 'Jane', 'year': 1},
#   {'class': 'A', 'email': 'alex@random_school.edu.uk', 'name': 'Alex', 'year': 2},
#   {'class': 'C', 'email': 'mary@random_school.edu.uk', 'name': 'Mary', 'year': 2},
#   {'class': 'A', 'email': 'sam@random_school.edu.uk', 'name': 'Sam', 'year': 3},
#   {'class': 'A', 'email': 'hannah@random_school.edu.uk', 'name': 'Hannah', 'year': 3},
#   {'class': 'A', 'email': 'ted@random_school.edu.uk', 'name': 'Ted', 'year': 3},
#   {'class': 'B', 'email': 'kim@random_school.edu.uk', 'name': 'Kim', 'year': 3}
# ]
```

Then apply `groupby`:

```py
grouped_data_single_line = {f"{k[0]}-{k[1]}": list(students) for k, students in itertools.groupby(sorted_students_ls, key=lambda x: (x["year"], x["class"]))}
print(grouped_data_single_line)
# {
#   "1-A": [{'class': 'A', 'email': 'john@random_school.edu.uk', 'name': 'John', 'year': 1}],
#   "1-B": [{'class': 'B', 'email': 'jane@random_school.edu.uk', 'name': 'Jane', 'year': 1}],
#   "2-A": [{'class': 'A', 'email': 'alex@random_school.edu.uk', 'name': 'Alex', 'year': 2}],
#   "2-C": [{'class': 'C', 'email': 'mary@random_school.edu.uk', 'name': 'Mary', 'year': 2}],
#   "3-A": [{'class': 'A', 'email': 'sam@random_school.edu.uk', 'name': 'Sam', 'year': 3},
#           {'class': 'A', 'email': 'hannah@random_school.edu.uk', 'name': 'Hannah', 'year': 3},
#           {'class': 'A', 'email': 'ted@random_school.edu.uk', 'name': 'Ted', 'year': 3}],
#   "3-B": [{'class': 'B', 'email': 'kim@random_school.edu.uk', 'name': 'Kim', 'year': 3}]
# }
```

or below if require data formatting

```py
grouped_data = {}
for k, students in itertools.groupby(sorted_students_ls, key=lambda x: (x["year"], x["class"])):
	data_key = f"{k[0]}-{k[1]}"
	grouped_data[data_key] = [{"name": student["name"], "email": student["email"]} for student in list(students)]

print(grouped_data_single_line)
# {
#     '1-A': [{'email': 'john@random_school.edu.uk', 'name': 'John'}],
#     '1-B': [{'email': 'jane@random_school.edu.uk', 'name': 'Jane'}],
#     '2-A': [{'email': 'alex@random_school.edu.uk', 'name': 'Alex'}],
#     '2-C': [{'email': 'mary@random_school.edu.uk', 'name': 'Mary'}],
#     '3-A': [{'email': 'sam@random_school.edu.uk', 'name': 'Sam'},
#             {'email': 'hannah@random_school.edu.uk', 'name': 'Hannah'},
#             {'email': 'ted@random_school.edu.uk', 'name': 'Ted'}],
#     '3-B': [{'email': 'kim@random_school.edu.uk', 'name': 'Kim'}]
# }
```

## Dates related

** Get last day of week from a given date **

```py
from datetime import date, datetime, timedelta

def get_week_end(date_input: date):
	# use 6 for week ending on Sunday
	return date_input + timedelta(days=5 - date_input.weekday())
```

** Get last day of month from a given date **

```py
import bisect
import calendar

from datetime import date, datetime, timedelta

def get_month_end(date_input: date):
	last_day_of_month = calendar.monthrange(date_input.year, date_input.month)[1]
	return date(date_input.year, date_input.month, last_day_of_month)
```

** Get start & end of quarter from a given date **

```py
import bisect
import calendar

from datetime import date, datetime, timedelta

def get_quarter_end(date_input: date):
	quarter_ends = [date(date_input.year, month, 1) + timedelta(days=-1) for month in (4, 7, 10)]
	quarter_ends.append(date(date_input.year + 1, 1, 1) + timedelta(days=-1))
	idx = bisect.bisect_left(quarter_ends, date_input)
	return quarter_ends[idx]


def get_quarter_start(date_input: date):
	quarter_start = [date(date_input.year, month, 1) for month in (1, 4, 7, 10)]
	idx = bisect.bisect(quarter_start, date_input)
	return quarter_start[idx - 1]
```

## User authorisation/permission

An tiny user authorisation/permission python/flask class inspired by Ruby's authorization library [Pundit](https://github.com/varvet/pundit)

```py
import inspect
import re

from flask import g, request
from path.to.user.model import User


@staticmethod
def to_snake_case(obj_str):
	reg_match = re.compile("((?<=[a-z0-9])[A-Z]|(?!^)(?<!_)[A-Z](?=[a-z]))")
	return reg_match.sub(r"_\1", obj_str).lower()


class UserAuth:
	def __init__(self):
		self._current_user = None

	@property
	def current_user(self):
		if self._current_user is None:
			# can replace with g.current_user or method to get current user
			self._current_user = User.get(g.user_id)

		return self._current_user

	@classmethod
	def get_policy_cls(cls, model_cls):
		model_cls_name = model_cls.__name__
		model_name_snake_case = to_snake_case(model_cls_name)
		policy_path = f"path.to.policies.{model_name_snake_case}_policy"
		policy_module = __import__(policy_path, fromlist=[f"{model_cls_name}Policy"])
		policy_cls = getattr(policy_module, f"{model_cls_name}Policy")

		return policy_cls

	@classmethod
	def authorised_action(cls, model_obj, action=None, *args, **kwargs):
		model_cls = cls.get_model_class(model_obj)
		model_policy = cls.get_policy_cls(model_cls)
		action = action or request.method.lower()
		return getattr(model_policy(self.current_user, model_obj), action)(*args, **kwargs)

	@classmethod
	def authorised_scope(cls, model_obj, *args, **kwargs):
		model_cls = cls.get_model_class(model_obj)
		model_policy = cls.get_policy_cls(model_cls)
		return getattr(model_policy(self.current_user, model_cls), "scope")(*args, **kwargs)

	@classmethod
	def get_model_class(cls, model_obj):
		if inspect.isclass(model_obj):
			return model_obj

		return model_obj.__class__
```

### How it works

Assuming we have a `User` class similar to the example below (with SQLAlchemy):

```py
# Path to SQLalchemy/Database config
from app.setting.database import db

class UserRole(enum.Enum):
	REGISTER = 1
	ADMIN = 2


class User(db.Model):
	# for dataclass, can do this instead of `__init__`
	# id: uuid.UUID
	# email: str
	# first_name: str
	# role: UserRole = UserRole.REGISTER

	def __init__(self, email: str, name: str, role: int = UserRole.REGISTER) -> None:
		self.email = email
		self.name = name
		self.role = role

	@classmethod
	def create(cls, email: str, name: str, role: int = UserRole.REGISTER) -> "User":
		user = cls(email=email, name=name, role=role)
		db.session.add(user)
		db.session.commit()
		return user

	@classmethod
	def get(cls, id: str) -> "User":
		return cls.query.get(id)

	@classmethod
	def all(cls) -> list["User"]:
		return cls.query.all()

	def update(self, name: str, email:str) -> None:
		self.name = name
		self.email = email
		db.session.commit()

	@property
	def is_admin(self) -> bool:
		return self.role == UserRole.ADMIN
```

And `UserPolicy` - assuming only admin users can create, read, update and view all users, while registered users can only view their only user accounts:

```py
class UserPolicy:
	def __init__(self, user: User, user_obj: User) -> None:
		self.user = user
		self.user_obj = user_obj

	def create(self) -> bool:
		return self.user.is_admin:

	def update(self) -> bool:
		return self.user.is_admin:

	def get(self) -> bool:
		return (self.user.id == self.user.id) or self.user.is_admin

	def scope(self) -> list[User]:
		if self.user.is_admin:
			return self.user_obj.all()

		return [self.user_obj.get(self.user.id)]

```

(Optional) And a custom exception class when current user not authorised to perform particular action(s) defined in the policy:

```py
class PermissionRequired(BaseError):
	def __init__(self):
		super().__init__(403, "Permission denied")
```

To check if the current user is authorised to perform a particular `User` action - in this example update, in a routing/view:

```py
from path.to.user_auth import UserAuth
from path.to.user.model import User

@app.post('/users/<user_id>')
def update(user_id):
	user = User.get(user_id)
	if UserAuth.authorize(user, "update"):
		return user

	return render_template("403.html"), 403


@app.post('/users')
def all():
	user_records = UserAuth.authorised_scope(User)
	return jsonify(user_records)

```

or in a service class:

```py
from path.to.user_auth import UserAuth
from path.to.user.model import User

class UsersService:
	def update(self, user_id: str, name: str, email: str) -> dict[str, str]:
		user = User.get(user_id)
		if not UserAuth.authorised_action(user, "update"):
			raise PermissionRequired()

		user.update(name=name, email=email)
		return {"message": "User successfully update"}

```

The `UserAuth.authorize` takes two parameters:

- the model object (can be an instance or a class) you want to authorise the current user on
- the action or class method to authorise - in this case `update`. By default, if nothing passes as the parameter it will use the `request.method`, e.g. `post` in the example, and will require to define a `post` method in the policy:

```py
class UserPolicy:
	...

	def post(self) -> bool:
		return self.user.is_admin:

	...
```

## Regex

### Remove file size from string

```py
def remove_filesize_str(txt: str) -> str:
		if not txt:
				return txt

		patt = r"(^|\s|\,|\:|\/|\(|\[)?\s*\d+[\.|\,]?\d*\s*((G|T|M|K)((?!G|T|M|K)B|O)|bytes)\s*(?=$|\s|\)|\]|(?!\w|\d))(\s|\)|\])?"
		cleaned_txt = re.sub(patt, " ", txt, flags=re.IGNORECASE)
		cleaned_txt = re.sub(r"\s\s*", " ", cleaned_txt).strip()
		return cleaned_txt

```
