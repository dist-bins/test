# Validated Python Data Modeling with Pydantic

**Purpose:** help coding agents design Python data models that accurately represent the domain, validate the inputs they actually receive, preserve the guarantees they promise, and serialize intentionally.

**Audience:** coding agents and reviewers working with modern Python and Pydantic v2.

**Revision:** 2026-10-05. Consolidated from the earlier Python/Rust modeling spec, the DeepWiki review against the Pydantic repository, and the revised v3 modeling guide.

**Scope:** data shape, runtime validation, constraints, variants, updates, serialization, schema, and consumer-side typing. This is not an application architecture, persistence, deployment, or framework guide.

**Design principle:** choose the smallest clear model that expresses the real contract. Do not optimize for resemblance to Rust, for minimizing the number of classes, or for using the maximum number of Pydantic features.

The words **MUST**, **SHOULD**, and **MAY** are recommendations of this guide. They are not claims about Pydantic defaults.

---

# 0. How to use this guide [ALWAYS READ]

When a task creates, changes, or reviews Python data models, validation, parsing, update behavior, or serialization:

1. Read **Part I: Core modeling rules**.
2. Read the **Topic trigger index** in section 11.
3. Follow every reference triggered by the actual contract, even when the model looks simple or the API seems familiar.
4. Inspect the repository's Python version, Pydantic version, dependency constraints, existing domain types, tests, and configured type checker before selecting syntax.
5. Do not upgrade dependencies or introduce an advanced feature merely because this guide documents it.
6. Validate the implementation through the same boundary the application will use.

The distinction between core and reference material is about **frequency**, not difficulty. A two-field model can require an advanced serialization rule. A large record can require only core features.

## 0.1 Project integration snippet

Keep this guide as reusable documentation. In each project's `AGENTS.md`, or equivalent project instructions, add only a short reference such as:

```markdown
## Python data modeling

When creating, modifying, or reviewing Python data models, validation,
parsing, update behavior, or serialization, read
`<path>/PYDANTIC_MODELING_GUIDE.md` before implementing the change.

Always read the core modeling sections and the topic trigger index.
Then read every referenced section relevant to the contract, regardless
of whether the model appears simple or you already know the Pydantic API.

Follow the Python and Pydantic versions and conventions of this repository.
Do not upgrade dependencies or introduce advanced features merely because
those features appear in the guide.

Validate the implementation with the project's configured tests and static
type checker where applicable.
```

Project-specific instructions override general preferences in this guide when they deliberately define a different contract or convention.

## 0.2 Separate three kinds of safety

Do not treat these as interchangeable:

- **Representation safety:** which states the type shape can express.
- **Boundary validation:** which external values a validation call accepts and what it produces.
- **Lifetime and consumer safety:** what can happen after construction during mutation, reuse, serialization, and branching.

For example, a discriminated union describes alternative shapes, `TypeAdapter` validates raw input against that union, and `assert_never` plus a configured type checker helps consumers handle every variant. No one mechanism provides all three guarantees.

---

# Part I. Core modeling rules [ALWAYS READ]

# 1. Compatibility and evidence [ALWAYS READ]

Before using an API, inspect the versions installed by the project. Do not infer support from memory or from another repository.

This guide targets Pydantic v2. Some documented features are version-sensitive:

| Feature | Compatibility note |
|---|---|
| `StrEnum`, `Self`, `assert_never` | Standard library on Python 3.11+. |
| `TypeIs`, `ReadOnly` | Standard library on Python 3.13+; `typing_extensions` provides compatible backports for older supported targets. |
| `TypedDict(closed=..., extra_items=...)` | Pydantic supports PEP 728 from v2.12. Use `typing_extensions.TypedDict` where the standard library does not support these class keywords. |
| `validate_by_name`, `validate_by_alias`, `serialize_by_alias` | Pydantic v2.11+. Input and output naming remain separate decisions. |
| `polymorphic_serialization` | Pydantic v2.13+. This affects serialization, not subtype selection during validation. |
| `pydantic.experimental.*` | Experimental. Use only for a concrete requirement and after checking the installed version. |

Runtime acceptance does not prove that the project's static type checker understands the same annotation. Test both when static behavior matters.

The executable anchor examples and version-sensitive examples in this revision were smoke-tested on Python 3.13.5 with Pydantic 2.13.4. This is evidence for those concrete versions, not a compatibility matrix.

If an API is version-sensitive or unfamiliar, consult the version-matched official documentation rather than guessing a parameter name or copying an example from a newer project.

---

# 2. Establish the contract before modeling [ALWAYS READ]

Before writing code, determine:

- the input representation: JSON text, decoded Python values, typed application objects, or existing model instances;
- required fields;
- nullable fields;
- defaults and what omission means;
- accepted coercions and normalizations;
- unknown-key policy;
- domain constraints and cross-field relationships;
- supported mutation or update operations;
- expected Python representation after validation;
- serialized output shape, aliases, exclusions, and secret handling;
- whether consumers need exhaustive handling of variants;
- external facts that cannot be proven by the model itself.

Do not invent ID formats, limits, units, default values, state transitions, timezones, ownership rules, or normalization rules when the contract does not specify them.

## 2.1 Classify each rule

Classify every nontrivial rule before choosing a Pydantic mechanism:

| Rule kind | Typical mechanism |
|---|---|
| Record shape | `BaseModel` or, when a dictionary result is required, `TypedDict` + `TypeAdapter` |
| Closed vocabulary | `Enum` / `StrEnum`; `Literal` for discriminator tags or genuinely local literal values |
| Scalar constraint | Built-in Pydantic type or `Annotated` constraint |
| Alternative shapes | Discriminated union |
| Relationship inside one shape | `model_validator(mode="after")` |
| Input normalization | Explicit parser or a carefully scoped before validator |
| State transition | Explicit operation that constructs a checked next state |
| External fact | Application/service check outside pure model validation |

Do not force all rules into validators. Do not force all relationships into unions. Do not use a union simply because several fields are optional.

---

# 3. Choose the representation and boundary [ALWAYS READ]

Use the representation that matches the value callers actually need.

| Requirement | Default choice | Choose something else when |
|---|---|---|
| Named structured application data | `BaseModel` | The result must remain a dictionary or an existing dataclass contract is important. |
| Dictionary-shaped validated result | `TypedDict` + reusable `TypeAdapter` | Model methods, model configuration, or object identity are useful. |
| Standalone scalar, collection, alias, or union | Type alias + reusable `TypeAdapter` | A named runtime wrapper is part of the domain. |
| Named root value with model API | `RootModel[T]` | Validation/schema alone is enough, in which case use `TypeAdapter(T)`. |
| Existing dataclass semantics | Pydantic dataclass or standard dataclass with an explicit validation boundary | A normal `BaseModel` is simpler. |

A type annotation is not a validation call. `cast(...)`, a `TypedDict` declaration, or a variable annotation does not validate runtime data.

## 3.1 Core example: a closed validated record

Use this pattern when the contract is an object with known keys and a clear distinction between required, nullable, and defaulted fields.

```python
from typing import Annotated

from pydantic import BaseModel, ConfigDict, Field, NonNegativeInt

PositiveUnits = Annotated[int, Field(strict=True, gt=0)]


class Allocation(BaseModel):
    model_config = ConfigDict(
        extra="forbid",
        frozen=True,
        validate_default=True,
    )

    units: PositiveUnits
    note: str | None
    retries: NonNegativeInt = 0
```

Expected behavior for this example:

```python
from pydantic import ValidationError

allocation = Allocation.model_validate_json('{"units":2,"note":null}')
assert allocation.units == 2
assert allocation.note is None
assert allocation.retries == 0

try:
    Allocation.model_validate_json('{"units":"2","note":null}')
except ValidationError:
    pass
else:
    raise AssertionError("string units must be rejected")

try:
    Allocation.model_validate_json('{"units":2}')
except ValidationError:
    pass
else:
    raise AssertionError("note is required even though null is allowed")

try:
    Allocation.model_validate_json(
        '{"units":2,"note":null,"surprise":true}'
    )
except ValidationError:
    pass
else:
    raise AssertionError("unknown keys must be rejected")
```

Do not copy `frozen=True`, `extra="forbid"`, strictness, or these defaults into unrelated models without a contract that requires them.

---

# 4. Presence, nullability, defaults, and patches [ALWAYS READ]

Presence and nullability are independent.

| Declaration | May be omitted? | May be `None`? |
|---|---:|---:|
| `name: str` | No | No |
| `nickname: str | None` | No | Yes |
| `bio: str | None = None` | Yes | Yes |
| `retries: int = 3` | Yes | No |

Pydantic v2 does not infer a default of `None` from `T | None`.

Defaults are not automatically validated in every configuration. If a default contributes to correctness, enable `validate_default=True` on the field or model and test it.

## 4.1 Patches are not full models with everything optional

A partial update has at least three possible meanings for a field:

- omitted: leave unchanged;
- supplied with a value: replace;
- supplied as `None`: explicitly clear, when null is legal.

Do not erase this distinction with `exclude_none=True`.

When a dictionary-shaped patch is appropriate:

```python
from typing_extensions import NotRequired, TypedDict


class ProfilePatch(TypedDict, closed=True):
    nickname: NotRequired[str | None]
```

A dedicated patch `BaseModel` plus `model_fields_set` / `exclude_unset=True` is another valid pattern.

The patch itself may be structurally valid while the resulting object is not. After applying a patch, validate the complete proposed state when cross-field or domain invariants can be affected.

Do not use `model_copy(update=...)` as if it validates untrusted patch values. It does not.

---

# 5. Value domains and scalar constraints [ALWAYS READ]

## 5.1 Closed vocabularies

Use an enum when a concept has a reusable closed vocabulary.

```python
from enum import StrEnum

from pydantic import BaseModel, ConfigDict


class TaskStatus(StrEnum):
    PENDING = "pending"
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"


class Task(BaseModel):
    model_config = ConfigDict(extra="forbid")
    status: TaskStatus
```

`class Status(str, Enum)` remains valid. `StrEnum` is simply the modern standard-library form when the Python target supports it.

Use `Literal[...]` when the literal identity itself is needed by the type system, especially for discriminated-union tags. A project-wide preference for enums does not remove this Pydantic requirement.

Do not use a free-form `str` when arbitrary strings are not legal.

## 5.2 Scalar constraints

Prefer built-in constrained/domain types or `Annotated` metadata before writing a custom validator.

```python
from typing import Annotated

from pydantic import Field, TypeAdapter

RetryCount = Annotated[int, Field(strict=True, ge=0)]
Percentage = Annotated[int, Field(strict=True, ge=0, le=100)]

retry_count_adapter = TypeAdapter(RetryCount)
percentage_adapter = TypeAdapter(Percentage)
```

Keep dimensions separate:

- `ge=0` means non-negative, not positive;
- strictness controls coercion, not numeric range;
- `min_length=1` means nonempty, not nonblank;
- trimming input is a transformation and must be intentional;
- finiteness is separate from ordinary floating-point type validation.

A constrained scalar alias is normally still the underlying scalar to the static type checker. Arithmetic does not preserve a proof that the result satisfies the constraint. Revalidate values that cross a trusted boundary again.

Do not invent business constraints merely because they are easy to encode.

---

# 6. Variants versus cross-field rules [ALWAYS READ]

The most important design question is whether invalid combinations represent **different cases** or a **relationship inside one case**.

## 6.1 Different payload shapes: use a discriminated union

Use a tagged union when one value is one of several cases and each case owns different fields.

```python
from typing import Annotated, Literal, assert_never

from pydantic import BaseModel, ConfigDict, Field, TypeAdapter
NonEmptyText = Annotated[str, Field(min_length=1)]


class PaymentSuccess(BaseModel):
    model_config = ConfigDict(extra="forbid", frozen=True)
    status: Literal["success"]
    receipt_id: NonEmptyText


class PaymentFailure(BaseModel):
    model_config = ConfigDict(extra="forbid", frozen=True)
    status: Literal["failure"]
    error_message: NonEmptyText


PaymentResult = Annotated[
    PaymentSuccess | PaymentFailure,
    Field(discriminator="status"),
]
payment_result_adapter = TypeAdapter(PaymentResult)


def render_result(result: PaymentResult) -> str:
    if isinstance(result, PaymentSuccess):
        return result.receipt_id
    if isinstance(result, PaymentFailure):
        return result.error_message
    assert_never(result)
```

This removes impossible combinations such as success with only an error or failure with only a receipt.

Do not model mutually exclusive cases as one broad model with many optional fields unless the wire contract forces that shape and a union is genuinely unavailable.

Do not use a union for independent optional attributes.

## 6.2 One shape with a relational invariant: use a model validator

A relationship such as `end >= start` does not create two domain variants.

```python
from typing import Self

from pydantic import AwareDatetime, BaseModel, ConfigDict, model_validator


class TimeWindow(BaseModel):
    model_config = ConfigDict(extra="forbid", frozen=True)

    start: AwareDatetime
    end: AwareDatetime

    @model_validator(mode="after")
    def check_order(self) -> Self:
        if self.end < self.start:
            raise ValueError("end must not precede start")
        return self
```

Use a model validator for a real relationship inside one coherent shape. It is not an inferior fallback to a union.

## 6.3 State shape is not transition correctness

A union such as `Pending | Running | Completed | Failed` can make snapshots precise. It does not automatically enforce legal transitions between snapshots, prove that an external action happened, or enforce authorization.

When transition rules matter, implement explicit checked operations that produce the next validated state.

---

# 7. Validation boundaries, extras, and strictness [ALWAYS READ]

## 7.1 Validate at the real boundary

Use the API matching the actual input representation:

- JSON text into a model: `Model.model_validate_json(...)`;
- decoded Python values into a model: `Model.model_validate(...)`;
- standalone alias/union/collection/dictionary shape: `TypeAdapter.validate_json(...)` or `validate_python(...)`.

Reuse `TypeAdapter` instances instead of rebuilding them repeatedly in hot paths.

Do not parse JSON into Python first merely to force every entry point through `model_validate` if strict behavior or JSON-specific parsing semantics matter. Strict Python and strict JSON validation are not identical for every supported type.

## 7.2 Unknown keys are a contract decision

For a closed object contract, `extra="forbid"` is usually the clearest choice.

For a tolerant reader, deliberate ignoring can be correct.

For an extensible contract, retain extras deliberately and give their values a type when possible. See section 12.2.

A parent model's configuration does not automatically reconfigure every nested model class. Apply policies to the types that own them.

## 7.3 Strictness is selective

Strict mode is useful when coercion would hide an error, for example accepting `"3"` for an integer count.

Do not enable strict mode universally without considering the boundary. Some domains intentionally parse strings into dates, UUIDs, URLs, or other semantic types.

Prefer an explicit parser or normalization rule when conversion is a business requirement. Test the exact input forms the application accepts and rejects.

---

# 8. Preserve invariants after construction [ALWAYS READ]

A model can be valid at construction and become invalid later. Decide which operations are supported.

## 8.1 Prefer immutable snapshots when they fit the domain

`ConfigDict(frozen=True)` prevents ordinary field reassignment on a model, but the protection is shallow. A list or dictionary stored inside a frozen model can still mutate unless the contained representation is also immutable.

Use tuples, frozen nested models, immutable value objects, or validated replacement when deep mutation would violate the contract.

## 8.2 Mutable models require explicit guarantees

`validate_assignment=True` revalidates assignments. It does not intercept all mutations of nested containers such as `model.items.append(...)`.

A cross-field `model_validator(mode="after")` can reject an assignment after fields have changed. Do not assume every rejected relational assignment is automatically transactional or rolled back. Test the exact operation when rollback semantics matter.

For coupled fields, validated replacement is often clearer than a sequence of assignments.

## 8.3 Reused model instances

Existing model instances can pass through another validation boundary without every internal value being reconstructed. When reused instances must be rechecked, configure `revalidate_instances` on the model type whose instances require revalidation and test the real nesting path.

This is a later boundary check, not continuous monitoring.

## 8.4 APIs that bypass validation

Do not use these to accept untrusted data:

- `model_construct()`;
- `model_copy(update=...)` as a validation mechanism;
- arbitrary `cast(...)`;
- unchecked object mutation.

`deep=True` on a copy changes copying behavior, not validation behavior.

A generic `model_dump()` -> merge -> `model_validate()` update helper is not automatically safe. Aliases, excluded fields, computed fields, custom serializers, nested instances, and `Json[T]` can make serialized output differ from canonical validation input. Reconstruct according to the model's actual contract.

---

# 9. Serialization and schema are contracts [ALWAYS READ]

Input, stored Python values, serialized Python data, serialized JSON, and JSON Schema can differ. Specify the differences intentionally.

## 9.1 Aliases

Use aliases for real naming requirements, not as an accidental side effect.

```python
from pydantic import BaseModel, ConfigDict, Field


class ExternalUser(BaseModel):
    model_config = ConfigDict(
        extra="forbid",
        validate_by_name=True,
        validate_by_alias=True,
    )

    user_id: str = Field(alias="userId")
```

When output names matter, choose `by_alias` or a serialization configuration deliberately and test the exact output. Validation aliases and serialization aliases are separate concerns.

## 9.2 Secrets and exclusions

`SecretStr` and related secret types reduce accidental exposure in ordinary representations. They are not encryption, access control, secure storage, or a guarantee that raw invalid input never appears in an error path.

`repr=False` does not mean “exclude from serialization.” Use explicit serialization exclusion when the output contract requires it.

## 9.3 JSON Schema

Generate schema from the actual type with `model_json_schema()` or `TypeAdapter.json_schema()`.

Custom Python validators are not automatically expressible in JSON Schema. A description in schema documents a rule but does not enforce it.

When another system or LLM produces data from the generated schema, validate the result again with the real Python validation boundary. Provider support for schema features, discriminators, recursion, and formats can differ.

## 9.4 Inheritance is not variant selection

A field annotated as a base model is validated and normally serialized according to that declared type. A runtime subclass does not make Pydantic infer a tagged variant from raw input.

For a closed family of wire variants, prefer a discriminated union. See section 13.3 for Pydantic v2.13 polymorphic serialization when runtime subclass fields genuinely must be emitted.

---

# 10. Definition of done [ALWAYS READ]

A generated model is not complete merely because the class definition imports.

Deliver or verify the relevant evidence for the contract:

| Contract area | Minimum relevant checks |
|---|---|
| Valid data | Representative accepted values through the real boundary. |
| Missing/null/default | Omitted required fields, explicit null, omitted defaulted values, invalid defaults where relevant. |
| Scalar constraints | Boundary values, zero/sign cases, empty vs blank strings, booleans vs integers, finiteness when relevant. |
| Variants | Every case, missing/unknown tag, wrong-case fields, invalid payload, ambiguous union input when applicable. |
| Unknown keys | Rejection, deliberate ignoring, or typed retention, including nested records where relevant. |
| Strictness | Expected coercions and expected rejections through Python and/or JSON entry points actually used. |
| Updates | Omitted vs explicit null, resulting-object validation, assignment behavior, nested mutation if supported. |
| Reuse | Revalidation policy for existing model instances when relevant. |
| Serialization | Exact output names, exclusions, secrets, subclass behavior, schema mode, round-trip behavior when required. |
| Static tooling | The project's configured type checker, including exhaustiveness or semantic-ID separation where promised. |

Tests should focus on meaningful failure paths rather than assert every character of version-dependent error messages.

### Core priority rules

**MUST**

- Model the actual domain rather than a convenient approximation.
- Validate untrusted input through an explicit boundary.
- Keep omission separate from null.
- Preserve promised invariants through supported operations.
- Treat serialized output as part of the contract.
- Never claim that annotations, freezing, strictness, or validators guarantee more than they actually do.

**SHOULD**

- Prefer precise shapes over optional-field clusters for true variants.
- Prefer declarative constraints and existing domain types before custom validators.
- Reject unknown keys in closed contracts.
- Validate defaults when correctness depends on them.
- Prefer immutable snapshots when mutation is not part of the domain.
- Test invalid and adversarial inputs, not only happy paths.

**MAY**

- Use callable discriminators, typed extras, runtime wrappers, generic models, custom serializers, or advanced APIs when a concrete requirement makes them preferable.

---

# 11. Topic trigger index [ALWAYS READ]

Read this index for every modeling task. Follow the triggered references based on the contract, not based on the apparent size of the model.

| If the task contains this characteristic | Read |
|---|---|
| Different semantic ID kinds such as `UserId` and `OrderId` | §12.1 Semantic identity |
| The validated value must remain a dictionary | §12.2 Typed dictionaries and typed extras |
| Known keys plus extension keys | §12.2 Typed dictionaries and typed extras |
| `ReadOnly` or dictionary immutability | §12.2 Typed dictionaries and typed extras |
| Arbitrary JSON values | §12.3 JSON value types |
| JSON text embedded inside a field | §12.3 JSON value types |
| Common formats such as URLs, email, UUID, byte sizes, dates, phone numbers, ULID | §12.4 Existing Pydantic and extra types |
| `RootModel`, dataclasses, or validating a type that is not a `BaseModel` | §12.5 Alternate runtime representations |
| Closed-union consumer logic, `assert_never`, `TypeIs`, or `TypeGuard` | §12.6 Consumer-side typing |
| Variants that do not share one discriminator field | §13.1 Callable discriminators |
| Untagged or ambiguous unions | §13.2 Union selection modes |
| Base-class fields with runtime subclasses | §13.3 Polymorphic serialization |
| Input and output aliases with version-sensitive configuration | §13.4 Alias configuration |
| `TypedDict(closed=...)` or `extra_items=...` | §13.5 PEP 728 compatibility |
| Experimental pipeline, `MISSING`, custom core schema, `SkipValidation`, `ValidateAs`, arbitrary types | §13.6 Experimental and escape-hatch APIs |
| You are considering a custom format validator | §12.4 first |
| You are considering a broad `dict[str, object]` or `Any` | §12.2 and §12.3 first |

---

# Part II. Topic reference [READ WHEN TRIGGERED]

# 12.1 Semantic identity

Two values can share the same runtime representation while representing different concepts.

## Static separation with `NewType`

Use `NewType` when the main risk is accidental mixing in typed application code and static checking covers the callers.

```python
from typing import NewType

UserId = NewType("UserId", str)
OrderId = NewType("OrderId", str)
```

`NewType` does not create a distinct runtime class. Pydantic sees the underlying representation for runtime validation.

## Runtime separation with wrappers

Use a named wrapper when runtime code must also distinguish concepts.

```python
from typing import Annotated

from pydantic import ConfigDict, Field, RootModel

NonEmptyId = Annotated[str, Field(min_length=1)]


class UserRef(RootModel[NonEmptyId]):
    model_config = ConfigDict(frozen=True)


class OrderRef(RootModel[NonEmptyId]):
    model_config = ConfigDict(frozen=True)
```

A raw string can still be validated into either wrapper unless the wire representation itself carries a distinguishing prefix or tag.

Do not claim that UUID syntax, a wrapper class, or `NewType` proves existence, ownership, authorization, or which database entity a raw value identifies.

`SecretStr` is not a semantic-ID mechanism. It addresses accidental exposure.

---

# 12.2 Typed dictionaries, typed extras, and `ReadOnly`

Use `TypedDict` when the validated result should remain a dictionary rather than become a model object.

A `TypedDict` declaration alone provides no runtime validation. Pair it with `TypeAdapter` at untrusted boundaries.

```python
from pydantic import TypeAdapter
from typing_extensions import TypedDict


class ClosedConfig(TypedDict, closed=True):
    name: str
    enabled: bool


class ExtensibleConfig(TypedDict, extra_items=str):
    name: str
    enabled: bool


closed_config_adapter = TypeAdapter(ClosedConfig)
extensible_config_adapter = TypeAdapter(ExtensibleConfig)
```

`closed=True` forbids additional keys during Pydantic validation. `extra_items=T` permits additional keys whose values validate as `T`.

For a `BaseModel` that keeps flat extras, Pydantic can type the extra-value dictionary:

```python
from pydantic import BaseModel, ConfigDict, Field


class FlatPluginConfig(BaseModel):
    model_config = ConfigDict(extra="allow")

    __pydantic_extra__: dict[str, str] = Field(init=False)
    name: str
    enabled: bool
```

Choose between flat extras and an explicit `extra: dict[...]` field based on the wire contract. They are different schemas.

## `ReadOnly` is static-only

`ReadOnly[...]` on a `TypedDict` describes a static constraint for type checkers. It does not make the runtime dictionary immutable, and Pydantic warns that it cannot protect dictionary items from mutation.

Do not use `ReadOnly` as a substitute for immutable runtime data.

If runtime immutability is required, use a representation whose supported operations actually enforce it, such as frozen models plus immutable contained values, or an explicit immutable mapping/value type where appropriate.

---

# 12.3 JSON value types

Distinguish three different contracts.

## Arbitrary decoded JSON: `JsonValue`

Use `JsonValue` when the value may be any decoded JSON-shaped value.

```python
from pydantic import BaseModel, ConfigDict, JsonValue


class EventMetadata(BaseModel):
    model_config = ConfigDict(extra="forbid")

    source: str
    extra: dict[str, JsonValue]
```

For an object specifically, use `dict[str, JsonValue]`. `JsonValue` itself can also be a scalar, array, object, or null.

Do not use `JsonValue` when the structure is known. Model known business fields explicitly.

## JSON text embedded in a field: `Json[T]`

Use `Json[T]` when the field input itself is encoded JSON text and should be parsed and then validated as `T`.

```python
from pydantic import BaseModel, ConfigDict, Json


class ToolArguments(BaseModel):
    model_config = ConfigDict(extra="forbid")
    query: str


class ToolCall(BaseModel):
    model_config = ConfigDict(extra="forbid")
    args: Json[ToolArguments]
```

If the input is already decoded, use `ToolArguments`, not `Json[ToolArguments]`.

If the whole request is JSON text, use `model_validate_json` or `TypeAdapter.validate_json`; do not wrap every field in `Json[...]`.

## Opaque JSON source text

A plain `str` can be correct when preserving the exact source text is itself part of the contract. Do not parse it automatically merely because it contains JSON-like content.

---

# 12.4 Existing Pydantic types and `pydantic-extra-types`

Before writing a custom format validator, check whether a standard-library type, Pydantic built-in type, or `pydantic-extra-types` type already represents the requirement.

| Need | Candidate | Important limitation |
|---|---|---|
| Positive/non-negative/strict numbers | `PositiveInt`, `NonNegativeInt`, `StrictInt`, `StrictFloat`, etc. | Sign, bounds, finiteness, and strictness are separate requirements. |
| Finite floating-point value | `FiniteFloat` | Does not define application-specific scale or units. |
| UUID-shaped value | `UUID` and version-specific UUID types where appropriate | Syntax does not prove entity identity or existence. |
| Timezone-aware timestamp | `AwareDatetime` | “Aware” does not mean “UTC only”. |
| Past/future time constraints | `PastDate`, `FutureDate`, `PastDatetime`, `FutureDatetime` | Validity is checked at a moment in time. |
| URL | `AnyUrl`, `HttpUrl`, DSN types | Format is not reachability or SSRF safety. |
| Email | `EmailStr` | Requires its optional dependency and does not prove mailbox ownership. |
| Human-readable byte size | `ByteSize` | Parsing units is intentional coercion. |
| Sensitive string/bytes | `SecretStr`, `SecretBytes`, `Secret[T]` | Redaction is not encryption or universal log sanitization. |
| Base64/encoded values | `Base64Bytes`, `Base64Str`, `EncodedBytes`, `EncodedStr` | Encoding is not encryption. |
| Existing file/directory | `FilePath`, `DirectoryPath` | Environment-dependent check, not access control. |

`pydantic-extra-types` is a separate dependency. Useful categories include:

- phone numbers;
- coordinates;
- MAC addresses;
- ISBN;
- country, currency, language, and script codes;
- semantic versions;
- timezone names;
- ULID;
- color values;
- payment-card formats.

Check the exact semantics and optional dependencies before adding the package.

Core `pydantic.color.Color` and `pydantic.PaymentCardNumber` are deprecated. Prefer their current `pydantic-extra-types` equivalents when those formats are genuinely required.

`ImportString` is special: it imports an object named by input. Use it only for deliberately trusted configuration. User-controlled import paths are an execution boundary, not merely a format-validation problem.

---

# 12.5 Alternate runtime representations

## `TypeAdapter`

Use `TypeAdapter` for validation/schema/serialization when the target type is not itself a `BaseModel`, such as:

- a scalar alias;
- a collection;
- a tagged union;
- a `TypedDict`;
- a dataclass-compatible type.

Create reusable adapters outside repeated hot-path calls.

## `RootModel`

Use `RootModel[T]` when a named runtime model whose wire root is `T` is useful.

Do not confuse it with a normal one-field object model. A `RootModel[str]` serializes as a string, not as `{"value": ...}`.

Do not apply an `extra` model configuration to a `RootModel`; it has no object-level extra fields to retain or reject.

## Dataclasses

Standard dataclasses do not validate merely because fields have annotations.

Use a Pydantic dataclass when dataclass semantics are required together with Pydantic validation. Use a normal `BaseModel` when no existing dataclass contract justifies the extra choice.

---

# 12.6 Consumer-side typing, exhaustiveness, and narrowing

## Exhaustive handling

For a closed union or enum, use the configured type checker and `assert_never` when adding a new case should surface missing consumer logic.

Avoid catch-all branches when silently accepting future variants would be incorrect.

Python does not gain Rust-level compiler guarantees merely because `assert_never` appears in the source. The checker must run and understand the narrowing.

## `TypeIs` and `TypeGuard`

`TypeIs[T]` is appropriate only when the predicate identifies membership in `T` in both directions.

```python
from typing import TypeGuard

from typing_extensions import TypeIs


def is_string(value: str | None) -> TypeIs[str]:
    return isinstance(value, str)


def is_non_empty_string(value: str | None) -> TypeGuard[str]:
    return isinstance(value, str) and value != ""
```

This would be unsound:

```python
# Wrong: empty strings are still str, but this returns False for them.
def is_non_empty_string_wrong(value: str | None) -> TypeIs[str]:
    return isinstance(value, str) and value != ""
```

The false branch of a `TypeIs[str]` check may exclude `str`. Therefore a predicate that recognizes only some strings must not claim `TypeIs[str]`.

A type checker trusts user-defined narrowing functions; it does not prove their implementations correct.

Do not use `cast`, `Any`, `# type: ignore`, or a custom narrowing function merely to silence a modeling error.

---

# Part III. Version-sensitive and advanced tools [READ WHEN TRIGGERED]

# 13.1 Callable discriminators and `Tag`

Use a callable `Discriminator` when variants do not share a common discriminator field and the external contract cannot reasonably be changed.

Prefer a normal shared tag such as `kind` when you control the schema. It is simpler and generally more portable in generated schema.

```python
from typing import Annotated, Literal

from pydantic import BaseModel, ConfigDict, Discriminator, Tag, TypeAdapter

class LegacyText(BaseModel):
    model_config = ConfigDict(extra="forbid")
    kind: Literal["text"]
    text: str


class LegacyFile(BaseModel):
    model_config = ConfigDict(extra="forbid")
    category: Literal["file"]
    path: str


def legacy_tag(value: object) -> str | None:
    if isinstance(value, LegacyText):
        return "text"
    if isinstance(value, LegacyFile):
        return "file"
    if isinstance(value, dict):
        if value.get("kind") == "text" and "category" not in value:
            return "text"
        if value.get("category") == "file" and "kind" not in value:
            return "file"
    return None


LegacyPart = Annotated[
    Annotated[LegacyText, Tag("text")]
    | Annotated[LegacyFile, Tag("file")],
    Discriminator(legacy_tag),
]
legacy_part_adapter = TypeAdapter(LegacyPart)
```

The discriminator must handle both raw input and model instances because Pydantic can call it during serialization as well as validation.

Return no tag for ambiguous or unsupported input instead of arbitrarily choosing a case.

Inspect generated schema before depending on a callable discriminator in another system. A shared discriminator property is easier for many external schema consumers.

---

# 13.2 Untagged unions and `union_mode`

Pydantic supports three broad union strategies:

- discriminated unions;
- smart selection for most untagged unions;
- left-to-right selection when explicitly configured.

Prefer discriminated unions for object variants when the wire contract permits them. They are more predictable and make the selection rule explicit.

A natural union such as `str | int` can be fine without a discriminator.

For overlapping shapes, do not rely on smart selection as an undocumented business rule. Smart selection can consider more than the first successful branch and its details may evolve between minor versions.

Use left-to-right mode only when branch order itself is part of the contract. Then test ambiguous inputs because coercion and order can change which member wins.

Do not use union-selection configuration to paper over a missing domain discriminator when you can model the variants explicitly.

---

# 13.3 Subclasses and polymorphic serialization

Pydantic v2 normally serializes a field according to the declared model type, not all fields present on a runtime subclass.

```python
from pydantic import BaseModel


class User(BaseModel):
    name: str


class UserLogin(User):
    password: str


class Envelope(BaseModel):
    user: User


value = Envelope(user=UserLogin(name="pydantic", password="secret"))
assert value.model_dump() == {"user": {"name": "pydantic"}}
```

This default is often desirable because it prevents a newly added subclass field from silently entering an output contract.

**Pydantic >=2.13:** polymorphic serialization can intentionally serialize a model or Pydantic dataclass according to its runtime subclass schema.

```python
assert value.model_dump(polymorphic_serialization=True) == {
    "user": {"name": "pydantic", "password": "secret"}
}
```

Treat this as an output decision with a data-exposure review. It can expose subclass fields that the declared base type intentionally hid.

Polymorphic serialization does **not** teach validation how to select a subclass from an untagged raw dictionary. Use a discriminated union or another explicit input contract when subtype selection is required.

`SerializeAsAny[...]` and `serialize_as_any=True` are broader serialization escape hatches. Prefer the narrower mechanism that matches the requirement.

---

# 13.4 Version-sensitive alias configuration

Pydantic v2.11 introduced more explicit control of name acceptance and serialization defaults.

Treat these as separate questions:

- Which field names are accepted as input?
- Which field names are used as output?
- Is the Python attribute name part of the wire contract?

Useful settings include `validate_by_name`, `validate_by_alias`, and `serialize_by_alias` where the installed Pydantic version supports them.

Do not adopt a newer alias setting simply to modernize syntax if the repository targets an older supported Pydantic version. Use the version-matched documented mechanism and preserve the existing external contract.

Test collisions or duplicate representations when both an alias and a field name may arrive.

---

# 13.5 PEP 728 `TypedDict` compatibility

Pydantic added support for PEP 728 `TypedDict` controls in v2.12.

The two important forms are:

```python
from typing_extensions import TypedDict


class ClosedPayload(TypedDict, closed=True):
    name: str


class ExtensiblePayload(TypedDict, extra_items=int):
    name: str
```

Use `typing_extensions.TypedDict` on Python versions whose standard-library `typing.TypedDict` does not support these class keywords.

Do not confuse Pydantic runtime support with static checker support. Verify the configured checker separately.

`ReadOnly[...]` is independent of `closed` / `extra_items`: it is a static qualifier and does not provide runtime dictionary immutability.

---

# 13.6 Experimental features and escape hatches

Do not use advanced APIs as automatic “stronger” alternatives to ordinary modeling.

## Experimental APIs

Examples include APIs under `pydantic.experimental`, such as the validation pipeline and experimental `MISSING` sentinel in versions that provide them.

Use them only when:

- the installed version supports them;
- the requirement materially benefits from them;
- the compatibility cost is acceptable;
- stable public alternatives are insufficient.

Ordinary omission tracking with `model_fields_set`, `exclude_unset`, a patch type, or explicit sentinels can remain preferable for stable library code.

## Escape-hatch APIs

Features such as custom core schemas, `ValidateAs`, `InstanceOf`, `SkipValidation`, arbitrary custom serializers, and `arbitrary_types_allowed` can be correct for integration boundaries.

Before using one, state exactly which normal guarantee is being bypassed or replaced and what now enforces correctness.

In particular:

- `SkipValidation` means the skipped contents are not validated by that path;
- accepting an arbitrary class instance does not validate the internal state of that instance;
- a custom serializer changes output and can invalidate assumptions about round trips;
- a custom core schema increases coupling to lower-level Pydantic behavior.

Prefer high-level public models and annotations unless a concrete integration requirement justifies the escape hatch.

---

# Appendix A. What to retain from the Rust comparison

Rust is useful here as a source of modeling ideas, not as the objective function for Python code.

The original comparison highlighted several valuable concepts:

- distinct semantic identities;
- enums with variant-specific payloads;
- impossible states removed from the type shape;
- exhaustive consumer handling;
- constructor boundaries;
- stronger mutation guarantees.

Keep those questions, but compare the same layer.

A Rust struct with typed fields is not itself a parser for arbitrary JSON. Serde performs runtime deserialization and has its own policies for unknown fields and conversions. Likewise, a Rust newtype or checked constructor protects an invariant only through construction paths that actually use that boundary.

For Python, state the real combination:

- annotations describe intended shapes for tooling;
- Pydantic validates selected runtime boundaries;
- model/update design preserves or weakens invariants after construction;
- a static type checker protects some consumer behavior;
- application logic handles external facts and state transitions.

Do not rank every design as “Python equal to Rust” or “Rust stronger.” For an implementation guide, the more useful questions are:

1. When should this Python pattern be used?
2. When should it not be used?
3. What does Pydantic enforce at runtime?
4. What does static typing enforce?
5. What remains unprotected?
6. Which test demonstrates the intended guarantee?

---

# Appendix B. Compatibility checklist and official references

## B.1 Quick compatibility checklist

Before using a version-sensitive feature:

1. Read the project's Python and Pydantic constraints.
2. Check the installed Pydantic API or version-matched official documentation.
3. Check `typing_extensions` when the feature is a typing backport.
4. Check the configured type checker independently.
5. Do not upgrade dependencies solely to copy an example from this guide.
6. Test both accepted and rejected cases through the real entry point.

## B.2 Official references

- Pydantic models: https://docs.pydantic.dev/latest/concepts/models/
- Pydantic fields: https://docs.pydantic.dev/latest/concepts/fields/
- Pydantic validators: https://docs.pydantic.dev/latest/concepts/validators/
- Pydantic unions and discriminators: https://docs.pydantic.dev/latest/concepts/unions/
- Pydantic strict mode: https://docs.pydantic.dev/latest/concepts/strict_mode/
- Pydantic serialization: https://docs.pydantic.dev/latest/concepts/serialization/
- Pydantic JSON: https://docs.pydantic.dev/latest/concepts/json/
- Pydantic JSON Schema: https://docs.pydantic.dev/latest/concepts/json_schema/
- Pydantic `TypeAdapter`: https://docs.pydantic.dev/latest/concepts/type_adapter/
- Pydantic built-in types: https://docs.pydantic.dev/latest/api/types/
- Pydantic configuration: https://docs.pydantic.dev/latest/api/config/
- Pydantic dataclasses: https://docs.pydantic.dev/latest/concepts/dataclasses/
- Pydantic experimental features: https://docs.pydantic.dev/latest/concepts/experimental/
- Pydantic v2.12 release and PEP 728 support: https://pydantic.dev/articles/pydantic-v2-12-release
- Python typing documentation: https://docs.python.org/3/library/typing.html
- Python typing guide for type narrowing: https://typing.python.org/en/latest/guides/type_narrowing.html
- `typing_extensions`: https://typing-extensions.readthedocs.io/
- Pydantic Extra Types: https://docs.pydantic.dev/latest/api/pydantic_extra_types/

---

# Appendix C. Review checklist for generated models

Use this final pass before accepting an LLM-generated model.

- Does every field have a domain reason to exist?
- Are required, nullable, and defaulted states distinct?
- Are closed vocabularies actually closed?
- Are true variants represented as variants instead of optional-field clusters?
- Are ordinary cross-field relationships kept as one shape with a clear validator?
- Is the real external boundary validated?
- Are accepted coercions intentional?
- Is the extras policy explicit where it matters?
- Can the model become invalid after construction through a supported operation?
- Are patches validated as resulting states when necessary?
- Are aliases, exclusions, secrets, and subclass serialization deliberate?
- Is known structure modeled instead of hidden in `Any`, `object`, or arbitrary JSON?
- Were existing Pydantic/domain types considered before custom validators?
- Are version-sensitive APIs supported by the project?
- Does the static checker provide every guarantee the code comments claim?
- Are relevant invalid inputs tested?
- Are any remaining guarantees actually application/service concerns rather than model concerns?

A good model should make the legal states obvious, reject invalid boundary input predictably, and avoid claiming guarantees that disappear as soon as the object is mutated, copied, serialized, or consumed.
