# PIR Schema

Each PIR file under `/pirs/` must follow this markdown structure. The orchestrator parses sections by heading using a deterministic regex, so heading names and ordering matter.

## Required sections (in order)

### `# <Title>` (H1)

The first H1 is treated as the PIR title.

### `## Summary`

One paragraph describing the incident.

### `## Anti-Pattern`

Description of the anti-pattern that caused or contributed to the incident, followed by a fenced code block showing the bad pattern.

~~~
```java
// example of the anti-pattern
```
~~~

### `## Fix`

Description of the corrective pattern, followed by a fenced code block showing the corrected approach.

~~~
```java
// example of the fix
```
~~~

### `## Scope Hints`

JSON block used by the orchestrator's Stage 2 filter to narrow the candidate service list to relevant repos before any scanning agents run.

~~~
```json
{
  "languages": ["java"],
  "frameworks": ["hikaricp"],
  "service_types": ["api", "worker"]
}
~~~

A service from `service-catalog.json` is considered a candidate if it matches at least one entry in each non-empty hint array. Empty arrays mean "don't filter on this dimension."
