# One use case, two application boundaries

Use this example when explaining the pattern. It is a small member-name lookup, not a mandatory project layout. The CLI and an automated test both drive the same input API. The application calls a directory port without knowing its storage technology.

```text
Runtime calls: CLI / test → GetMemberName.execute → MemberDirectory.find_by_id
Source imports: cli → core; adapters → core; core → standard library only
Wiring: CLI startup constructs the directory and injects it into the use case
```

Save the following four Python blocks under their indicated filenames in one empty directory. Python 3.10+ is sufficient; no third-party package is required.

## core.py — application API and outbound contract

```python
from dataclasses import dataclass
from typing import Protocol


@dataclass(frozen=True)
class Member:
    id: str
    display_name: str


class MemberDirectory(Protocol):
    def find_by_id(self, member_id: str) -> Member | None: ...


class MemberNotFound(LookupError):
    pass


class GetMemberName:
    def __init__(self, directory: MemberDirectory):
        self.directory = directory

    def execute(self, member_id: str) -> str:
        if not member_id.strip():
            raise ValueError("Member ID is required")
        member = self.directory.find_by_id(member_id)
        if member is None:
            raise MemberNotFound(member_id)
        return member.display_name
```

`execute` is the input port. `MemberDirectory` is the output port, owned by the application. `Member` is application-owned data and can cross this boundary without another DTO. No input interface or entity base class is needed.

## adapters.py — replaceable outbound implementation

```python
from collections.abc import Iterable

from core import Member


class InMemoryMemberDirectory:
    def __init__(self, members: Iterable[Member] = ()):
        self.members = {member.id: member for member in members}

    def find_by_id(self, member_id: str) -> Member | None:
        return self.members.get(member_id)
```

Python's structural typing means the adapter satisfies the protocol without inheriting from it. A SQL adapter would implement the same operation, map database rows to `Member`, return `None` for absence, and translate relevant vendor failures to application-owned outcomes. It would need its own integration checks.

## cli.py — inbound adapter and composition root

```python
import sys
from collections.abc import Sequence

from adapters import InMemoryMemberDirectory
from core import GetMemberName, Member, MemberNotFound


def main(args: Sequence[str], use_case: GetMemberName) -> int:
    if len(args) != 1:
        print("Usage: python3 cli.py MEMBER_ID", file=sys.stderr)
        return 2
    try:
        print(use_case.execute(args[0]))
        return 0
    except MemberNotFound:
        print("Member not found", file=sys.stderr)
        return 1
    except ValueError as error:
        print(str(error), file=sys.stderr)
        return 2


if __name__ == "__main__":
    directory = InMemoryMemberDirectory([Member("member-1", "Nathan")])
    raise SystemExit(main(sys.argv[1:], GetMemberName(directory)))
```

This small file houses both CLI translation and startup wiring. That is sufficient for this example; a larger application's startup can have its own module. The use case never imports the CLI, its exit codes, or the concrete directory.

## test_architecture.py — application outcomes and adapter mapping

```python
import unittest
from contextlib import redirect_stderr, redirect_stdout
from io import StringIO

from adapters import InMemoryMemberDirectory
from cli import main
from core import GetMemberName, Member, MemberNotFound


class GetMemberNameTests(unittest.TestCase):
    def test_returns_member_name_without_external_services(self):
        directory = InMemoryMemberDirectory([Member("member-1", "Nathan")])
        self.assertEqual(GetMemberName(directory).execute("member-1"), "Nathan")

    def test_reports_missing_member(self):
        with self.assertRaises(MemberNotFound):
            GetMemberName(InMemoryMemberDirectory()).execute("missing")

    def test_rejects_empty_id_before_accessing_directory(self):
        class UnavailableDirectory:
            def find_by_id(self, member_id: str) -> Member | None:
                raise AssertionError("Invalid ID must not reach the directory")

        with self.assertRaises(ValueError):
            GetMemberName(UnavailableDirectory()).execute("  ")

    def test_cli_maps_success(self):
        use_case = GetMemberName(
            InMemoryMemberDirectory([Member("member-1", "Nathan")])
        )
        output = StringIO()
        with redirect_stdout(output):
            self.assertEqual(main(["member-1"], use_case), 0)
        self.assertEqual(output.getvalue(), "Nathan\n")

    def test_cli_maps_missing_member(self):
        output = StringIO()
        with redirect_stderr(output):
            self.assertEqual(main(["missing"], GetMemberName(InMemoryMemberDirectory())), 1)
        self.assertEqual(output.getvalue(), "Member not found\n")

    def test_cli_maps_invalid_invocations(self):
        use_case = GetMemberName(InMemoryMemberDirectory())
        for args in ([], ["one", "two"], ["  "]):
            with self.subTest(args=args), redirect_stderr(StringIO()):
                self.assertEqual(main(args, use_case), 2)


if __name__ == "__main__":
    unittest.main()
```

Run:

```bash
python3 -m unittest -v
python3 cli.py member-1
```

The tests establish application outcomes and CLI mappings. They do not establish real database behavior, network failure handling, persistence, or production authentication. Keep the in-memory implementation as a demo/test adapter; test a production adapter against its actual technology and preserve the same port contract.
