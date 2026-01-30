# Python Review Rules

## Idiomatic Patterns

- Use list/dict/set comprehensions over `map`/`filter` with lambdas for readability.
- Use `pathlib.Path` over `os.path` for file system operations.
- Use f-strings for string formatting. Avoid `%` formatting and `.format()` in new code.
- Use `dataclasses` or `attrs` for data containers; avoid plain dicts for structured data.
- Use `typing` annotations on all public function signatures. Use `from __future__ import annotations` for forward references.
- Prefer `enum.Enum` over string constants for fixed option sets.
- Use `contextlib.contextmanager` for simple resource management, not full class-based context managers.
- Use `collections.defaultdict`, `Counter`, `deque` instead of hand-rolling equivalents.

## Common Anti-Patterns

- Flag: mutable default arguments (`def f(items=[])`). Use `None` and initialize inside the function.
- Flag: bare `except:` or `except Exception:` without re-raise or explicit handling rationale.
- Flag: `import *` in any non-`__init__.py` module.
- Flag: global mutable state modified at module level during import.
- Flag: nested functions deeper than 2 levels; refactor into named functions or classes.
- Flag: `type()` for type checking instead of `isinstance()`.
- Flag: string-based type dispatch (`if obj.type == "foo"`) instead of polymorphism or match/case.
- Flag: `os.system()` or `subprocess.call(shell=True)` for command execution.

## Error Handling

- Catch specific exceptions, never bare `except:`.
- Use custom exception hierarchies inheriting from domain-specific base exceptions.
- Use `raise ... from err` to preserve exception chains.
- Never silently swallow exceptions. At minimum, log at warning level.
- Use `contextlib.suppress(SpecificError)` for intentionally ignored exceptions, not try/except/pass.
- Validate preconditions early with `ValueError`/`TypeError`; do not return `None` for error cases.

## Naming Conventions

- `snake_case` for functions, methods, variables, modules. `PascalCase` for classes.
- `UPPER_SNAKE_CASE` for module-level constants.
- Private attributes with single underscore prefix: `_internal`. Avoid double underscore name mangling.
- Boolean variables and functions: use `is_`, `has_`, `can_`, `should_` prefixes.
- Avoid abbreviations in names. `calculate_total_price` not `calc_tot_prc`.
- Module names should be short, lowercase, no hyphens. Use underscores sparingly.

## Testing Patterns

- Use `pytest` as the standard test runner. Avoid `unittest.TestCase` subclassing in new code.
- Use fixtures with `@pytest.fixture` for setup/teardown, not `setUp`/`tearDown` methods.
- Use `pytest.mark.parametrize` for table-driven tests.
- Use `pytest.raises(SpecificError)` as a context manager for exception testing.
- Mock external dependencies with `unittest.mock.patch`, but prefer dependency injection.
- Use `factory_boy` or `faker` for test data generation over hand-crafted fixtures.
- Separate unit tests from integration tests with markers: `@pytest.mark.integration`.
- Use `tmp_path` fixture for temporary file operations in tests.

## Concurrency

- Use `asyncio` for I/O-bound concurrency. Use `concurrent.futures.ProcessPoolExecutor` for CPU-bound work.
- Never mix `asyncio` and threads without explicit bridge functions (`asyncio.to_thread`, `loop.run_in_executor`).
- The GIL makes threads useless for CPU parallelism; use multiprocessing or subprocesses.
- Use `async with` and `async for` for async context managers and iterators.
- Flag: creating `asyncio.get_event_loop()` manually; use `asyncio.run()` as the entry point.
- Flag: blocking calls (`time.sleep`, `requests.get`) inside async functions.
- Use `asyncio.TaskGroup` (3.11+) for structured concurrency over `gather`.

## Package/Module Structure

- Use `src/` layout with `pyproject.toml` for installable packages.
- One public class or closely related group per module. Avoid monolithic 1000+ line modules.
- `__init__.py` should re-export public API; keep it thin.
- Use `py.typed` marker and inline type annotations for typed packages.
- Pin dependencies in `requirements.txt` or `poetry.lock`; use ranges in `pyproject.toml`.
- Separate dev dependencies from production dependencies.

## Performance Gotchas

- Avoid repeated string concatenation in loops; use `str.join()` or `io.StringIO`.
- Use generators and `itertools` for large data pipelines instead of materializing lists.
- `in` on a `list` is O(n); use a `set` or `dict` for membership testing.
- Avoid creating large intermediate lists; use generator expressions.
- Use `__slots__` on data-heavy classes to reduce memory overhead.
- Profile with `cProfile` or `py-spy`, not intuition. Use `timeit` for micro-benchmarks.
- Prefer `orjson` or `msgspec` over `json` stdlib for hot-path serialization.
- Attribute access in tight loops is slow; localize frequently accessed attributes.
