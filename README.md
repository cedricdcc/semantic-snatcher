# semantic-snatcher

a deno project that will extract triples and a object URI from a given source URI

### Step-by-Step Plan for the semantic snatching

#### 1. **Initialize Function**

- Input: A URI that will be used to retrieve triples.
- Output: A triplestore and the subject URI that describes the object (if found).

#### 2. **Perform Preflight `HEAD` Request**

- **Send a `HEAD` request** to the URI to check available `Accept` headers.
- **If available headers are returned**, note them for subsequent GET requests.
- **If no headers are returned**, use default Accept headers:
  - `text/html`, `text/turtle`, `application/ld+json`.

#### 3. **Retrieve Initial Triples (with GET requests)**

- **3.1 Send HTTP GET Request with determined Accept headers:**
  - For each header type, send a `GET` request to try and retrieve triples.
  - **If a response with `text/turtle` or `application/ld+json` is successful**:
    - Parse the content and add it to the triplestore.
    - **Exit this step** if triples are found.
  - **If `text/html` is used**:
    - Parse for embedded `application/ld+json` or `text/turtle` in the HTML.
    - Look for JSON-LD content in `<script>` tags or `<link>` tags.
    - If found, fetch content, parse JSON-LD, and add it to the triplestore.
  - **If no triples are found** in all Accept headers, exit the function without a triplestore.

#### 4. **Check if Triples Are Found**

- If the triplestore is empty, terminate the function.
- If triples are found, proceed to find the `subject URI` within the triplestore.

#### 5. **Query Triplestore for Subject URI**

- Use the original URI as the subject in a query.
- If triples are returned, check for `schema:describes`.
- If no results, query the triplestore using the final redirected URI if applicable.

#### 6. **Return Results**

- Return the populated triplestore and the determined `subject URI`.

```mermaid
flowchart TD
    A[Start: Initialize with URI] --> B[Send HEAD Request to Check Accept Headers]
    B --> C{Headers Returned?}
    C -->|Yes| D[Use Returned Headers for GET Requests]
    C -->|No| E[Use Default Headers &lpar;text/turtle, application/ld+json, text/html&rpar;]

    E --> F[Send GET Request with Accept Headers]
    D --> F

    F --> G{Response Content}
    G -->|text/turtle or application/ld+json| H[Parse and Store Triples]
    G -->|text/html| I[Parse HTML for Embedded Triples]
    I --> J{Embedded JSON-LD or Turtle Found?}
    J -->|Yes| H
    J -->|No| K[Check Next Accept Header]
    K --> F

    H --> L{Triplestore Populated?}
    L -->|No| M[Return No Triples Found]
    L -->|Yes| N[Query Triplestore for Subject URI]

    N --> O{Triples Found for Original URI?}
    O -->|Yes| P[Check for schema:describes Predicate]
    P -->|Yes| Q[Return Triplestore and Object as Subject URI]
    P -->|No| R[Use Original URI as Describing Subject URI]
    O -->|No| S{Was URI Redirected?}

    S -->|Yes| T[Query with Final Redirected URI]
    T --> U{Triples Found?}
    U -->|Yes| V[Check for schema:describes Predicate]
    V -->|Yes| Q
    V -->|No| W[Query Triplestore with Original URI as Object]

    U -->|No| W
    S -->|No| W
    W --> X{Triples Found for URI as Object?}
    X -->|Yes| Y[Use Subject of Triple as Subject URI]
    Y --> Q
    X -->|No| M
```

---

### Sequence Diagram Description

1. **User/System** invokes the function with an initial URI.
2. **Function** sends HTTP GET requests with specified Accept headers (`text/turtle`, `application/ld+json`, and `text/html`).
   - For each response, it tries to parse and store triples in the triplestore.
   - If `text/html` is used, **HTML Parser** looks for embedded JSON-LD or Turtle content.
3. **Function checks the triplestore** to see if triples were found:
   - If empty, **returns** without data.
   - If populated, proceeds to subject URI search.
4. **Function performs query** with the original URI as the subject:
   - If successful, checks for `schema:describes`.
   - If found, uses the triple’s object as the `subject URI` and **returns the triplestore and subject URI**.
5. If no relevant triples are found and if there were redirects:
   - **Function performs query with the final redirected URI** as the subject.
6. If still no results, **Function performs query using the URI as an object**.
7. If triples are returned, uses the subject of the triple as the `subject URI`.
8. **Return Result**:
   - Function returns the `triplestore` and the determined `subject URI`.

```mermaid
sequenceDiagram
    actor User
    participant SemanticSnatcher
    participant HTTPService as HTTP Service
    participant HTMLParser as HTML Parser
    participant Triplestore

    User->>SemanticSnatcher: call getTriples(uri)

    SemanticSnatcher->>HTTPService: _getAvailableHeaders(uri) - HEAD request
    alt Headers Returned
        HTTPService-->>SemanticSnatcher: Available Accept headers
    else No Headers Returned
        Note right of SemanticSnatcher: Use default headers (text/turtle, application/ld+json, text/html)
    end

    loop for each Accept Header
        SemanticSnatcher->>HTTPService: _fetchWithHeaders(uri, header) - GET request
        alt Turtle or JSON-LD response
            HTTPService-->>SemanticSnatcher: Returns RDF data
            SemanticSnatcher->>Triplestore: _parseAndStoreTriples(data, format)
        else HTML Content
            HTTPService-->>SemanticSnatcher: Returns HTML content
            SemanticSnatcher->>HTMLParser: _extractTriplesFromHTML(html)
            alt JSON-LD found in HTML
                HTMLParser-->>SemanticSnatcher: JSON-LD data
                SemanticSnatcher->>Triplestore: Store extracted triples
            else No embedded JSON-LD
                HTMLParser-->>SemanticSnatcher: No embedded JSON-LD
            end
        else No data returned
            HTTPService-->>SemanticSnatcher: No content found
        end
    end

    alt Triplestore empty
        SemanticSnatcher-->>User: Return empty result (No triples found)
    else Triples found
        SemanticSnatcher->>Triplestore: _findSubjectUri(uri) - query triplestore
        alt Describes Predicate Found
            Triplestore-->>SemanticSnatcher: Object URI as subject
            SemanticSnatcher-->>User: Return triplestore and subject URI
        else No matching subject
            SemanticSnatcher->>Triplestore: Check if URI is used as an object
            alt URI found as object
                Triplestore-->>SemanticSnatcher: Subject URI
                SemanticSnatcher-->>User: Return triplestore and subject URI
            else No URI matches found
                SemanticSnatcher-->>User: Return triplestore (no describing subject)
            end
        end
    end
```

---

# Class Diagram Description for the Semantic Snatcher

```mermaid
classDiagram
    class SemanticSnatcher {
        - triplestore: rdf.Store
        - n3Parser: N3Parser
        + constructor()
        + getTriples(uri: string): Promise<TripleResult>
        - _getAvailableHeaders(uri: string): Promise<string[]>
        - _fetchWithHeaders(uri: string, acceptHeader: string): Promise<string | null>
        - _parseAndStoreTriples(data: string, format: string): Promise<void>
        - _extractTriplesFromHTML(html: string): Promise<void>
        - _findSubjectUri(uri: string): Promise<string | null>
    }

    SemanticSnatcher --> HTTPService : uses
    SemanticSnatcher --> HTMLParser : uses
    SemanticSnatcher --> Triplestore : manages

    class HTTPService {
        + fetch(uri: string, options: RequestInit): Promise<Response>
    }

    class HTMLParser {
        + parseFromString(html: string, mimeType: string): Document
        + querySelectorAll(selector: string): NodeList
    }

    class Triplestore {
        + add(quad: Quad): void
        + match(subject: Node, predicate: Node, object: Node): Array<Quad>
        + length: number
    }

    class N3Parser {
        + parse(data: string, callback: (error: Error | null, quad: Quad | null) => void): void
    }

    SemanticSnatcher *-- Triplestore : stores
    SemanticSnatcher *-- N3Parser : parses

    class jsonld {
        + toRDF(jsonldData: object, options: FormatOptions): Promise<string>
    }

    SemanticSnatcher --> jsonld : uses

    class TripleResult {
        + triplestore: rdf.Store
        + subjectUri: string | null
    }

    class FormatOptions {
        + format: string
    }
```
