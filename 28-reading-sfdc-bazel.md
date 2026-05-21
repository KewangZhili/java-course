# Module 28 — Reading SFDC-Bazel-Style Codebases

> Goal: open a large, unfamiliar Java monorepo and become productive without panic. This module is about *navigation* and *recognition* — how Bazel-built code is organized, what the annotations mean, what's likely generated, and how to find what you need fast.

The patterns here apply broadly to any Bazel-Java monorepo. Salesforce Core is the example, but the same skills work for Google's java code, Twitter's older Pants/Bazel layout, Pinterest, etc.

---

## 1. Repo shape: why it's huge

SFDC-Bazel codebases tend to look like:

```
core/                                   ← top of the monorepo
├── WORKSPACE / WORKSPACE.bazel         ← Bazel workspace declaration
├── BUILD.bazel
├── tools/                              ← build tooling
├── third_party/                        ← external dependencies as Bazel targets
├── proto/                              ← .proto schemas; codegen produces Java
├── api/                                ← public-facing API contracts
├── services/
│   ├── account-service/
│   │   ├── BUILD.bazel
│   │   ├── src/main/java/com/sfdc/...
│   │   └── src/test/java/com/sfdc/...
│   ├── billing-service/
│   └── ...
├── shared/
│   ├── util/
│   ├── persistence/
│   └── ...
└── platform/
```

A few thousand top-level packages, millions of lines, hundreds of services. The tree is *built as one unit* — there's no per-service `pom.xml`/`build.gradle`. Every directory with code has a `BUILD.bazel`.

---

## 2. Reading a `BUILD.bazel` quickly

```python
java_library(
    name = "account_service_lib",
    srcs = glob(["src/main/java/**/*.java"]),
    deps = [
        "//shared/util:util_lib",
        "//shared/persistence:persistence_lib",
        "//proto/account:account_proto_java",
        "@maven//:com_google_guava_guava",
        "@maven//:com_fasterxml_jackson_core_jackson_databind",
    ],
    visibility = ["//services:__subpackages__"],
    resources = glob(["src/main/resources/**"]),
    plugins = ["//tools/lombok:lombok_plugin"],
)

java_test(
    name = "account_service_test",
    srcs = glob(["src/test/java/**/*.java"]),
    deps = [
        ":account_service_lib",
        "@maven//:org_junit_jupiter_junit_jupiter",
        "@maven//:org_mockito_mockito_core",
        "@maven//:org_assertj_assertj_core",
    ],
    runtime_deps = ["@maven//:org_junit_jupiter_junit_jupiter_engine"],
    resources = glob(["src/test/resources/**"]),
)

java_binary(
    name = "account_service_main",
    main_class = "com.sfdc.account.Main",
    runtime_deps = [":account_service_lib"],
)
```

Reading order:
1. **`name`** — the target's local name. Reference it as `//services/account-service:account_service_lib`.
2. **`srcs`** — source files. `glob` pulls everything matching the pattern.
3. **`deps`** — explicit list of what this target needs to compile.
   - `//path:target` — another target in the repo.
   - `:target` — same package.
   - `@maven//:org_apache_commons_commons_lang3` — external Maven dep, with periods/hyphens replaced by underscores.
4. **`visibility`** — who can depend on this. `//visibility:public` is permissive; specific paths are restrictive.
5. **`runtime_deps`** — needed at runtime but not compile time (e.g., JDBC driver).
6. **`resources`** — non-Java files included on the classpath.
7. **`plugins`** — annotation processors.

When you find the BUILD file for the package you're working in, the dep list **tells you everything that file imports** — much faster than reading every `import` statement.

---

## 3. The annotations you'll see (a tour)

Annotations are everywhere in enterprise Java — they're how frameworks communicate. Some you'll know from earlier modules; others are SFDC-specific. Here's the field guide:

### 3.1 Standard Java
- `@Override` — promised override.
- `@Deprecated` — marked for removal.
- `@SuppressWarnings("...")` — disable a specific compiler warning.
- `@FunctionalInterface` — declares a SAM interface.

### 3.2 Spring (Module 25/26)
- `@Component`, `@Service`, `@Repository`, `@Controller`, `@RestController` — bean stereotypes.
- `@Autowired`, `@Qualifier`, `@Primary` — injection.
- `@Configuration`, `@Bean`, `@Profile` — configuration.
- `@Value("${prop}")` — inject property.
- `@RequestMapping`, `@GetMapping`, `@PostMapping`, `@PathVariable`, `@RequestParam`, `@RequestBody` — REST.
- `@Transactional` — transactions.

### 3.3 Persistence
- `@Entity`, `@Table`, `@Id`, `@GeneratedValue`, `@Column` — JPA mapping.
- `@OneToMany`, `@ManyToOne`, `@JoinColumn` — relations.
- `@Query` — custom JPQL.

### 3.4 Validation
- `@Valid`, `@Validated` — trigger validation.
- `@NotNull`, `@NotBlank`, `@Email`, `@Min`, `@Max`, `@Size`, `@Pattern` — constraints.

### 3.5 Lombok (very common in SFDC code)
- `@Getter`, `@Setter` — generates accessors.
- `@AllArgsConstructor`, `@NoArgsConstructor`, `@RequiredArgsConstructor` — generates constructors.
- `@Data` — equivalent to `@Getter @Setter @ToString @EqualsAndHashCode @RequiredArgsConstructor`. Used to be ubiquitous; modern code prefers records.
- `@Builder` — generates a fluent builder.
- `@Slf4j` — generates `private static final Logger log = ...`.

Lombok is *bytecode rewriting at compile time* via an annotation processor. The Java source has annotations; the `.class` file has actual methods. Your IDE needs the Lombok plugin to render the synthesized methods or you'll see "method not found" all over.

### 3.6 Jackson (JSON)
- `@JsonProperty("name")` — explicit JSON name.
- `@JsonIgnore` — skip in serialization.
- `@JsonCreator` — non-default constructor for deserialization.
- `@JsonInclude(Include.NON_NULL)` — omit null fields.

### 3.7 Codegen markers
- `@Generated("...")` — usually ignored by tooling; signals "do not edit; regenerate".
- Any class in a `gen/` or `generated/` directory.

### 3.8 SFDC-specific (illustrative)
Internal codebases ship many custom annotations. Common patterns:
- `@MetricCounter("name")` — wired by an aspect to update an internal metrics registry.
- `@AuditLog("event")` — captures method calls into an audit trail.
- `@RequiresPermission("PERM")` — security check via AOP.
- `@Feature("FLAG")` — gate by feature flag.
- `@RetryOnFailure(attempts = 3)` — retry aspect.

When you see an unfamiliar annotation:
1. Click through to its declaration.
2. If it's a *meta-annotation* (annotated `@Target`, `@Retention`), it's likely processed at runtime by an aspect or framework.
3. Search the codebase for the annotation type name in non-source-file contexts (filter by file extension to find bytecode-time / runtime processors).

---

## 4. Generated code

Bazel monorepos lean on codegen. You'll routinely encounter Java sources you can't find in `src/`:

- **Protocol Buffers** (`.proto` → Java classes via `proto_library` + `java_proto_library`).
- **Avro/Thrift** schemas similarly.
- **Lombok** — annotation processor adds methods.
- **MapStruct** — generates `XxxMapperImpl` classes.
- **JPA static metamodel** — generates `User_` companion classes.
- **Internal SFDC codegen** — DSLs that produce data-access classes, RPC stubs, etc.

When IntelliJ jumps to "no source attached" or to a `.class` in `bazel-bin/` or `bazel-out/`, you're looking at generated output. Look for the input — usually a sibling `.proto` or `.idl` file, or an annotation processor in `BUILD.bazel`'s `plugins` list.

In Bazel, generated files live under `bazel-bin/` and `bazel-out/`. They're recreated by `bazel build`. Don't edit them.

---

## 5. Navigation tools that actually work

### 5.1 IntelliJ IDEA + Bazel plugin

The official **Bazel plugin** for IntelliJ syncs a Bazel workspace into a project model IntelliJ understands. You point at the WORKSPACE and a list of targets/directories to import; IntelliJ runs a query, downloads dependencies, and builds an index. Once synced:
- Ctrl-click on a class jumps to source.
- Ctrl-N opens a class by name.
- Ctrl-Alt-Shift-N opens any symbol.
- Ctrl-F12 lists members in the current file.
- Ctrl-Shift-F searches across the imported scope.

Tip: import only the subtree you need. Importing the entire monorepo can be impossibly slow.

### 5.2 Command-line search

`grep -r "FooService" --include='*.java' .` works but slow on huge repos. Better:

- **`rg`** (ripgrep) — fast text search, respects `.gitignore`.
- **`fd`** — fast file finder.
- **`codesearch`** (Salesforce internal tool) — indexed code search across the monorepo. Far faster than ripgrep on millions of files.

### 5.3 Bazel queries

```
bazel query "//services/account-service:account_service_lib"      # exists?
bazel query "deps(//services/account-service:account_service_lib)"  # what does it depend on?
bazel query "rdeps(//..., //shared/util:util_lib)"                  # who depends on this?
bazel query "kind('java_test', //services/...)"                     # all tests in services/
bazel query "somepath(//a:x, //b:y)"                                # show one path between two targets
```

`rdeps` is invaluable when changing a shared API: you'll see every caller.

### 5.4 javap / javapdb

If you only have a `.class` and no source, `javap -p -c FooImpl` prints all members and bytecode. Useful for generated classes you can't otherwise inspect.

---

## 6. Reading a service from cold

A pattern that works:

1. **Find the entry point.** Look for `java_binary` in the service's BUILD file, then its `main_class`. That's `Main.java` (or similar).
2. **Trace the bootstrap.** A Spring app calls `SpringApplication.run(Application.class, args)` — from there, `@ComponentScan` finds the rest. Internal frameworks may use a different bootstrap; follow the calls from `main`.
3. **Skim the controllers.** They are the public surface. `@RestController`/`@Controller` classes show what the service exposes.
4. **Pick one endpoint.** Trace it: controller → service → repository → database. You now understand one full vertical slice.
5. **Look for `@Configuration`** classes near the root. They tell you the wiring — what beans exist, what profiles flip behavior.
6. **Find the tests.** Tests are documentation. A `XxxControllerTest` shows expected request/response shapes. A `XxxServiceTest` shows what behaviors matter to the team.

After one full slice, you can navigate the rest by analogy.

---

## 7. Patterns specific to large monorepos

### 7.1 Many small libraries, explicit deps

A typical SFDC service `BUILD.bazel` lists 30–80 deps. Pre-Bazel monorepos used a few fat JARs; Bazel encourages tiny libraries. The benefit: change in `shared/util` only rebuilds & retests the targets that actually depend on it. The cost: dep lists are long.

### 7.2 Internal API/Impl split

```
api/          ← public interfaces, DTOs
impl/         ← implementations, hidden behind api visibility
```

Visibility is enforced. Only some packages can depend on `impl/`. Always check what `visibility = [...]` says before adding a dep.

### 7.3 Versioned services

You may see two implementations of the same logic (`v1`, `v2`) live side-by-side during a migration. Routing is gated by a feature flag. Read the flag's definition to understand which path is currently live.

### 7.4 Generated DTOs from canonical schemas

Avoid hand-rolling DTOs that mirror schemas. Bazel monorepos run codegen so a single `.proto` (or schema file) is the source of truth. Java DTO is regenerated. Don't edit it by hand.

### 7.5 Lots of cross-cutting via aspects

Logging, metrics, audit, security, retry — all done with annotations + aspects. `grep -r "@MetricCounter"` shows the pattern; the aspect that processes it lives in a shared module.

### 7.6 Heavy use of `Optional`, `@Nullable`

Many SFDC codebases mark every nullable field/parameter with `@Nullable` (often `org.checkerframework.checker.nullness.qual.Nullable` or `javax.annotation.Nullable`) and use `Optional<T>` for return types where absent is meaningful. Treat the absence of `@Nullable` as a contract: this is non-null.

### 7.7 Lombok or records

Newer code: records. Older code: `@Data` / `@Value` / `@Builder` from Lombok. They're equivalent in spirit; reading both becomes second nature.

---

## 8. Editing safely in a monorepo

When you change something:
- **Run `bazel test`** on the *target* you changed. Fast.
- **Find downstream users**: `bazel query "rdeps(//..., //changed/target:...)"`. If you changed a public API, build/test those reverse deps.
- **Don't bypass visibility.** If you can't add a dep because of visibility, talk to the package owner — they made the boundary deliberate.
- **Don't blindly edit generated code**. Find the input.
- **Mind annotations.** Adding/removing `@Transactional`, `@Async`, `@Cacheable` changes runtime behavior even when source-level tests pass.
- **Keep PRs small.** A million-line monorepo's review queue moves faster on focused diffs.

---

## 9. Common reading mysteries solved

> "I can't find where this method is implemented."

It's probably:
- An interface method dispatched via DI — find implementations: IntelliJ's "Find usages" + "Implementations" (Ctrl-Alt-B on the method).
- A Spring proxy — the call goes through a generated proxy. Look at the interface; the implementation is the bean.
- Generated by Lombok — trust the annotation; check `bazel-bin` for the actual class.

> "This file imports a class I can't open."

Likely codegen. Check `BUILD.bazel` for `proto_library`, `java_proto_library`, or annotation processors.

> "The class compiles but at runtime I get `NoSuchMethodError`."

Version skew. Two transitive deps brought different versions of the same library. `bazel query` to find both paths; pin a version.

> "This `@Service` doesn't get injected."

It's not in a scanned package, or it's `@Profile`-gated, or there's an ambiguous match Spring is silently picking. Add a startup log to dump the bean registry, or use `@Qualifier` to disambiguate.

> "Tests pass locally, fail on CI."

Usually one of: time-zone differences, classpath ordering, stable test seeds, or a flaky external dependency. Reproduce locally first by clearing caches and running fresh.

---

## 10. Try this

1. Pick a service in your monorepo. Open its `BUILD.bazel`. Identify the `java_library`, `java_test`, and `java_binary` targets. Run `bazel test` on the test target.
2. From the controller of that service, trace one endpoint to the database. Make a sketch of every class you crossed.
3. Run `bazel query "rdeps(//..., <some shared lib>)"`. Pick the smallest output and read its BUILD to see how it consumes the lib.
4. Find a `@Generated`-annotated class. Locate its source schema. Verify that regenerating it produces the same class.

---

**Next:** [Module 29 — Competitive programming in Java](./29-competitive-programming.md)
