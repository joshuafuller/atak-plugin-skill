# Works on the JVM, fails on device

The failure mode to fear is not a crash — it is code that passes every unit
test and silently does nothing right on Android. Everything here was found by
an on-device test after the JVM suite was fully green.

## XML: `disallow-doctype-decl` is not supported

Android's `DocumentBuilderFactory` throws for
`http://apache.org/xml/features/disallow-doctype-decl` — the usual XXE
recommendation. This leaves two bad options and one good one:

- Set it inside a broad `try`/ignore → the control is **silently absent**.
- Treat the failure as fatal → **every document fails to parse on device**,
  while the JVM suite stays green.
- Enforce the rule on the raw bytes, which does not depend on the parser.

Scan the prolog and refuse any `<!DOCTYPE`, stopping at the first tag that is
not `<?…` or `<!…` (that is the root; a DOCTYPE cannot follow it). Scanning the
whole document instead would reject a legitimate file whose URL happens to
contain the word.

Keep the feature calls as best-effort defence in depth — each in its own try, so
one unsupported feature does not skip the rest:

```java
harden(() -> factory.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true));
harden(() -> factory.setFeature(XMLConstants.FEATURE_SECURE_PROCESSING, true));
harden(() -> factory.setFeature("http://xml.org/sax/features/external-general-entities", false));
harden(() -> factory.setFeature("http://xml.org/sax/features/external-parameter-entities", false));
harden(() -> factory.setXIncludeAware(false));
harden(() -> factory.setExpandEntityReferences(false));
```

`setXIncludeAware(false)` is documented to throw
`UnsupportedOperationException` on some implementations, which is what makes a
single shared try block actively dangerous.

An `EntityResolver` that throws is worth adding too — it costs nothing and
prevents any fetch a document requests.

## API level

The 5.8 template sets `minSdkVersion 21`. `List.of()`, `Map.of()`,
`Optional` chains and other Java 9+ APIs need API 30+ or core library
desugaring. `Collections.emptyList()` and friends always work.
`lintVitalCivRelease` catches most of this on `assembleCivRelease` — but your
unit tests never will, so do not rely on tests to notice.

## Streams

`InputStream.read()` returning `0` is legal. A drain loop written
`while ((n = in.read(buf)) > 0)` exits early on such a stream and silently
returns a truncated result; use `!= -1`.

## The general rule

If a class comes from the platform rather than from your own code, its
behaviour on Android is a fact to be measured, not inferred from the JVM. Put
one on-device test on any code path whose correctness depends on a platform
class — that test is the only thing that can see this class of bug.
