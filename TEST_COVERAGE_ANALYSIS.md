# Test Coverage Analysis

## Current State

The codebase has **virtually no meaningful test coverage**. Only two placeholder test files exist — both are Laravel scaffolding defaults:

| File | Tests | What it does |
|------|-------|-------------|
| `tests/Unit/ExampleTest.php` | 1 | `assertTrue(true)` — no-op |
| `tests/Feature/ExampleTest.php` | 1 | Asserts `/` contains "The Bootstrap Blog" |

**Estimated line coverage: < 5%**

The PHPUnit config (`phpunit.xml`) is properly set up with separate Unit and Feature suites, and the `./app` directory is in the coverage whitelist. Mockery is installed but unused. A `User` model factory exists but no factories for `Post` or `Comment`.

---

## Recommended Test Improvements

### Priority 1 — Model Unit Tests (High Impact, Low Effort)

These are pure unit tests for Eloquent models that exercise business logic and relationships.

**1.1 `Post` model tests** (`app/Post.php`)
- Test `comments()` relationship returns a HasMany relation
- Test `user()` relationship returns a BelongsTo relation
- Test `addComment($body)` creates a comment associated with the post
- Test `$fillable` allows only `user_id`, `title`, `body`

**1.2 `User` model tests** (`app/User.php`)
- Test `posts()` relationship returns a HasMany relation
- Test `publish(Post $post)` saves the post under the user
- Test `$fillable` allows only `name`, `email`, `password`
- Test `$hidden` hides `password` and `remember_token`

**1.3 `comment` model tests** (`app/comment.php`)
- Test `post()` relationship returns a BelongsTo relation
- Test `user()` relationship returns a BelongsTo relation
- Test `$fillable` allows only `id`, `user_id`, `post_id`, `body`

### Priority 2 — Controller Feature Tests (High Impact, Medium Effort)

These are HTTP-level integration tests that exercise routes, middleware, validation, and database interactions.

**2.1 `PostsController` tests** (`app/Http/Controllers/PostsController.php`)
- `GET /index` — returns 200, displays posts
- `GET /posts/{id}` — returns 200 for valid post, 404 for missing
- `GET /create` — accessible without auth (per middleware config)
- `POST /posts` — **authenticated** user can create a post
- `POST /posts` — unauthenticated user gets redirected (auth middleware)
- `POST /posts` — validation rejects missing `title` or `body`
- `POST /posts` — created post is associated with the authenticated user

**2.2 `CommentsController` tests** (`app/Http/Controllers/CommentsController.php`)
- `POST /posts/{post}/comments` — creates a comment on the post
- `POST /posts/{post}/comments` — validation rejects empty or too-short body (`min:2`)
- `POST /posts/{post}/comments` — returns redirect back

**2.3 `RegistrationController` tests** (`app/Http/Controllers/RegistrationController.php`)
- `GET /register` — returns 200 with registration form
- `POST /register` — creates user and logs them in
- `POST /register` — validation rejects missing name, invalid email, missing password
- `POST /register` — **security concern**: password is stored as plaintext (no `bcrypt`). A test should verify this and flag the bug.

**2.4 `SessionsController` tests** (`app/Http/Controllers/SessionsController.php`)
- `GET /login` — returns login form
- `POST /login` — valid credentials redirect to home
- `POST /login` — invalid credentials return errors
- `GET /logout` — logs user out
- Guest middleware blocks authenticated users from login/register pages

### Priority 3 — Missing Model Factories (Test Infrastructure)

Only a `User` factory exists. To write effective tests, add factories for:

- **`Post` factory** — generate fake title and body, associate with a user
- **`Comment` factory** — generate fake body, associate with a post and user

### Priority 4 — Security & Edge Case Tests

**4.1 Password hashing**
`RegistrationController@store` calls `User::create(request(['name','email','password']))` without hashing the password. This is a **bug** — the password is stored in plaintext. A test should verify that stored passwords are hashed.

**4.2 Mass assignment protection**
- Verify that models only allow expected fields via `$fillable`
- Attempt to set `user_id` on a post via the create endpoint (should be ignored, set from `auth()->id()`)

**4.3 Authorization**
- Verify users cannot modify/delete other users' posts (no update/delete actions exist yet — this is a feature gap)
- Verify the `auth` middleware on `PostsController` protects `store` and `show` but not `index` and `create`

**4.4 Input validation edge cases**
- XSS payloads in post title/body/comment body
- Very long strings
- Empty strings vs. whitespace-only strings

### Priority 5 — Route Tests

Verify that all routes in `routes/web.php` resolve to the correct controller and method:
- `GET /` renders the master layout
- All CRUD routes for posts and comments
- Auth routes (login, register, logout)

---

## Suggested File Structure

```
tests/
├── Feature/
│   ├── PostsControllerTest.php
│   ├── CommentsControllerTest.php
│   ├── RegistrationControllerTest.php
│   ├── SessionsControllerTest.php
│   └── RoutesTest.php
├── Unit/
│   ├── PostTest.php
│   ├── UserTest.php
│   └── CommentTest.php
└── ...
```

## Bugs Discovered During Analysis

1. **Plaintext passwords**: `RegistrationController@store` does not hash passwords before storing them.
2. **Middleware typo**: `SessionsController` uses `'expect'` instead of `'except'` — the guest middleware will not correctly exclude the `destroy` method.
3. **No CSRF or auth on comments**: `CommentsController` has no auth middleware, meaning unauthenticated users can post comments (may be intentional, but should be a conscious decision verified by tests).
4. **No update/delete functionality**: Posts and comments have no edit or delete endpoints — a potential feature gap.
