# Correct Sanic Request Form Type Annotations

## Issue

This change addresses [Sanic issue #3192](https://github.com/sanic-org/sanic/issues/3192): `Request.form` and `Request.get_form()` were annotated as returning `RequestParameters | None`, although their runtime behavior initializes and returns a `RequestParameters` object.

## Why This Issue Was Happening

`Request.parsed_form` starts as `None` because form parsing is lazy. When callers access `request.form`, Sanic calls `get_form()` if parsing has not happened yet. `get_form()` immediately initializes `parsed_form` with an empty `RequestParameters` instance and replaces it with parsed values when the request contains form data.

Therefore, the internal lazy state is optional, but the public values returned by `form` and `get_form()` are not optional. The public annotations exposed the internal initialization state instead of the public runtime contract.

This caused valid code such as the following to produce an unnecessary mypy error:

```python
user_project_id = int(request.form.get("user_project_pk"))
```

## What We Changed

- Changed `Request.get_form()` to return `RequestParameters`.
- Changed the `Request.form` property to return `RequestParameters`.
- Added an explicit narrowing assertion after lazy parsing so the implementation
        also satisfies the non-optional public return type.
- Updated the corresponding docstrings.
- Added runtime assertions to the existing populated and empty-form tests.
- Added a changelog entry for issue #3192.

The implementation behavior is unchanged. This is an annotation correction, not a parsing algorithm change.

## Approach

1. Located the public request annotations in `sanic/request/types.py`.
2. Traced initialization and all normal return paths for `parsed_form`.
3. Confirmed that `get_form()` initializes an empty `RequestParameters` before parsing.
4. Confirmed that `form` forces parsing before returning.
5. Reused existing request-form tests instead of introducing a second test harness.
6. Changed only the public return annotations, documentation, focused test assertions, and changelog.

## Complexity

There is no algorithmic change.

- **Time complexity:** unchanged.
- **Space complexity:** unchanged.
- **Runtime behavior:** unchanged.
- **API impact:** improves the static type contract while preserving the existing runtime API.

## Tests Added or Updated

The existing form tests now explicitly verify that:

- A populated URL-encoded form returns `RequestParameters`.
- An invalid or absent form content type returns an empty `RequestParameters` object.

The existing suite also continues to cover URL-encoded, multipart, ASGI, blank-value, and multi-value form handling.

## Verification

Executed locally on Windows with Python 3.13:

```text
30 passed, 105 deselected
```

Command:

```powershell
python -m pytest tests/test_requests.py -k form -q
```

Targeted mypy check:

```text
passed with no output
```

Lint and whitespace checks:

```text
ruff check: All checks passed
git diff --check: passed
```

The focused test run reports existing Sanic warnings, including deprecation warnings and a pending-task warning after test completion. These warnings did not cause test failures and are outside this annotation-only change.

## Files Changed

- `sanic/request/types.py`
- `tests/test_requests.py`
- `changelogs/3192.bugfix.rst`

## Pull Request Status

The work is prepared on branch `fix/request-form-return-type` in the contributor fork:

```text
https://github.com/sachinbisht1/3192-sanic.git
```

The branch was initially committed as `623dc716` and pushed to the fork. The follow-up correction is being committed on the same branch. The pull request target is:

```text
sachinbisht1/3192-sanic:fix/request-form-return-type
        -> sanic-org/sanic:main
```

After review, any requested changes should be committed to the same branch and pushed. GitHub will update the existing pull request automatically.