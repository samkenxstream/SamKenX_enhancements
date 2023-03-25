# KEP-2876: CRD Validation Expression Language

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Overview of existing validation](#overview-of-existing-validation)
  - [Descriptive, self contained CRDs](#descriptive-self-contained-crds)
  - [Webhooks: Development Complexity](#webhooks-development-complexity)
  - [Webhooks: Operational Complexity](#webhooks-operational-complexity)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
    - [Transition Rules](#transition-rules)
      - [Use Cases](#use-cases)
      - [Considerations](#considerations)
      - [Alternatives Evaluated](#alternatives-evaluated)
    - [Expression lifecycle](#expression-lifecycle)
    - [Function library](#function-library)
      - [Function Library Updates](#function-library-updates)
  - [User Stories](#user-stories)
  - [Notes/Constraints/Caveats (Optional)](#notesconstraintscaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
    - [Accidental misuse](#accidental-misuse)
    - [Malicious use](#malicious-use)
  - [Future Plan](#future-plan)
  - [CEL for General Admission Control](#cel-for-general-admission-control)
    - [CEL Custom Resource Definition Conversion](#cel-custom-resource-definition-conversion)
    - [Other expression languages](#other-expression-languages)
- [Design Details](#design-details)
  - [Type Checking](#type-checking)
  - [Type System Integration](#type-system-integration)
    - [Why not represent associative lists as maps in CEL?](#why-not-represent-associative-lists-as-maps-in-cel)
  - [Resource constraints](#resource-constraints)
    - [Estimated Cost Limits](#estimated-cost-limits)
    - [Runtime Cost Budget](#runtime-cost-budget)
  - [Request lifetime Bound](#request-lifetime-bound)
  - [Bounds](#bounds)
  - [Test Plan](#test-plan)
      - [Prerequisite testing updates](#prerequisite-testing-updates)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Beta](#beta)
    - [GA](#ga)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Graduation Criteria](#graduation-criteria-1)
  - [Beta](#beta-1)
- [Alternatives](#alternatives)
  - [Introduce CEL for General Admission Control](#introduce-cel-for-general-admission-control)
  - [Rego](#rego)
  - [Expr](#expr)
  - [WebAssembly](#webassembly)
  - [Starlark (formerly known as Skylark)](#starlark-formerly-known-as-skylark)
  - [Build our own](#build-our-own)
  - [Make it easier to validate CRDs using webhooks](#make-it-easier-to-validate-crds-using-webhooks)
- [Infrastructure Needed (Optional)](#infrastructure-needed-optional)
<!-- /toc -->

## Release Signoff Checklist

<!--
**ACTION REQUIRED:** In order to merge code into a release, there must be an
issue in [kubernetes/enhancements] referencing this KEP and targeting a release
milestone **before the [Enhancement Freeze](https://git.k8s.io/sig-release/releases)
of the targeted release**.

For enhancements that make changes to code or processes/procedures in core
Kubernetes—i.e., [kubernetes/kubernetes], we require the following Release
Signoff checklist to be completed.

Check these off as they are completed for the Release Team to track. These
checklist items _must_ be updated for the enhancement to be released.
-->

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input (including test refactors)
  - [ ] e2e Tests for all Beta API Operations (endpoints)
  - [ ] (R) Ensure GA e2e tests for meet requirements for [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md) 
  - [ ] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [ ] (R) Graduation criteria is in place
  - [ ] (R) [all GA Endpoints](https://github.com/kubernetes/community/pull/1806) must be hit by [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md) 
- [ ] (R) Production readiness review completed
- [ ] (R) Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

<!--
**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.
-->

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

CRDs need direct support for non-trivial validation. While admission webhooks do support
CRDs validation, they significantly complicate the development and
operability of CRDs.

This KEP proposes that an inline expression language be integrated directly into CRDs such that a
much larger portion of validation use cases can be solved without the use of webhooks. 

This KEP proposes the adoption of [Common Expression Language
(CEL)](https://github.com/google/cel-go). It is sufficiently lightweight and safe to be run directly
in the kube-apiserver (since CRD creation is a privileged operation), has a straight-forward and
unsurprising grammar, and supports pre-parsing and typechecking of expressions, allowing syntax and
type errors to be caught at CRD registration time.

CEL might be used in Kubernetes for extensibility beyond CRD validation. The future plans section of
this KEP explains how CEL might be used for general admission control, defaulting and
conversion. This KEP aims to prove the utility of CEL for both the immediate use case (CRD validation)
and these future use cases. This KEP also aims to use CEL in a way that is congruent with these
future use cases.

## Motivation

### Overview of existing validation

CRDs currently support two major categories of built-in validation:

- [CRD structural
schemas](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#specifying-a-structural-schema):
Provide type checking of custom resources against schemas.

- OpenAPIv3 validation rules: Provide regex ('pattern' property), range
limits ('minimum' and 'maximum' properties) on individual fields and size limits
on maps and lists ('minItems', 'maxItems').

In addition, the API Expression WG is working on KEPs that would improve CRD validation:

- OpenAPIv3 'formats' which could (and I believe should) be leveraged by
Kubernetes to handle validation of string fields for cases where regex is poorly
suited or insufficient.
- Immutability
- Unions

These improvements are largely complementary to expression support and either
are (or should be) addressed by in separate KEPs.

For use cases that cannot be covered by build-in validation support:

- Admission Webhooks: have validating admission webhook for further validation
- Custom validators: write custom checks in several languages such as Rego 

### Descriptive, self contained CRDs

This KEP will make CRDs more self-contained. Instead of having
validation rules coded into webhooks that must be
registered and upgraded independent of a CRD, the rules will be contained within
the CRD object definition, making them easier to author and introspect by
cluster administrators and users, and eliminating version skew issues that can
happen between a CRD and webhook since they can be registered and
upgraded/rolled-back independently.

### Webhooks: Development Complexity

Introducing a production grade webhook is a substantial development task.
Beyond authoring the actual core logic that a webhook must perform, the webhook
must be instrumented for monitoring and alerting and integrated with the
packaging/releases processes for the environments it will be run it.

The developer must also carefully consider the upgrade and rollback ordering
between the webhook and CRD.

### Webhooks: Operational Complexity

Admission webhooks are part of the critical serving path of the kube-apiserver.
Admission webhooks add latency to requests, and large numbers of webhooks
can cause, or contribute to, request timeouts being exceeded.

Webhooks must either be configured as `FailPolicy.Fail` or `FailPolicy.Ignore`. If
`FailPolicy.Ignore` is used, there is potential for requests skip the webhook and
be admitted. If `FailPolicy.Fail` is used, a webhook outage can result in a
localized or widespread Kubernetes control plane outage depending on which
objects the webhook is configured to intercept.

### Goals

- Make CRDs more self-contained and declarative
- Simplify CRD development
- Simplify CRD operations

### Non-Goals

- Support for validation formats, immutability or unions. These are all valuable improvements
  but can be addressed orthogonally in separate KEPs.
- Eliminate the need for webhooks entirely. Webhooks will still be needed for
  some use cases. For example, if a validation check requires making a network
  request to some other system, it will still need to be implemented in a webhook.
- Support all validation done on native Kubernetes types. For example, CRD structural schemas has
  complex validation rules that we CEL cannot support due to the lask of support for arbitrarily
  deeply nested terms (CEL cannot support recursive data types).
- Change how Kubernetes built-in types are validated, defaulted and converted. We are, however,
  interested in admission control support for build-in types. See future work for more details.

## Proposal

An inline expression language like [Common Expression Language (CEL)](https://github.com/google/cel-go) would be an
excellent supplement to the current validation mechanism because it is sufficiently expressive to
satisfy a large set of remaining uses cases that none of the above can solve.
For example, cross-field validation use cases can only be solved using
expressions or webhooks.

`x-kubernetes-validations` extension will be added to CRD structural schemas to allow CEL validation expressions.

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
...
  schema:
    openAPIV3Schema:
      type: object
      properties:
        spec:
          x-kubernetes-validations: 
            - rule: "self.minReplicas <= self.maxReplicas"
              messageExpression: "'minReplicas (%d) cannot be larger than maxReplicas (%d)'.format([self.minReplicas, self.maxReplicas])"
          type: object
          properties:
            minReplicas:
              type: integer
            maxReplicas:
              type: integer
```

Example Validation Rules:

| Rule                                                                               | Purpose                                                                           |
| ----------------                                                                   | ------------                                                                      |
| `self.minReplicas <= self.replicas && self.replicas <= self.maxReplicas`           | Validate that the three fields defining replicas are ordered appropriately        |
| `'Available' in self.stateCounts`                                                  | Validate that an entry with the 'Available' key exists in a map                   |
| `(self.list1.size() == 0) != self.list2.size() == 0)`                              | Validate that one of two lists is non-empty, but not both                         |
| `!('MY_KEY' in self.map1) \|\| self['MY_KEY'].matches('^[a-zA-Z]*$')`              | Validate the value of a map for a specific key, if it is in the map               |
| `self.envars.filter(e, e.name = 'MY_ENV').all(e, e.value.matches('^[a-zA-Z]*$')`   | Validate the 'value' field of a listMap entry where key field 'name' is 'MY_ENV'  |
| `has(self.expired) && self.created + self.ttl < self.expired`                      | Validate that 'expired' date is after a 'create' date plus a 'ttl' duration       |
| `self.health.startsWith('ok')`                                                     | Validate a 'health' string field has the prefix 'ok'                              |
| `self.widgets.exists(w, w.key == 'x' && w.foo < 10)`                               | Validate that the 'foo' property of a listMap item with a key 'x' is less than 10 |
| `type(self) == string ? self == '100%' : self == 1000`                             | Validate an int-or-string field for both the the int and string cases             |
| `self.metadata.name == 'singleton'`                                                | Validate that an object's name matches a specific value (making it a singleton)   |
| `self.set1.all(e, !(e in self.set2))`                                              | Validate that two listSets are disjoint                                           |
| `self.names.size() == self.details.size() && self.names.all(n, n in self.details)` | Validate the 'details' map is keyed by the items in the 'names' listSet           |


- Each validator may have multiple validation rules.
- Each validation rule has an optional 'message' field for the error message that
will be surfaced when the validation rule evaluates to false.
- As an alternative to the `message` field, there is also a
`messageExpression` field that represents a CEL expression that evaluates to a
string that will be used when the validation rule evaluates to false.
  - There are several validation rules for the `messageExpression` field:
    - The  `messageExpression` field must statically evaluate to a string.
    - There are several compile-time checks performed on CEL's `string.format` function that can affect `messageExpression`:
      - If the formatting string passed to `string.format` is a constant, then the formatting string will be checked at
        compile-time that it parses correctly.
      - If the `arg` array passed to `string.format` is a literal, and the formatting string is a constant, then
        at compile-time `arg`'s cardinality will be checked against the formatting string. Passing too few/too many arguments
        compared to what the formatting string expects is not allowed.
      - Type checking of each individual  argument in `arg` will be done against what is expected in the formatting string 
        to ensure the proper formatting clauses are used on each member of arg.
  - The runtime cost limit must not be exceeded during `messageExpression` execution (executing `messageExpression` counts towards that limit). 
  - If `messageExpression` results in a runtime error, then `message` will be used instead if non-empty. If `message`
    is empty, then the default error message is used instead. In either case (if `message` or the default error message is 
    used) the CEL error that `messageExpression` failed with will be logged.
  - If both `message` and `messageExpression` are present, `messageExpression` will take priority.

- The validator will be scoped to the location of the `x-kubernetes-validations`
extension in the schema. In the above example, the validator is scoped to the
`spec` field. `self` will be used to represent the name of the field which the validator
is scoped to.

  - Consideration under adding the representative of scoped field name: There would be composition
    problem while generating CRD with tools like `controller-gen`.  When trying to add validation as
    a marker comment to a field, the validation rule will be hard to define without the actual field
    name. As the example showing below. When we want to put cel validation on ToySpec, the field
    name as `spec` has not been identified yet which makes rule hard to define.
  
     ```
     // +kubebuilder:validation:XValidator=
     type ToySpec struct {
       fieldSample string `json:"fieldSample"`
       ...
     }
    
     type Toy struct {
       Spec ToySpec `json:"spec"`
     }
     ```
  
  - Alternatives: 
    - Provide a local scoped variable with a fixed name for different types:
      - scalar: value
      - array: items
      - map: entries
      - object: object
    
      It will cause a lot of keywords to be reserved and users have to memorize those variable when writing rules.
    - Using other names like `this`, `me`, `value`, `_`. The name should be self-explanatory, less chance of conflict and easy to be picked up.

- For OpenAPIv3 object types, the expression may use field selection to access all the
  properties of the object the validator is scoped to, e.g. `self.field == 10`.
  
- For OpenAPIv3 scalar types (integer, string & boolean), the expression will have access to the
  scalar data element the validator is scoped to. The data element will be accessible to CEL
  expressions via `self`, e.g. `self.size() > 10`.

- For OpenAPIv3 list and map types, the expression will have access to the data element of the list
or map. These will be accessible to CEL via `self`. The elements of a map or list can be validated using the CEL support for collections
like the `all` macro, e.g. `self.all(listItem, <predicate>)` or `self.all(mapKey,
<predicate>)`.

- Rule expressions may reference existing object state using the identifier `oldSelf`. Such
  "[transition rules](#transition-rules)" apply only under limited circumstances.

- Only property names of the form `[a-zA-Z_.-/][a-zA-Z0-9_.-/]*` are accessible and are escaped
  according to the following rules when accessed in the expression:
	- `__` escapes to `__underscores__`
	- `.` escapes to `__dot__`
	- `-` escapes to `__dash__`
	- `/` escapes to `__slash__`
	- Property names that match a CEL RESERVED keyword exactly escape to `__{keyword}__`. The
	keywords are: "true", "false", "null", "in", "as", "break", "const", "continue", "else", "for",
	"function", "if", "import", "let", "loop", "package", "namespace", "return".

- Rules may be written at the root of an object, and may make field selection into any fields
  declared in the OpenAPIv3 schema of the CRD as well as `apiVersion`, `kind`, `metadata.name` and
  `metadata.generateName`. This includes selection of fields in both the `spec` and `status` in the
  same expression, e.g. `self.status.quantity <= self.spec.maxQuantity`. Because CRDs only allow the
  `name` and `generateName` to be declared in the `metadata` of an object, these are the only
  metadata fields that may be validated using CEL validator rules. For example,
  `self.metadata.name.endsWith('mySuffix')` is allowed, but `self.metadata.labels.size() < 3` it not
  allowed. The limit on which `metadata` fields may be validated is an intentional design choice
  (that aims to allow for generic access to labels and annotations across all kinds) and applies to
  all validation mechanisms (e.g. the OpenAPIV3 `maxItems` restriction), not just CEL validator
  rules. xref rule 4 in [specifying a structural schema](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#specifying-a-structural-schema).

- If the CEL evaluation exceeds the bounds we set (details below), the server will return a 408
  (Request Timeout) HTTP status code. The timeout will be a backstop we expect to rarely be used
  since CEL evaluations are multiple orders of magnitude faster that typical webhook invocations,
  and we can use CEL expression complexity estimations
  ([xref](https://github.com/jinmmin/cel-go/blob/a661c99f8e27676c70fc00f4f328476ca4dcdb7f/cel/program.go#L265))
  during CRD update to bound complexity.

#### Transition Rules

A rule that contains an expression referencing the identifier `oldSelf` is implicitly considered a
"transition rule". Transition rules allow schema authors to prevent certain transitions between two
otherwise valid states. For example:

```yaml
type: string
enum: ["low", "medium", "high"]
x-kubernetes-validations:
- rule: "!(self == 'high' && oldSelf == 'low') && !(self == 'low' && oldSelf == 'high')"
  message: cannot transition directly between 'low' and 'high'
```

Unlike other rules, transition rules apply only to operations meeting the following criteria:

- The operation updates an existing object. Transition rules never apply to create operations.

- Both an old and a new value exist. It remains possible to check if a value has been added or
  removed by placing a transition rule on the parent node. Transition rules are never applied to
  custom resource creation. When placed on an optional field, a transition rule will not apply to
  update operations that set or unset the field.

- The path to the schema node being validated by a transition rule must resolve to a node that is
  comparable between the old object and the new object. For example, list items and their
  descendants (`spec.foo[10].bar`) can't necessarily be correlated between an existing object and a
  later update to the same object.

  - Mergeability semantics from [server-side
    apply](https://kubernetes.io/docs/reference/using-api/server-side-apply/#merge-strategy) are
    leveraged when determining whether or not a given schema node and its descendants support
    transition rules, as follows:

    - Elements of maps marked `x-kubernetes-map-type=granular` or `x-kubernetes-map-type=atomic` are
      correlated by key.

    - Elements of lists marked `x-kubernetes-list-type=map` are correlated if an element exists in
      both the old and new object having identical values for all key fields.

    - Elements of lists marked `x-kubernetes-list-type=set`, which must be scalars, do not support
      transition rules, however, set membership changes are visible to transition rules defined on
      the parent (i.e. array) node.

    - Elements of lists with type `atomic` do not support transition rules.

If all of the above criteria are satisfied for a given operation, the transition rule will be
enforced. The identifier `oldSelf` is guaranteed to be bound to a non-null value during expression
evaluation.

Errors will be generated on CRD writes if a schema node contains a transition rule that can never be
applied, e.g. "*path*: update rule *rule* cannot be set on schema because the schema or its parent
schema is not mergeable".

##### Use Cases

| Use Case                                                          | Rule
| --------                                                          | --------
| Immutability                                                      | `self.foo == oldSelf.foo`
| Prevent modification/removal once assigned                        | `oldSelf != 'bar' \|\| self == 'bar'` or `!has(oldSelf.field) \|\| has(self.field)`
| Append-only set                                                   | `self.all(element, element in oldSelf)`
| If previous value was X, new value can only be A or B, not Y or Z | `oldSelf != 'X' \|\| self in ['A', 'B']`
| Nondecreasing counters                                            | `self >= oldSelf`

##### Considerations

- The use of the prefix "old" in the identifier `oldSelf` is congruent with how `AdmissionReview`
  identifies the existing object as `oldObject`.

- CEL doesn't support checking if a root variable is bound. The macro `has` can test for the
  presence of fields, as in `has(self.child)`, but `has(oldSelf)` is not legal.

- It is possible for an attempt to update a custom resource to be rejected solely due to a
  disallowed transition from existing state. This [complicates declarative reconciliation
  tools](https://docs.google.com/document/d/1ZpCHE4yrOXoawai8Ldz49A4t9ni-QsHSuoy71VhBTLw/edit) that
  need to decide whether or not to delete-and-recreate an existing object. This problem exists today
  with fields that are immutable or immutable-once-assigned. Once a mechanism exists to communicate
  to clients that an operation was rejected due to transition errors only, that mechanism will be
  adopted for errors produced by the transition rules described in this document (see
  https://github.com/kubernetes/kubernetes/issues/107919).

##### Alternatives Evaluated

1. A boolean field is added to the extension indicating when true that the rule applies only to
   updates. The expression in "rule" may not reference the identifier "oldSelf" unless "updateRule"
   is present and true. If "oldSelf" is referenced by the expression, and updateRule is absent or
   false, an error is generated at CRD update time. The additional boolean field serves as an
   explicit user acknowledgement that the rule should be treated as a transition rule, but doesn't
   serve a technical purpose.

```yaml
x-kubernetes-validations:
  - rule: "oldSelf.xyz"
    updateRule: true
```

2. A validation context object (`context`) is exposed to expressions. For comparable update
   operations, the context object will provide access to the old value via a field. Schema authors
   take responsibility for avoiding runtime errors in more complex expressions.

```yaml
x-kubernetes-validations:
  - rule: "self.size() <= 10 || (has(context.oldSelf) && self.size() <= context.oldSelf.size())"
```

3. A new string field named "updateRule" permits CEL expressions that reference `oldSelf`. If "rule"
   and "updateRule" are mutually exclusive, this is effectively equivalent to alternative 1.

```yaml
x-kubernetes-validations:
  - updateRule: "oldSelf.xyz"
```

4. An "updateRule" field is added, as in alternative 3, but it is not mutually exclusive with the
   "rule" field. The "updateRule" applies when the transition rule criteria are satisfied, otherwise
   "rule" applies. This alternative allows validations to accept an update while rejecting the
   creation of an identical new object, but may be in conflict with support for
   auto-ratcheting. Additionally, requiring two expressions instead of one makes validation rules
   more difficult for users to understand.

```yaml
x-kubernetes-validations:
  - rule: "self in [2, 3]"
    updateRule: "self in [2, 3] || (self == 1 && self == oldSelf)"
```

#### Expression lifecycle

When CRDs are written to the kube-apiserver, all expressions will be [parsed and
typechecked](https://github.com/google/cel-go#parse-and-check) and the resulting
program will be cached for later evaluation (CEL evaluation is thread-safe and
side-effect free). Any parsing or type checking errors will cause the CRD write
to fail with a descriptive error.

#### Function library

The functions will become VERY difficult to change as this feature matures.
There are two types of risks here:

- We add a function and later need to change it.
- We don't add a function that is essential for use cases and later need to add it.

Proposal:

- Keep all the CEL standard functions and macros as well as the extended string library.
- Introduce a new "kubernetes" CEL extension library.
  - Add `isSorted` for lists with comparable elements. This is useful for ensuring that a list is kept in-order.
  - Add `sum` for lists of {int, uint, double, duration, string, bytes} (consistent with the [CEL _+_ operation](https://github.com/google/cel-spec/blob/master/doc/langdef.md#list-of-standard-definitions) )
    and add `min`, `max` for lists of {bool, int, uint, double, string, bytes, duration, timestamp} (consistent with the [CEL comparison operations](https://github.com/google/cel-spec/blob/master/doc/langdef.md#list-of-standard-definitions)).
    These operations can also be used in CEL on scalars by defining list literals
    inline , e.g. `[self.val1, self.val2].max()`. Overflow will raise the same error raised by arithmetic operation [overflow](https://github.com/google/cel-spec/blob/master/doc/langdef.md#overflow).
  - Add `indexOf` / `lastIndexOf` support for lists (overloading the existing string functions), this can be useful for
    validating partial order (i.e. the tasks of a workflow)
  - Add URL parsing: `url(string) URL`, `string(URL) string` (conversion), `getScheme(URL) string`, `getUserInfo(URL) string`,
    `getHost(URL) string`, `getPort(URL) string`, `getPath(URL) string`, `getQuery(URL) string`, `getFragment(URL) string`. This is
    useful for validating URLs. The go URL parser implements all the functionality.
  - Add regex `find(string) string` and `findAll(string) []string`. This makes it possible to parse out parts of string
    formats (we only provide timestamp, duration and url support directly).

The function libraries we need can be added using [extension
functions](https://github.com/google/cel-spec/blob/master/doc/langdef.md#extension-functions) to either cel-go (if they accept our proposals)
or directly to the Kubernetes codebase.

How we selected this function library:

First let's look at what CEL provides, and then look at what some of the core libraries of major programming
languages provide, and see what notable absences there are.

CEL standard functions, defined in the [list of standard definitions](https://github.com/google/cel-spec/blob/master/doc/langdef.md#list-of-standard-definitions) include:

- `size` (or string, lists, maps, ...)
- String inspection: `startsWith`, `endsWith`, `contains`
- Regex: `matches`
- Date/time accessors: `getDate`, `getDayOfMonth`, `getDayOfWeek`, `getDayOfYear`, `getFullYear`, `getHours`, `getMilliseconds`, `getMinutes`, `getMonth`, `getSeconds`

CEL standard [macros](https://github.com/google/cel-spec/blob/master/doc/langdef.md#macros) include:

- `has`
- `all`
- `exists`
- `exists_one`
- `map`
- `filter`

CEL [extended string function library](https://github.com/google/cel-go/blob/master/ext/strings.go) includes:

- `charAt`
- `indexOf`, `lastIndexOf`
- `upperAscii`, `lowerAscii`
- `replace` (string literal replacement, not regex)
- `split`
- `substring`
- `trim`

Notable absences from the standard functions, when comparing against the core libraries of Go, Java and Python are:

Strings:

- trim/trimLeft/trimRight (trimming either by whitespace or by provided cutset)
- trimPrefix/trimSuffix
- join
- replace (regex)
- format (e.g. `fmt.Sprintf()`)

Lists:

- sort
- indexOf/lastIndexOf
- replace
- sublist
- split
- reverse
- sum/min/max (xref: [Rego function library](https://www.openpolicyagent.org/docs/latest/policy-reference/#built-in-functions))

Numbers:

- abs/cel/floor/round
- exp/log/log10/logN/pow/sqrt
- max(x, y, ..)/min(x, y, ...)
- <trig functions>

Formats:

- semver: compare (xref [Kyverno function library](https://github.com/kyverno/kyverno/blob/main/pkg/engine/jmespath/functions.go))
- urls: parsing out {scheme, userInfo, host, port, path, query, fragment}. This is [hard](https://community.helpsystems.com/forums/intermapper/miscellaneous-topics/5acc4fcf-fa83-e511-80cf-0050568460e4?_ga=2.113564423.1432958022.1523882681-2146416484.1523557976)
  to get right with regex, particularly with ipv6.
- images: parsing out {name, tag, digest} as well as {host, port} from name and algorithm, hex} from digest. (xref: [grammar](https://github.com/distribution/distribution/blob/v2.7.1/reference/reference.go#L4-L24)).

Maps:

- <Ability to construct a map using a comprehension>

Some considerations when selecting which of the above we should include in CEL expressions in Kubernetes:

- We should include functions that will be needed for future use cases of CEL in Kubernetes (e.g. general admission) as
  ones needed for validation.
- Regex replace is very powerful and useful. It is also potentially dangerous due to its ability to allocate memory.
- `format`: Since CEL supports string concatenation, the value of having format would only be to do things like format floats.
  Formatting functions are complex and require extensive documentation to teach.
- `isSorted` makes it possible to check if a list is sorted without the expense of performing a sort
- `indexOf` / `lastIndexOf` / `split` / `replace` (the string functions) will be overloaded to provide the equivalent list functions.
- `reduce` is going to be non-obvious to anyone that doesn't have a functional programming background?
  - If we provide `reduce` are developers going to also expect to have `fold` or `zip`?
- "Ability to construct a map using a comprehension": this appears mechanically problematic to support in CEL?
- None of the other policy libraries provide extended math/trig support and I asked Tristan (maintainer of CEL) if
  they are requested and he said "almost never", which he clarified to mean it has been requested exactly one time.
- Average and quantile (median, 99th percentile, ...) and stddev are, however, often requested by other CEL users

Future work:

We've decided NOT to add the following functions. They may be useful for mutation, in which case we will consider adding them as part
of any future work done involving CEL and mutating admission control:

- Regex replace
- Add `trimPrefix` / `trimSuffix` for strings
- Add `trim` / `trimLeft` / `trimRight` (overloaded to take an optional cutset arg) for strings
- `sublist`, `split`, `replace` and `reverse` for lists (overloading existing string functions where appropriate)

##### Function Library Updates

Any changes to the CEL function library will make it possible for CRD authors to create CRDs that are incompatible with
all previous Kubernetes releases that supported CEL. Because of this incompatibility, changing the function library
will need to carefully considered and kept to a minimum. All changes will need to be limited to function additions.
We will not add functions to do things that can already be reasonably accomplished with existing functions Improving ease-of-use
at the cost of fragmentation / incompatibility with older servers is not a good trade-off.

Any function library change must follow the [API Changes](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api_changes.md)
guidelines. Since a change to the function library is a change to the `x-kubernetes-validations.rule` field, it must be
introduced one Kubernetes release prior to when it may be included in create/update requests to CRDs. Specifically:

- Kubernetes Version N-1: Entirely unaware of the library update.
- Kubernetes Version N: Supports CRD that use library updates, but does not allow the library updates to used when
  setting `x-kubernetes-validations.rule` fields.
- Kubernetes Version N+1: Library update fully enabled.

The mechanism for this will be:

- All new functions, macros, or overloads of existing functions, will be added to a separate "future compatibility" CEL 
  extension library (which extension library a function is in is an internal detail that is not visible to users).
- For create requests, and for any CEL expressions that are changed as part of an update, the CEL expression will be 
  compiled **without** the "future compatibility" CEL extension library.
- For CEL expressions not changed in an update, the CEL expression will be compiled **with** the
  "kubernetes-future-compatibility" CEL extension. This ensures that persisted fields that already use the change continue
  to compile.
- The "future compatibility" CEL extension library will always be included when CEL expressions are evaluated.
- When the next version of Kubernetes is release, the library functions will be moved from "future compatibility"
- to the main CEL extension library we use to extend CEL for Kubernetes.

Alternatives considered:

Versioning Alternative:

- All changes to CEL (core language and libraries) will be versioned and CEL expressions must specify the version they
  are compatible with.
- All versions must be backward compatible (a newer version of CEL is compatible with all older versions of CEL)
- The API server rejects requests containing CEL versions newer than it supports.

Pros:

- Very clear what CRDs are compatible with what Kubernetes versions.

Cons:

- Validating that a CEL expression is the version it claims to be is difficult for older version of CEL. Either the API
  server would need to contain multiple versions of the CEL compiler, or it would need a compiler that can be configured to
  compile a CEL expression according to the language rules of an older version.

### User Stories

- Cases provided by @deads2k
  - list of type foo struct {name string ... }, no item in the list can have a name == "value X", [ref](https://github.com/openshift/kubernetes/blob/75ee3073266f07baaba5db004cde0636425737cf/openshift-kube-apiserver/admission/customresourcevalidation/apiserver/validate_apiserver.go#L68).
  - metadata.name must equal "valueX", [ref](https://github.com/openshift/kubernetes/blob/75ee3073266f07baaba5db004cde0636425737cf/openshift-kube-apiserver/admission/customresourcevalidation/apiserver/validate_apiserver.go#L47)
  - if name == "foo", then fieldX must not be nil, [ref](https://github.com/openshift/kubernetes/blob/75ee3073266f07baaba5db004cde0636425737cf/openshift-kube-apiserver/admission/customresourcevalidation/apiserver/validate_apiserver.go#L177)
  - if name == "foo", then field X must be nil, [ref](https://github.com/openshift/kubernetes/blob/75ee3073266f07baaba5db004cde0636425737cf/openshift-kube-apiserver/admission/customresourcevalidation/apiserver/validate_apiserver.go#L177)
  - validate new label selector format: metav1.LabelSelector, this matches our deployment API
  - validate old label selector format: map[string]string, this matches our node selectors for pods
  - len(A)>0 xor len(B)>0
  - quantity validation to match quota and usage specifications, [buried inside of here (most of it is not generally applicable)](https://github.com/openshift/kubernetes/blob/75ee3073266f07baaba5db004cde0636425737cf/openshift-kube-apiserver/admission/customresourcevalidation/clusterresourcequota/validation/validation.go#L44)
  - if old value is X, don't allow changing the value, [ref](https://github.com/openshift/kubernetes/blob/75ee3073266f07baaba5db004cde0636425737cf/openshift-kube-apiserver/admission/customresourcevalidation/features/validate_features.go#L82)
  - shorthand to match our NameIsDNSSubdomain and the like would be nice.
  - if X exists in list1, then X must not existing in list2, both list1 and list2 are in the manifest, [ref](https://github.com/openshift/kubernetes/blob/75ee3073266f07baaba5db004cde0636425737cf/openshift-kube-apiserver/admission/customresourcevalidation/securitycontextconstraints/validation/validation.go#L139)
- Use case: [Tekton pipeline validation](https://github.com/tektoncd/pipeline/blob/main/pkg/apis/pipeline/v1beta1/pipeline_validation.go)
  - Referential integrity checks
  - Custom formatted validation error messages
  - "Either list X or list Y must be non-empty"
  - "There exists a X in one list/map for each Y in another map/list"
- (PRs to add additional user stories to this list welcome!)

### Notes/Constraints/Caveats (Optional)

While we believe the expressiveness of CEL is pretty complete for our purposes, it is non-turing
complete, and lacks support recursive data types (OpenAPIv3 & CRD structural schemas are not
possible to validate with CEL).

### Risks and Mitigations

#### Accidental misuse

Break the control plane by consuming excessive CPU and/or memory the api-server.

Mitigation: CEL is specifically designed to constrain the running time of expressions and to limit
the memory utilization. We will run a series of performance benchmarks with CEL programs designed
utilize a range of CPU and memory resources and document the results of the benchmarks before
promoting this feature to GA.

Also we can use [CEL complexity
estimations](https://github.com/jinmmin/cel-go/blob/a661c99f8e27676c70fc00f4f328476ca4dcdb7f/cel/program.go#L265)
to help bound running time.

#### Malicious use

Breaking out of the sandbox to run untrusted code in the apiserver or exfiltrate data.

Mitigation: CEL is designed to sandbox code execution. Also, because CRD creation is a privileged
operation, it should be safe to integrate.

Additional limits we can put in place, as needed, include:
- Use [CEL complexity
  estimations](https://github.com/jinmmin/cel-go/blob/a661c99f8e27676c70fc00f4f328476ca4dcdb7f/cel/program.go#L265)
  to bound running time.
- A max execution time limit to but could bound running time of CEL programs. This would require
  modifying CEL (by working with the CEL community) to make CEL evaluation cancelable. Ideally this
  would be based on CPU time dedicated to CEL evaluation, but since there is no clear way to measure
  that, it might have to be based on wall time, which is a poor signal of resource consumption and
  so would need to be set very high and used primarily as a backstop.
- Work with the CEL community to introduce metrics that approximate actual CPU and/or memory
  consumption of CEL programs.

### Future Plan

### CEL for General Admission Control

The ability to use CEL to extend Kubernetes admission control could allow for both validating and
mutating admission control without the use of webhooks. It improves on this KEP by providing
admission control of all Kubernetes objects, including native types.

We decided to start with CRD validation because it is:

- Better scoped (validation rule can be constrained to just the relevant field/subtree)
- Self-contained (single CRD manifest that all takes effect at once)
- Easier to type check (a CRD validation rule has more knowledge about the schema of the data that
it will be asked to validate than an admission expression that could have arbitrary resources routed
to it)

Even if we later add general admission control support using CEL, we believe having inline validation support
in CRDs is sufficiently valuable due to its convenience that it should exist as it's own feature. We also
believe CRD validation expressions can be kept congruent with general admission control CEL support.

(Thanks @liggitt for idea of using CEL for general admission control. This section is largely
a copy-paste of [this comment](https://github.com/kubernetes/enhancements/pull/2877#discussion_r704513565)).
  
#### CEL Custom Resource Definition Conversion

Similar to CEL for General Admission Control, CEL could also be used to author CRD Conversions.

#### Other expression languages

While we believe CEL should be sufficient. If another language we introduce in the future, it would be supported:

- Add an identifier for each expression language.
- Add a `type` field specified to `x-kubernetes-validations` (it would default to `cel`).
- Implement the validation support for the languages or even allow 3rd party validators to have a
  way to inline their validation rules in CRDs?

## Design Details

<!--
This section should contain enough information that the specifics of your
change are understandable. This may include API specs (though not always
required) or even code snippets. If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.
-->

### Type Checking

CEL type checking requires "declarations" be registered for any types that are to
be type checked.  In our case, the type information of interest is the CRD's
structural schemas. So we need to translate structural schemas to declarations.

CEL provides direct support for type checking protobuf types, which is not useful
for our use case.

The good news is that https://github.com/google/cel-policy-templates-go already has
demonstrated integrating CEL with OpenAPIv3. We plan to leverage this work.

Support for comparing integers and floats is supported in beta and later (in alpha, ints could only be compared to ints and floats to floats).

We will add detailed test coverage for numeric comparisons due to
[google/cel-spec#54](https://github.com/google/cel-spec/issues/54#issuecomment-491464172) including
coverage of interactions in these dimensions:

- schemas defining integer and number fields
- data specifying integers and float values (and integer values for float fields)
- expressions specifying literal integers, literal floats, and explicitly typed integers and floats

(Thanks to @liggitt for pointing this out)

### Type System Integration

Types:

| OpenAPIv3 type                                     | CEL type                                                                                                                     |
|----------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| 'object' with Properties                           | object / "message type" (`type(<object>)` evaluates to `selfType<uniqueNumber>.path.to.object.from.self`                     |
| 'object' with AdditionalProperties                 | map                                                                                                                          |
| 'object' with x-kubernetes-embedded-type           | object / "message type", 'apiVersion', 'kind', 'metadata.name' and 'metadata.generateName' are implicitly included in schema |
| 'object' with x-kubernetes-preserve-unknown-fields | object / "message type", unknown fields are NOT accessible in CEL expression                                                 |
| x-kubernetes-int-or-string                         | union of int or string,  `self.intOrString < 100 \|\| self.intOrString == '50%'` evaluates to true for both `50` and `"50%"`  |
| 'array                                             | list                                                                                                                         |
| 'array' with x-kubernetes-list-type=map            | list with map based Equality & unique key guarantees                                                                         |
| 'array' with x-kubernetes-list-type=set            | list with set based Equality & unique entry guarantees                                                                       |
| 'boolean'                                          | boolean                                                                                                                      |
| 'number' (all formats)                             | double                                                                                                                       |
| 'integer' (all formats)                            | int (64)                                                                                                                     |
| <no equivalent>                                    | uint (64)                                                                                                                    |
| 'null'                                             | null_type                                                                                                                    |
| 'string'                                           | string                                                                                                                       |
| 'string' with format=byte (base64 encoded)         | bytes                                                                                                                        |
| 'string' with format=date                          | timestamp (google.protobuf.Timestamp)                                                                                        |
| 'string' with format=datetime                      | timestamp (google.protobuf.Timestamp)                                                                                        |
| 'string' with format=duration                      | duration (google.protobuf.Duration)                                                                                          |

xref: [CEL types](https://github.com/google/cel-spec/blob/master/doc/langdef.md#values), [OpenAPI
types](https://swagger.io/specification/#data-types), [Kubernetes Structural Schemas](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#specifying-a-structural-schema).

Although `x-kubernetes-preserve-unknown-fields` allows custom resources to contain values without
corresponding schema information, we will not provide access to these "schemaless" values in CEL
expressions. Reasons for this include:

- Without schema information, types (e.g. `map` vs. `object`), formats (e.g. plain `string`
  vs. `date`) and list types (plain `list` vs. `set`) are not available, and this feature depends
  on this schema information to integrate custom resources values with CEL.

- The contents of unknown fields are unvalidated. So it is possible for the contents to be any
  possible values. This significantly complicates the authoring of validation rules and makes them
  far more difficult write safely. Specifically, all unchecked field selection and assumptions about
  value types will result in runtime errors when the values contents of the unknown data does not
  match the expectations of the validation rule author. None of the benefits of static type checking
  at CRD registration time would apply.

- For objects with both `x-kubernetes-embedded-type` and `x-kubernetes-preserve-unknown-fields` set to `true`,
  if we were to allow for validation of embedded resource while the validation rules that are part of
  the embedded resource are not enforced, we would be allowing for developers to use CEL to validate
  the contents of embedded types without allowing developers to write validation rules directly on the
  embedded type to validate it. We don't want to do that.

Implementation:

Type integration will be added by implementing CEL's
[ref.Val](https://github.com/google/cel-go/blob/8e5d9877f0ab106269dee64e5bf10c5315281830/common/types/ref/reference.go#L40)
and related [trait interfaces](https://github.com/google/cel-go/tree/master/common/types/traits). 

The initial changes made in the type integration will be:

- `Equals` for lists with type "map" and "set" will be map and set based instead of list based (order will be ignored)
- `Add` for lists with type "map" will overwrite existing entries with the same key, but with the position of existing entries in the list retained. New entries will be appended.
- `Add` for lists with type "set" will perform a union. The positions of existing set entries in the list will be retained and new set entries will be appended.

#### Why not represent associative lists as maps in CEL?

- CEL maps must be keyed by a single scalar type, "associative lists" may be keyed by 1+ scalar key fields
- While it would be possible to use a string representation of mult-field maps, there are problems:
  - Developers [sometimes care](https://github.com/kubernetes/kubernetes/issues/104641) about associative list order, and if allow the associative lists to be treated as a map in a way that results in order being discarded, we would break them.
  - We couldn't statically type check the key string literals being used as multi part keys
  - We couldn't have both: (a) equality of string keys, which implies a canonical ordering of key fields, and (b) ability for developers to successfully lookup a map entry regardless of how they order the keys in the string representation


So instead of treating "associative lists" as maps in CEL, we will continue to treat them as lists, but override equality to ignore object order (i.e. use map equality semantics).

Looking up and iterating over entiries is available via the `exists`, `exists_one` and `all` macros:

```
// Since there is already a guarantee that "associative lists" only one entry
// exists in the list for each key, `exists` can be used to check if a map contains a particular key instead of `exists_one`.
// Note that `exists_one` can also be used, but handles any errors encountered more strictly.
// See the CEL language spec how errors are handled by `exists_one` and `exists` for more details.

// Check if an "associative list" contains an entry for a key:
associativeList.exists(e, e.key1 == 'a' && e.key2 == 'b')

// To validate a map contains an entry with a particular key and that some condition is met on the other fields of the entry:
associativeList.exists(e, e.key1 == 'a' && e.key2 == 'b' && e.val == 100)

// To check the value for a particular key meets some condition (but also allow the entry to be absent):
associativeList.all(e, e.key1 == 'a' && e.key2 == 'b' && e.val == 100)

// To check some condition on all entries of an "associative list":
associativeList.all(e, e.val == 100)
```

### Resource constraints

CEL expressions have the potential to consume unacceptable amounts of API server resources. We intend to 
constrain the resource utilization in a few ways:

- Validation of CEL expression's "cost" when a CEL expression is written to a field in a CRD (at CRD creation/update time)
- Use a runtime cost budget during CEL evaluation that is integrated with API Priority and Fairness to determine the budget
- Use go context cancelation to bound CEL expression evaluation to the request lifetime

Combined, these limits protect against both accidental and malicious misuse and also work in concert with API Priority and Fairness to ensure the
API server as a whole makes good use of the available resources.

#### Estimated Cost Limits

Validation of CEL expression's "cost" is primarily intended to provide authors of CEL expressions with 
actionable feedback on the expected runtime costs of their expressions and prevent them from authoring 
expressions that have poor worst case running times.

We will use CEL's [cost subsystem](https://github.com/google/cel-go/blob/dfef54b359b05532fb9695bc88937aa8530ab055/cel/program.go#L309) to provide our proactive limits. If an
expression is analyzed and has a cost greater than a predefined limit, it will not be allowed. If an
expression is rejected for this reason, the error message will include how much the limit was exceeded.

A major problem with the cost system is assigning the cost of list iteration. The cost of a CEL expression is 
computed statically without any knowledge about the data that will be validated. Only the CEL expression is 
available when computing cost. This means that the sizes of variable length data types (lists, maps, sets, 
strings, byte arrays) are entirely unknown. This limits our options for computing the cost of CEL expressions 
that iterate across these list types. Setting high costs for iteration is a major problem because, in 
practice, lists are short. For example, a pod has a list of containers, but the list is usually quite short.

There are a couple ways to address this problem:

- We require CRD authors to provide `maxLength` on all variable length data types that they iterate across in CEL expressions so we know exactly how to compute cost
- We do not attempt to statically compute a cost and instead rely entirely on time based limits
- We prove that, in practice, the cost limits can be found that are operationally safe while still not being overly restrictive

We have explored the cost system and realized (thanks @liggitt!) that O(n) iteration is bounded by custom 
resource size limits (specifically the 3MB request limit) and we can use this to establish more manageable 
worst case sizes.

If a list contains small elements (e.g. a `0` stored in JSON) then the worst case list size can be
3MB/2bytes=~1572k elements, but as the element size grows, the worst case list size quickly becomes 
manageable. We ran a [series of experiments](https://docs.google.com/document/d/1yR746Rf-rw-_zoq36Ypzu8LTqOa-LaCk1pv3e0kjF3A/edit?usp=sharing) to test the resource utilization of CEL expressions that iterate across lists. We found that (with a safety factor of 5x):

- O(n) iteration of a worst case list (list length == 1572k) is < 2000ms (400ms x5)
- O(n) iteration of 10 byte data elements (list length == 300k) is < 350ms (70ms x5)
- O(n) iteration of 100 byte data elements (list length == 30k) is < 35ms (7ms x5)

Consider an example:

```
list.all(element, element.startsWith("prefix:"))
```

The cost of this expression is:

```
cost('startsWith()' function) * worst_case_length(list)
```

If the `worst_case_length()` is 30k for the list (e.g. the list elements are objects where the minimum size of 
each object can be computed statically to be 100 bytes) and the cost of `statsWith()` is, say, 3, we know that 
this is < 21ms to evaluate.

Note that we could also have a much lower `worst_case_length()` if the CRD author provided a `maxLength` limit on their list size.

This implies that we can use cost to:

- Set limits that ensure CEL expressions evaluate within known time bounds
- Allow O(n) operations so long as the per-element cost is moderate (e.g. 100 cost)
  - And again, allow more expensive per-element cost when `maxLength` is set
- Allow many (100s) CEL expressions in a CRD that are within these complexity bounds
- Disallow O(n<sup>2</sup>) or worse operations based purely on their cost
  - But still allow O(n<sup>2</sup>) or worse so long as the CRD author adds a `maxLength` limit to the lists and the resulting cost is within cost bounds (e.g. O(n<sup>3</sup>) is still fine for a list size of 3).

Also, not requiring `maxLength` on most O(n) CEL expressions keeps the cost system low friction; the majority 
of CEL expressions can be written and used without bumping into cost limits or needing to set `maxLength`.

For the cases where a `maxLength` is needed to ensure cost is acceptable, encouraging CRD authors to include 
`maxLength` on variable length data elements is generally considered good hygiene anyway, so having a cost 
system that incentivizes developers to set this field, has other benefits.

We will use the minimum possible size of list elements to estimate the size (e.g. objects with many fields 
will have much larger minimum sizes). This helps us pick smaller worst case list sizes that we might pick if we had to assume all lists contain 1 byte elements.

For example, the list element size `x` in this schema:

```
spec:
  type: object
  properties:
    x:
      x-kubernetes-validation:
      - rule: "self.all(element, element.y.startsWith('prefix:'))"
      type: array
      items:
        type: object
        properties:
          name: y
          type: string
          name: z
          type: string
```

The worst case list size of `x` would be based on the size of an object with two strings (`y` and `z`).

We will also account for where in a schema CEL expression is placed when calculating its cost. For example:

```
spec:
  type: object
  properties:
    x:
      type: array
      items:
        type: object
        properties:
          name: y
          type: string
          x-kubernetes-validation:
          - rule: "self.startsWith('prefix:')"
```

In this example the validation rule is contained within a list, so its expression cost would be `cost(startsWith) * worst_case_length(x)`. Note that this means some of the cost calculation will
happen outside of the CEL cost function (which would only calculate the `startsWith` cost).

During CRD validation, when an expression exceeds the cost limit, the error message will include both the limit and the cost of the expression to facilitate debugging. The cost calculation rules will be documented.

#### Runtime Cost Budget

We will instrument CEL to increment a "cost" counter during CEL expression evaluation. The costs for 
operations will be the same as used to compute Estimated Costs. The main difference is that the cost of list 
iteration and branch operations (ternary, short circuited ORs) will be measured during evaluation and won't 
need to be based on worst case estimates. If the cost exceeds a provided cost limit, CEL evaluation will be halted.

We will integrate with the API Priority and Fairness system, providing it with information needs to be 
informed about CEL resource utilization.

### Request lifetime Bound

We will wire in the request context into CEL so that CEL will halt expression evaluation if the context is 
canceled.

### Bounds

We will set cost bounds in the future on both a per-request and per-expression basis as we perform further
testing and benchmarking. 

### Test Plan

<!--
**Note:** *Not required until targeted at a release.*
The goal is to ensure that we don't accept enhancements with inadequate testing.

All code is expected to have adequate tests (eventually with coverage
expectations). Please adhere to the [Kubernetes testing guidelines][testing-guidelines]
when drafting this test plan.

[testing-guidelines]: https://git.k8s.io/community/contributors/devel/sig-testing/testing.md
-->

[x] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

##### Prerequisite testing updates

<!--
Based on reviewers feedback describe what additional tests need to be added prior
implementing this enhancement to ensure the enhancements have also solid foundations.
-->
N/A

##### Unit tests

<!--
In principle every added code should have complete unit test coverage, so providing
the exact set of tests will not bring additional value.
However, if complete unit test coverage is not possible, explain the reason of it
together with explanation why this is acceptable.
-->

<!--
Additionally, for Alpha try to enumerate the core package you will be touching
to implement this enhancement and provide the current unit coverage for those
in the form of:
- <package>: <date> - <current test coverage>
The data can be easily read from:
https://testgrid.k8s.io/sig-testing-canaries#ci-kubernetes-coverage-unit

This can inform certain test coverage improvements that we want to do before
extending the production code to implement this enhancement.
-->
The unit tests are added together with the added code:

- `k8s.io/apiextensions-apiserver/pkg/apiserver/schema/cel`: `May 23rd, 2022` - `79.6%`
- `k8s.io/apiextensions-apiserver/third_party/forked/celopenapi/model`: `May 23rd, 2022` - `76.5%`
- `k8s.io/apiextensions-apiserver/pkg/registry/customresourcedefinition`: `May 23rd, 2022` - `13.1%`
- `k8s.io/apiextensions-apiserver/pkg/apiserver/validation`: `May 23rd, 2022` - `87.2%`

##### Integration tests

<!--
This question should be filled when targeting a release.
For Alpha, describe what tests will be added to ensure proper quality of the enhancement.

For Beta and GA, add links to added tests together with links to k8s-triage for those tests:
https://storage.googleapis.com/k8s-triage/index.html
-->
The integration test has been added to test the crd expression validation with feature gate on/off:

- test/integration/apiserver/crd_validation_expressions_test.go

##### e2e tests

<!--
This question should be filled when targeting a release.
For Alpha, describe what tests will be added to ensure proper quality of the enhancement.

For Beta and GA, add links to added tests together with links to k8s-triage for those tests:
https://storage.googleapis.com/k8s-triage/index.html

We expect no non-infra related flakes in the last month as a GA graduation criteria.
-->
We plan to add e2e test under api-machinery for crd expression validation:

- test/e2e/apimachinery/crd_expressions_validation.go: https://storage.googleapis.com/k8s-triage/index.html?sig=api-machinery

### Graduation Criteria

#### Alpha

- Feature implemented behind a feature flag
- Ensure proper tests are in place.

#### Beta

- Resolve topic of what support we should provide for access to the previous versions of object (ie. 'oldSelf' feature)
- x-kubernetes-int-or-string is upgraded to use a union type of just int or string, not a dynamic type (CEL go support is planned in lates 2021)
- Understanding of upper bounds of CPU/memory usage and appropriate limits set to prevent abuse.
- Build-in macro/function library is comprehensive and stable (any changes to this will be a breaking change)
- CEL numeric comparison issue is resolved (e.g. ability to compare ints to doubles)
- [Reduce noise of invalid data messages reported from cel.UnstructuredToVal](https://github.com/kubernetes/kubernetes/issues/106440)
- [Benchmark cel.UnstructuredToVal and optimize away repeated wrapper object construction](https://github.com/kubernetes/kubernetes/issues/106438)
- Demonstrate adoption and successful feature usage in the community
- Optimization on super-linear complexity growth
- Adding metric of the latency of CEL evaluation for CRD evaluation

#### GA

- Formatted message. This targets in 1.27 and ideally should be in at least one release before going to GA.
- Scalability evaluation. Evaluate CRD scalability with validation rules against [previous scale target](https://github.com/kubernetes/enhancements/blob/master/keps/sig-api-machinery/95-custom-resource-definitions/README.md#scale-targets-for-ga).

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

###### How can this feature be enabled / disabled in a live cluster?

- [x] Feature gate (also fill in values in `kep.yaml`)
  - Feature gate name: CustomResourceValidationExpressions
  - Components depending on the feature gate: kube-apiserver

###### Does enabling the feature change any default behavior?

No, default behavior is the same.

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Yes, disabling the feature will result in validation expressions being ignored.

We will add a unit test that ensures that if the featuregate is off, but the x-kubernetes-validations
field is present, custom resource definition updates that do not add additional x-kubernetes-validations
fields will succeed.

###### What happens if we reenable the feature if it was previously rolled back?

Validation expressions will be enforced again.

###### Are there any tests for feature enablement/disablement?

These will be introduced in the Alpha implementation.

### Rollout, Upgrade and Rollback Planning

<!--
This section must be completed when targeting beta to a release.
-->

###### How can a rollout or rollback fail? Can it impact already running workloads?

`x-kubernetes-validations` it not currently allowed in the OpenAPI schemas defined in Custom Resource
Definitions. This creates a rollout issue: Any CRDs that are defined using this new field will
be invalid according to versions of Kubernetes that pre-date the introduction of the field.


Mitigation: Once we introduce the field, also backport the code that allows it to be included (but
ignored) in CRDs to all supported Kubernetes versions. Before this feature goes to Beta we will
need to make an assessment of how much support we have in older kubernetes versions for this feature.

###### What specific metrics should inform a rollback?

Custom resource create/update failures.

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

Will be completed before Beta.

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

No

### Monitoring Requirements

<!--
This section must be completed when targeting beta to a release.
-->

###### How can an operator determine if the feature is in use by workloads?

Check if there exist any custom resource definition with the x-kubernetes-validations field in the OpenAPIv3 schema.

###### How can someone using this feature know that it is working for their instance?

Test that a validation rule rejects a custom resource create/update/patch/apply as expected.

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

No impact on latency for custom resource create/update/patch/apply when validation rules are absent
from a custom resource definition.

Performance when validation rules are in use will need to be measured and optimized. We anticipate negligible
impact (<5%) for typical use.

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

Custom resource definition create/update/patch/apply latencies are available today and should be sufficient.

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

We don't anticipate the performance implications to justify the introduction of a validation latency metric, but
if performance 

### Dependencies

###### Does this feature depend on any specific services running in the cluster?

No

### Scalability

###### Will enabling / using this feature result in any new API calls?

No

###### Will enabling / using this feature result in introducing new API types?

No

###### Will enabling / using this feature result in any new calls to the cloud provider?

No

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

Not immediately, but custom resource definitions might become larger (anticipating <10% size increase based on similar
functionality in other systems).

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

Yes, custom resource create/update/patch/apply latencies will be impacted when the feature is used. We expect this to be negligible
but will measure it before Beta.

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

We don't expect it to. We will measure this before Beta.

Specifically we will ddemonstrate an upper bound on the resource cost someone could incur with a CEL expression that is some some
combination of large, compact, complex vs. similar combinations using existing validation rules (suggested by @liggitt)

### Troubleshooting

<!--
This section must be completed when targeting beta to a release.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.

The Troubleshooting section currently serves the `Playbook` role. We may consider
splitting it into a dedicated `Playbook` document (potentially with some monitoring
details). For now, we leave it here.
-->

###### How does this feature react if the API server and/or etcd is unavailable?

Same as without the feature.

###### What are other known failure modes?

N/A

###### What steps should be taken if SLOs are not being met to determine the problem?

1. The feature can be disabled by setting the feature-gate to false if the performance impact of it is not tolerable.
2. Try to run the rules separately to see which rule is slow
3. Remove the problematic rules or update the rules to meet the requirements

## Graduation Criteria

### Beta

Code that allows x-kubernetes-validations to be included (but ignored) in CRDs is backported all
supported Kubernetes versions. Also, we should make an assessment of how much support we have in
older kubernetes versions for this feature before we promote to Beta, delaying promotion as needed
to minimize negative impact to ecosystem.

## Alternatives

### Introduce CEL for General Admission Control

See also the future plans section for this. We believe that CEL for General Admission Control is
valuable and should be implemented. We are implementing CRD validation with CEL first because is is
a more constrained problem and is complementary to CEL for general admission (even if we already
had CEL for general admission implemented, the convenience of inline CEL validation expressions in
CRDs is sufficiently convenient to justify it being added).

### Rego

See Open Policy Agent (https://github.com/open-policy-agent/opa/tree/main/rego).
The syntax is more extensive than CEL and is designed specifically to work well
with kubernetes objects. It allows larger, multi-line programs and 
includes a package and module system. It does not offer the same sandbox constraints
as CEL, nor does it type check code.

### Expr

See github.com/antonmedv/expr. Has many similarities to CEL: type checking, minimalist syntax,
good performance and sandboxing properties.

This is used by the argo.

### WebAssembly

We looked closely at WebAssembly and created a [proof-of-concept implementation](https://github.com/jpbetz/omni-webhook/blob/main/validators/wasm.go).
The biggest problems with WebAssembly are:
- It doesn't work well as an embedded expression language. With WebAssembly, we would really want to
  have the binaries published somewhere (docker images?) and then referenced in CRD
  declarations so the apiserver could then load and execute them. This would be
  far less convenient for writing simple validation rules than just inlining expressions.
- WebAssembly runtimes require `cgo` to build, something that
  might be difficult to integrate into api-server.
- Passing strings across a WebAssembly boundary is currently dependent on the target language, so any supported
  target language would need a small shim library to be supported. This complicates the developer
  workflow.

See also github.com/chimera-kube/chimera-admission

### Starlark (formerly known as Skylark)

See https://github.com/google/starlark-go/.
Starlark is an untyped dynamic language with high-level data types, first-class functions with lexical scope, and automatic memory management or garbage collection.
It is mostly used in build system and has been added as a dependency in k/k.

Python dialect designed for scripts embedded in the Bazel build system. It is designed to allow for
determinstic and hermetic execution. Implementations exist in Go, Java and Rust. It is used
primarily in build and documentation generators. The language definition is much larger than the
other embeddable expression languages considered.

Cons:
- Does not provide type checking
- Indention aware grammar is not a good fix to single line expressions
- Execution of untrusted code in a sandbox is not a top level project goal

### Build our own

Given that this would require a much larger engineering investment, we do not plan on entertaining
unless there is strong evidence that none available expression languages are able to support CRD validation
use cases well.

### Make it easier to validate CRDs using webhooks

This has been explored by the community. There are examples in the ecosystem of Rego, Expr and WebAssembly
in the ecosystem.

Kubebuilder can automatically create and manage a webhook to run validation and defaulting code (
https://book.kubebuilder.io/cronjob-tutorial/webhook-implementation.html).

But for a CRD developer that just needs to add a simple validation, having direct access
to an expression language is far simpler than exploring this ecosystem to find the easiest
way to do validation and then investing in it (which may require buy-in to a larger framework)
is a time consuming way to solve what should be a simple problem.

For cluster operators, regardless of what extensions they install in their cluster,
it is to their advantage to install the fewest webhooks possible since.

## Infrastructure Needed (Optional)

<!--
Use this section if you need things from the project/SIG. Examples include a
new subproject, repos requested, or GitHub details. Listing these here allows a
SIG to get the process for these resources started right away.
-->
